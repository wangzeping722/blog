---
title: "gRPC Balancer 分析"
date: 2020-11-06T18:07:55+08:00
draft: false
---

## 1. 负载均衡

gRPC 实现负载均衡的方法主要有三种，分别是

- 集中式 LB（Proxy Model）
- 进程内 LB（Balancing-aware Client）
- 独立 LB 进程（External Load Balancing Service）

具体的实现方式请参考这篇文章[gRPC 服务发现&负载均衡](https://segmentfault.com/a/1190000008672912)。

需要注意的是，gRPC 使用 **进程内 LB** 的方案，并且是基于每次调用 RPC 接口实现的负载均衡， 如下图：

<img src="https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/load-balancing.png" style="zoom:67%;" />

1. 服务启动后 gRPC 客户端向注册中心获取后端服务列表
2. 客户端实例化负载均衡策略
3. 负载均衡器（Balancer）为每一个后端地址创建一个子连接（SubConn）
4. 当要发起 RPC 请求的时候，负载均衡器就会选择当前最匹配的子连接来进行 RCP 调用

## 2. 问题

今后有关代码剖析的文章都会提出问题，带着问题来看源码，收获会更多：

- Balancer 如何接收来自 Resolver 解析出来的地址？
- 与这些地址的连接是在哪建立的？
- 如何选择每次调用适合的连接？

[L9-go-resolver-balancer-API](https://github.com/grpc/proposal/blob/master/L9-go-resolver-balancer-API.md)这篇文章中给出了 resolver-balancer 的设计理念，并且给出了官方架构图：

![resolver-balancer](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/resolver-balancer-arch.png)

图中，我们可以看见 Balancer 位于 gRPC 的右方，并且 Resolver 与 Balancer 没有耦合在一起，而是采用接口隔离加 Builder 和Wrapper 设计模式。其中，Balancer 负责与后端服务器建立连接，并且创建 Picker 来执行负载均衡的逻辑。



## 3. 一切都要从 Resolver 说起

上文，我们提到 Resolver 在解析到后端服务器列表之后会通过 `resolver.ClientConn.UpdateState(state)`来通知 gRPC 后端地址有更新，对应上图的左边 `Addr Updates`。代码如下：

``` go
// ccResolverWrapper 可以看做是 Resolver 的代理，并且实现了resolver.ClientConn接口
type ccResolverWrapper struct {
	cc         *ClientConn
	resolverMu sync.Mutex
	resolver   resolver.Resolver
	done       *grpcsync.Event
	curState   resolver.State

	pollingMu sync.Mutex
	polling   chan struct{}
}

func (ccr *ccResolverWrapper) UpdateState(s resolver.State) {
	if ccr.done.HasFired() {
		return
	}
	...
	// 更新地址池
	ccr.curState = s
	ccr.poll(ccr.cc.updateResolverState(ccr.curState, nil))
}
```

可见最终会调用 `ClientConn.updateResolverState(ccr.curState, nil)` 方法，在这个方法里面，gRPC 会处理 Balancer 的逻辑。

这里需要梳理一下：

- ccResolverWrapper 实现了 `resolver.ClientConn` 接口
- ccResolverWrapper 中的 cc 是 `grpc.ClientConn` 结构体
- 当在 Resolver 中调用  `resolver.ClientConn.UpdateState(state)` 的时候，便最终会调用 `ClientConn.updateResolverState(ccr.curState, nil)`

以 dnsResolver 为例，调用链如下：

``` sh
dnsResovler.watcher --> ccResolverWrapper.UpdateState --> ClientConn.updateResolverState
-->  cc.balancerWrapper.updateClientConnState
```

### ClientConn.updateResolverState

updateResolverState 会与 Balancer 模块进行交互：

``` go
func (cc *ClientConn) updateResolverState(s resolver.State, err error) error {
	defer cc.firstResolveEvent.Fire()
	cc.mu.Lock()
	// Check if the ClientConn is already closed. Some fields (e.g.
	// balancerWrapper) are set to nil when closing the ClientConn, and could
	// cause nil pointer panic if we don't have this check.
	if cc.conns == nil {
		cc.mu.Unlock()
		return nil
	}

	if err != nil {
		// May need to apply the initial service config in case the resolver
		// doesn't support service configs, or doesn't provide a service config
		// with the new addresses.
		cc.maybeApplyDefaultServiceConfig(nil)

		if cc.balancerWrapper != nil {
			cc.balancerWrapper.resolverError(err)
		}

		// No addresses are valid with err set; return early.
		cc.mu.Unlock()
		return balancer.ErrBadResolverState
	}

	var ret error
	// 如果禁用服务配置或者服务配置为空, 则可能使用默认配置
	if cc.dopts.disableServiceConfig || s.ServiceConfig == nil {
    // A
		cc.maybeApplyDefaultServiceConfig(s.Addresses)
		// TODO: do we need to apply a failing LB policy if there is no
		// default, per the error handling design?
	} else {
    // B
		if sc, ok := s.ServiceConfig.Config.(*ServiceConfig); s.ServiceConfig.Err == nil && ok {
			cc.applyServiceConfigAndBalancer(sc, s.Addresses)
		} else {
			// 配置不正确
			ret = balancer.ErrBadResolverState
			if cc.balancerWrapper == nil {
				var err error
				if s.ServiceConfig.Err != nil {
					err = status.Errorf(codes.Unavailable, "error parsing service config: %v", s.ServiceConfig.Err)
				} else {
					err = status.Errorf(codes.Unavailable, "illegal service config type: %T", s.ServiceConfig.Config)
				}
				cc.blockingpicker.updatePicker(base.NewErrPicker(err))
				cc.csMgr.updateState(connectivity.TransientFailure)
				cc.mu.Unlock()
				return ret
			}
		}
	}

	var balCfg serviceconfig.LoadBalancingConfig
	// 优先使用客户端配置的负载均衡策略
	if cc.dopts.balancerBuilder == nil && cc.sc != nil && cc.sc.lbConfig != nil {
		balCfg = cc.sc.lbConfig.cfg
	}

	cbn := cc.curBalancerName
	bw := cc.balancerWrapper
	cc.mu.Unlock()
	// 如果负载的负载均衡策略不是 grpclb, 就剔除所有类型是 grpclb 的地址
	if cbn != grpclbName {
		// Filter any grpclb addresses since we don't have the grpclb balancer.
		for i := 0; i < len(s.Addresses); {
			if s.Addresses[i].Type == resolver.GRPCLB {
				copy(s.Addresses[i:], s.Addresses[i+1:])
				s.Addresses = s.Addresses[:len(s.Addresses)-1]
				continue
			}
			i++
		}
	}
 	// 调用负载均衡策略更新连接状态
	uccsErr := bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
	if ret == nil {
		ret = uccsErr // prefer ErrBadResolver state since any other error is
		// currently meaningless to the caller.
	}
	return ret
}
```

一般情况下，我们都会进入 A 代码块（使用默认配置）：

``` go
func (cc *ClientConn) maybeApplyDefaultServiceConfig(addrs []resolver.Address) {
	if cc.sc != nil {
		cc.applyServiceConfigAndBalancer(cc.sc, addrs)
		return
	}
	if cc.dopts.defaultServiceConfig != nil {
		cc.applyServiceConfigAndBalancer(cc.dopts.defaultServiceConfig, addrs)
	} else {
    // 
		cc.applyServiceConfigAndBalancer(emptyServiceConfig, addrs)
	}
}
```

当我们未指定配置的时候，会传入 `emptyServiceConfig` 空的配置,  然后调用 `ClientConn.applyServiceConfigAndBalancer`：

``` go
func (cc *ClientConn) applyServiceConfigAndBalancer(sc *ServiceConfig, addrs []resolver.Address) {
	if sc == nil {
		// should never reach here.
		return
	}
	// 可能是 emptyServiceConfig
	cc.sc = sc

	if cc.sc.retryThrottling != nil {
		newThrottler := &retryThrottler{
			tokens: cc.sc.retryThrottling.MaxTokens,
			max:    cc.sc.retryThrottling.MaxTokens,
			thresh: cc.sc.retryThrottling.MaxTokens / 2,
			ratio:  cc.sc.retryThrottling.TokenRatio,
		}
		cc.retryThrottler.Store(newThrottler)
	} else {
		cc.retryThrottler.Store((*retryThrottler)(nil))
	}

  // 如果没有指定 balancerBuilder, 就需要先获取一个 builder, 一般情况下都会指定
	if cc.dopts.balancerBuilder == nil {
		// Only look at balancer types and switch balancer if balancer dial
		// option is not set.
		var newBalancerName string
		if cc.sc != nil && cc.sc.lbConfig != nil {
      // 1. 使用 ServiceConfig 指定的 Builder
			newBalancerName = cc.sc.lbConfig.name
		} else {
			var isGRPCLB bool
			for _, a := range addrs {
				if a.Type == resolver.GRPCLB {
					isGRPCLB = true
					break
				}
			}
      // 2. 如果至少有一个地址的类型是 GRPCLB, 那么使用 GRPCLB
			if isGRPCLB {
				newBalancerName = grpclbName
			} else if cc.sc != nil && cc.sc.LB != nil {
        // 3. 使用 WithBalancer 提供的 builder
				newBalancerName = *cc.sc.LB
			} else {
        // 4. 使用 pick_first
				newBalancerName = PickFirstBalancerName
			}
		}
    // 切换负载均衡策略
		cc.switchBalancer(newBalancerName)
	} else if cc.balancerWrapper == nil {
		// Balancer dial option was set, and this is the first time handling
		// resolved addresses. Build a balancer with dopts.balancerBuilder.
		cc.curBalancerName = cc.dopts.balancerBuilder.Name()
		cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
	}
}
```

从上面代码可以看出 ClientConn 会通过服务配置选择对应的 Balancer：

1. 优先使用客户端配置的负载均衡策略

2. 如果客户端没有指定负载均衡策略，那么会通过服务配置来决定使用什么负载均衡策略，选择规则如下：

   1. 使用 ServiceConfig 指定的负载均衡策略

   2. 如果至少有一个地址的类型是 GRPCLB, 那么使用 GRPCLB

   3. 使用 WithBalancer 提供的负载均衡策略， 不过这个选择已经被优化掉了，可以查看字段 LB 上面的注释：

      > ```go
      > // LB is the load balancer the service providers recommends. The balancer
      > // specified via grpc.WithBalancerName will override this.  This is deprecated;
      > // lbConfigs is preferred.  If lbConfig and LB are both present, lbConfig
      > // will be used.
      > LB *string
      > ```

      说明，如果使用了 WithBalancerName 来指定负载均衡策略，那么会覆盖掉这个选项，即会**优先使用客户端配置的负载均衡策略**

   4. 使用 pick_first 负载均衡策略



在完成负载均衡策略的初始化或切换之后，会调用`bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})` 来更新连接的状态（bw 就是 balancerWrapper），这个方法在后面会解析。



## 4. Balancer 分析

Balancer 包的结构跟 Resolver 的包结构类似，都是采用 Builder + Wrapper 设计模式相结合，使用者只需要实现自己的 Balancer，然后再以插件的形式注册到 gRPC 中就可以与其他模块配合使用。

### newCCBalancerWrapper

新建一个 ccBalancerWrapper，并且和 ccResolverWrapper 一样，存储到 `ClientConn` 的 `balancerWrapper` 中：

``` go
func newCCBalancerWrapper(cc *ClientConn, b balancer.Builder, bopts balancer.BuildOptions) *ccBalancerWrapper {
	// 初始化实例
  ccb := &ccBalancerWrapper{
		cc:       cc,
		scBuffer: buffer.NewUnbounded(),
		done:     grpcsync.NewEvent(),
		subConns: make(map[*acBalancerWrapper]struct{}),
	}
  // 启动协程监听
	go ccb.watcher()
  // Build 负载均衡器
	ccb.balancer = b.Build(ccb, bopts)
	return ccb
}
```

### ccBalancerWrapper.watcher

在创建 ccBalancerWrapper 会启动一个goroutine 来执行 watcher 方法，那 *watcher* 的作用是什么呢？

``` go
// watcher balancer functions sequentially, so the balancer can be implemented
// lock-free.
func (ccb *ccBalancerWrapper) watcher() {
	for {
		select {
		case t := <-ccb.scBuffer.Get():
			ccb.scBuffer.Load()
			if ccb.done.HasFired() {
				break
			}
			ccb.balancerMu.Lock()
			su := t.(*scStateUpdate)
			ccb.balancer.UpdateSubConnState(su.sc, balancer.SubConnState{ConnectivityState: su.state, ConnectionError: su.err})
			ccb.balancerMu.Unlock()
		case <-ccb.done.Done():
		}

		if ccb.done.HasFired() {
			ccb.balancer.Close()
			ccb.mu.Lock()
			scs := ccb.subConns
			ccb.subConns = nil
			ccb.mu.Unlock()
			for acbw := range scs {
				ccb.cc.removeAddrConn(acbw.getAddrConn(), errConnDrain)
			}
			ccb.UpdateState(balancer.State{ConnectivityState: connectivity.Connecting, Picker: nil})
			return
		}
	}
}
```

从代码的注释中，我们可以知道为什么要专门启动一个 goroutine 来执行 watcher 方法：为了监听服务器地址的更新和状态变化，并且让 balancer 实现无锁更新，后文会详细讲解。

### balancer.Builder

和 Resolver 一样，在实现 Balancer 的组件的同时也要实现一个 Builder，用来构建 Balancer 实例。以 `grpclb` 负载均衡器为例：

``` go
func (b *lbBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
	// This generates a manual resolver builder with a fixed scheme. This
	// scheme will be used to dial to remote LB, so we can send filtered
	// address updates to remote LB ClientConn using this manual resolver.
  // grpclb 内部的 resolver，有特殊用途
	r := &lbManualResolver{scheme: "grpclb-internal", ccb: cc}

	lb := &lbBalancer{
    // 有缓存功能的 ClientConn wrapper
		cc:              newLBCacheClientConn(cc),
		target:          opt.Target.Endpoint,
		opt:             opt,
		fallbackTimeout: b.fallbackTimeout,
		doneCh:          make(chan struct{}),

		manualResolver: r,
		subConns:       make(map[resolver.Address]balancer.SubConn),
		scStates:       make(map[balancer.SubConn]connectivity.State),
    // 初始化 picker
		picker:         &errPicker{err: balancer.ErrNoSubConnAvailable},
		clientStats:    newRPCStats(),
		backoff:        backoff.DefaultExponential, // TODO: make backoff configurable.
	}

	...

	return lb
}
```

### Balancer 接口

我们再来看看 Balancer 都有哪些方法：

``` go
// Balancer takes input from gRPC, manages SubConns, and collects and aggregates
// the connectivity states.
//
// It also generates and updates the Picker used by gRPC to pick SubConns for RPCs.
//
// UpdateClientConnState, ResolverError, UpdateSubConnState, and Close are
// guaranteed to be called synchronously from the same goroutine.  There's no
// guarantee on picker.Pick, it may be called anytime.
type Balancer interface {
	// 当 ClientConn 的状态发生变化时, gRPC 会调用这个方法
	// 如果返回的错误是ErrBadResolverState, ClientConn 会立即以指数退避的方式调用 Resolver 的 ResolveNow 方法,
	// 直到调用 UpdateClientConnState 返回 nil
	UpdateClientConnState(ClientConnState) error
	// 当 resolver 解析发生错误时, gRPC 会调用这个方法
	ResolverError(error)
	// 当子连接状态发生变化时，gRPC 会调用这个方法
	UpdateSubConnState(SubConn, SubConnState)
	
	Close()
}
```

### lbBalancer.UpdateClientConnState

UpdateClientConnState 是如何被调用的呢？原来是通过前文提到的 `bw.updateClientConnState` 来间接调用的，由 Resolver 到 ClientConn，再到 Balancer，在下面的调用链中一览无余：

``` go
ccResolverWrapper.UpdateState -> ClientConn.updateResolverState -> ccBalancerWrapper.updateClientConnState -> balancer.Balancer.UpdateClientConnState
```

上面的调用关系中不涉及到具体的实现，我们来看看 `lbBalancer.UpdateClientConnState` 是如何实现的：

``` go
func (lb *lbBalancer) UpdateClientConnState(ccs balancer.ClientConnState) error {
	if logger.V(2) {
		logger.Infof("lbBalancer: UpdateClientConnState: %+v", ccs)
	}
	// 处理 balancer 配置
	gc, _ := ccs.BalancerConfig.(*grpclbServiceConfig)
	lb.handleServiceConfig(gc)

	addrs := ccs.ResolverState.Addresses

	// 区分地址的类型, 并且把 GRPCLB 类型的地址改为了 Backend
	var remoteBalancerAddrs, backendAddrs []resolver.Address
	for _, a := range addrs {
		if a.Type == resolver.GRPCLB {
			a.Type = resolver.Backend
			remoteBalancerAddrs = append(remoteBalancerAddrs, a)
		} else {
			backendAddrs = append(backendAddrs, a)
		}
	}
	if sd := grpclbstate.Get(ccs.ResolverState); sd != nil {
		// Override any balancer addresses provided via
		// ccs.ResolverState.Addresses.
		remoteBalancerAddrs = sd.BalancerAddresses
	}

	if len(backendAddrs)+len(remoteBalancerAddrs) == 0 {
		// There should be at least one address, either grpclb server or
		// fallback. Empty address is not valid.
		return balancer.ErrBadResolverState
	}

	if len(remoteBalancerAddrs) == 0 {
		if lb.ccRemoteLB != nil {
			lb.ccRemoteLB.close()
			lb.ccRemoteLB = nil
		}
	} else if lb.ccRemoteLB == nil {
		// 第一次收到解析到的 remoteBalancerAddrs 地址, 建立与远程负载均衡器的连接
		lb.newRemoteBalancerCCWrapper()
		// Start the fallback goroutine.
		go lb.fallbackToBackendsAfter(lb.fallbackTimeout)
	}

	if lb.ccRemoteLB != nil {
		// cc to remote balancers uses lb.manualResolver. Send the updated remote
		// balancer addresses to it through manualResolver.
		lb.manualResolver.UpdateState(resolver.State{Addresses: remoteBalancerAddrs})
	}

	lb.mu.Lock()
	lb.resolvedBackendAddrs = backendAddrs
	if len(remoteBalancerAddrs) == 0 || lb.inFallback {
		// If there's no remote balancer address in ClientConn update, grpclb
		// enters fallback mode immediately.
		//
		// If a new update is received while grpclb is in fallback, update the
		// list of backends being used to the new fallback backends.
		lb.refreshSubConns(lb.resolvedBackendAddrs, true, lb.usePickFirst)
	}
	lb.mu.Unlock()
	return nil
}
```

lbBalancer 把地址分为两种类型：`resolver.GRPCLB`，`resolver.Backend`。当 dnsResolver 传过来的地址列表中有 resolver.GRPCLB，lbBalancer 会与`resolver.GRPCLB`的负载均衡器建立连接以获取真正服务的地址。不过，我们在这里并不做深究。

### lbBalancer.newRemoteBalancerCCWrapper

``` go
func (lb *lbBalancer) refreshSubConns(backendAddrs []resolver.Address, fallback bool, pickFirst bool) {
	opts := balancer.NewSubConnOptions{}
	if !fallback {
		opts.CredsBundle = lb.grpclbBackendCreds
	}

	// 更新后端地址
	lb.backendAddrs = backendAddrs
	lb.backendAddrsWithoutMetadata = nil

	fallbackModeChanged := lb.inFallback != fallback
	lb.inFallback = fallback
	if fallbackModeChanged && lb.inFallback {
		// Clear previous received list when entering fallback, so if the server
		// comes back and sends the same list again, the new addresses will be
		// used.
		lb.fullServerList = nil
	}

	// 判断选取策略是否变化，获取第一个可用连接
	balancingPolicyChanged := lb.usePickFirst != pickFirst
	oldUsePickFirst := lb.usePickFirst
	lb.usePickFirst = pickFirst

	if fallbackModeChanged || balancingPolicyChanged {
		// 如果策略发生变化，删除所有的已经建立的连接
		for a, sc := range lb.subConns {
			if oldUsePickFirst {
				// If old SubConn were created for pickfirst, bypass cache and
				// remove directly.
				lb.cc.cc.RemoveSubConn(sc)
			} else {
				lb.cc.RemoveSubConn(sc)
			}
			delete(lb.subConns, a)
		}
	}
	// 使用 pickFirst 策略,只需要获取一个子连接就 ok 了
	if lb.usePickFirst {
		var sc balancer.SubConn
		for _, sc = range lb.subConns {
			break
		}
		// 更新后端服务地址并触发连接
		if sc != nil {
			sc.UpdateAddresses(backendAddrs)
			sc.Connect()
			return
		}
		// This bypasses the cc wrapper with SubConn cache.
		// 新建一个 SubConn, 并连接
		sc, err := lb.cc.cc.NewSubConn(backendAddrs, opts)
		if err != nil {
			logger.Warningf("grpclb: failed to create new SubConn: %v", err)
			return
		}
		sc.Connect()
		lb.subConns[backendAddrs[0]] = sc
		lb.scStates[sc] = connectivity.Idle
		return
	}

	// addrsSet is the set converted from backendAddrsWithoutMetadata, it's used to quick
	// lookup for an address.
	addrsSet := make(map[resolver.Address]struct{})
	// Create new SubConns.
	for _, addr := range backendAddrs {
		addrWithoutMD := addr
		addrWithoutMD.Metadata = nil
		addrsSet[addrWithoutMD] = struct{}{}
		lb.backendAddrsWithoutMetadata = append(lb.backendAddrsWithoutMetadata, addrWithoutMD)

		if _, ok := lb.subConns[addrWithoutMD]; !ok {
			// Use addrWithMD to create the SubConn.
			sc, err := lb.cc.NewSubConn([]resolver.Address{addr}, opts)
			if err != nil {
				logger.Warningf("grpclb: failed to create new SubConn: %v", err)
				continue
			}
			lb.subConns[addrWithoutMD] = sc // Use the addr without MD as key for the map.
			if _, ok := lb.scStates[sc]; !ok {
				// Only set state of new sc to IDLE. The state could already be
				// READY for cached SubConns.
				lb.scStates[sc] = connectivity.Idle
			}
			sc.Connect()
		}
	}

	for a, sc := range lb.subConns {
		// a was removed by resolver.
		if _, ok := addrsSet[a]; !ok {
			lb.cc.RemoveSubConn(sc)
			delete(lb.subConns, a)
			// Keep the state of this sc in b.scStates until sc's state becomes Shutdown.
			// The entry will be deleted in UpdateSubConnState.
		}
	}

	// Regenerate and update picker after refreshing subconns because with
	// cache, even if SubConn was newed/removed, there might be no state
	// changes (the subconn will be kept in cache, not actually
	// newed/removed).
	// 更新 lb 的状态, 并且重新生成 Picker
	lb.updateStateAndPicker(true, true)
}
```

### lbBalancer.updateStateAndPicker

``` go
func (lb *lbBalancer) updateStateAndPicker(forceRegeneratePicker bool, resetDrop bool) {
	oldAggrState := lb.state
	lb.state = lb.aggregateSubConnStates()
	// Regenerate picker when one of the following happens:
	//  - caller wants to regenerate
	//  - the aggregated state changed
	if forceRegeneratePicker || (lb.state != oldAggrState) {
    // 重新生成 picker
		lb.regeneratePicker(resetDrop)
	}

	lb.cc.UpdateState(balancer.State{ConnectivityState: lb.state, Picker: lb.picker})
}
```

### lbBalancer.regeneratePicker

``` go
func (lb *lbBalancer) regeneratePicker(resetDrop bool) {
  // 创建 errpicker
	if lb.state == connectivity.TransientFailure {
		lb.picker = &errPicker{err: balancer.ErrTransientFailure}
		return
	}
	// 创建 errpicker
	if lb.state == connectivity.Connecting {
		lb.picker = &errPicker{err: balancer.ErrNoSubConnAvailable}
		return
	}
	
  // 如果是 pickFirst， 则获取第一个 subConn 就行了
	var readySCs []balancer.SubConn
	if lb.usePickFirst {
		for _, sc := range lb.subConns {
			readySCs = append(readySCs, sc)
			break
		}
	} else {
		for _, a := range lb.backendAddrsWithoutMetadata {
			if sc, ok := lb.subConns[a]; ok {
				if st, ok := lb.scStates[sc]; ok && st == connectivity.Ready {
					readySCs = append(readySCs, sc)
				}
			}
		}
	}
	
	if len(readySCs) <= 0 {
		// If there's no ready SubConns, always re-pick. This is to avoid drops
		// unless at least one SubConn is ready. Otherwise we may drop more
		// often than want because of drops + re-picks(which become re-drops).
		//
		// This doesn't seem to be necessary after the connecting check above.
		// Kept for safety.
		lb.picker = &errPicker{err: balancer.ErrNoSubConnAvailable}
		return
	}
  // 如果是 fallback，则使用 RoundRobin
	if lb.inFallback {
		lb.picker = newRRPicker(readySCs)
		return
	}
  // 如果是需要重置，创建新的 Picker
	if resetDrop {
		lb.picker = newLBPicker(lb.fullServerList, readySCs, lb.clientStats)
		return
	}
  // 原来的 Picker 不是 lbPicker，创建新 Picker
	prevLBPicker, ok := lb.picker.(*lbPicker)
	if !ok {
		lb.picker = newLBPicker(lb.fullServerList, readySCs, lb.clientStats)
		return
	}
  // 更新 SubConn
	prevLBPicker.updateReadySCs(readySCs)
}
```

## 5. 后端服务建立连接

虽然前面剖析了 Balancer 的实现原理，但我们却并不知道与后端服务的连接是在哪里建立的，答案就在 `ClientConn.NewSubConn`中：

``` go
func (ccb *ccBalancerWrapper) NewSubConn(addrs []resolver.Address, opts balancer.NewSubConnOptions) (balancer.SubConn, error) {
	if len(addrs) <= 0 {
		return nil, fmt.Errorf("grpc: cannot create SubConn with empty address list")
	}
	ccb.mu.Lock()
	defer ccb.mu.Unlock()
	if ccb.subConns == nil {
		return nil, fmt.Errorf("grpc: ClientConn balancer wrapper was closed")
	}
  // 调用 grpc.ClientConn.newAddrConn 来创建于后端服务的连接
	ac, err := ccb.cc.newAddrConn(addrs, opts)
	if err != nil {
		return nil, err
	}
	acbw := &acBalancerWrapper{ac: ac}
	acbw.ac.mu.Lock()
	ac.acbw = acbw
	acbw.ac.mu.Unlock()
  // 
	ccb.subConns[acbw] = struct{}{}
	return acbw, nil
}
```

在 NewSubConn 中会调用 `cc.newAddrConn` 来建立连接对象 `acBalancerWrapper` 并把它添加到 ClientConn 的 conns 连接池中，但是并没有真正建立连接，而只是初始化了连接状态为 `Idle`，然后返回了连接的实例：

``` go
// newAddrConn creates an addrConn for addrs and adds it to cc.conns.
//
// Caller needs to make sure len(addrs) > 0.
func (cc *ClientConn) newAddrConn(addrs []resolver.Address, opts balancer.NewSubConnOptions) (*addrConn, error) {
	ac := &addrConn{
		state:        connectivity.Idle,	// 初始化状态为 Idle
		cc:           cc,
		addrs:        addrs,
		scopts:       opts,
		dopts:        cc.dopts,
		czData:       new(channelzData),
		resetBackoff: make(chan struct{}),
	}
	ac.ctx, ac.cancel = context.WithCancel(cc.ctx)
	// Track ac in cc. This needs to be done before any getTransport(...) is called.
	cc.mu.Lock()
	if cc.conns == nil {
		cc.mu.Unlock()
		return nil, ErrClientConnClosing
	}
	if channelz.IsOn() {
		ac.channelzID = channelz.RegisterSubChannel(ac, cc.channelzID, "")
		channelz.AddTraceEvent(logger, ac.channelzID, 0, &channelz.TraceEventDesc{
			Desc:     "Subchannel Created",
			Severity: channelz.CtInfo,
			Parent: &channelz.TraceEventDesc{
				Desc:     fmt.Sprintf("Subchannel(id:%d) created", ac.channelzID),
				Severity: channelz.CtInfo,
			},
		})
	}
	cc.conns[ac] = struct{}{}
	cc.mu.Unlock()
	return ac, nil
}
```

### acBalancerWrapper

acBalancerWrapper 是 addrConn 的 wrapper，实现了 SubConn 接口：

``` go 
// acBalancerWrapper is a wrapper on top of ac for balancers.
// It implements balancer.SubConn interface.
type acBalancerWrapper struct {
	mu sync.Mutex
	ac *addrConn
}

// SubConn represents a gRPC sub connection.
// Each sub connection contains a list of addresses. gRPC will
// try to connect to them (in sequence), and stop trying the
// remainder once one connection is successful.
// gRPC 会按顺序尝试连接所有的地址, 在成功连接一个之后停止连接
//
// The reconnect backoff will be applied on the list, not a single address.
// For example, try_on_all_addresses -> backoff -> try_on_all_addresses.
//
// All SubConns start in IDLE, and will not try to connect. To trigger
// the connecting, Balancers must call Connect.
// When the connection encounters an error, it will reconnect immediately.
// When the connection becomes IDLE, it will not reconnect unless Connect is
// called.
//
// This interface is to be implemented by gRPC. Users should not need a
// brand new implementation of this interface. For the situations like
// testing, the new implementation should embed this interface. This allows
// gRPC to add new methods to this interface.
type SubConn interface {
	// UpdateAddresses updates the addresses used in this SubConn.
	// gRPC checks if currently-connected address is still in the new list.
	// If it's in the list, the connection will be kept.
	// If it's not in the list, the connection will gracefully closed, and
	// a new connection will be created.
	//
	// This will trigger a state transition for the SubConn.
	UpdateAddresses([]resolver.Address)
	// Connect starts the connecting for this SubConn.
	Connect()
}
```

在完成 NewSubConn 的调用后，还需要手动调用 `SubConn.Connect` 来完成连接的建立，在 Connect 中又会调用 `addrConn.connect`：

``` go
func (acbw *acBalancerWrapper) Connect() {
	acbw.mu.Lock()
	defer acbw.mu.Unlock()
	acbw.ac.connect()
}

func (ac *addrConn) connect() error {
	ac.mu.Lock()
  // 连接已经关闭，返回错误
	if ac.state == connectivity.Shutdown {
		ac.mu.Unlock()
		return errConnClosing
	}
  // 连接状态不是 Idle，说明正在连接中，或者已经创建了连接
	if ac.state != connectivity.Idle {
		ac.mu.Unlock()
		return nil
	}
	// Update connectivity state within the lock to prevent subsequent or
	// concurrent calls from resetting the transport more than once.
  // 把连接状态更新为连接中
	ac.updateConnectivityState(connectivity.Connecting, nil)
	ac.mu.Unlock()

	// Start a goroutine connecting to the server asynchronously.
  // 异步连接服务器
	go ac.resetTransport()
	return nil
}
```

### addrConn.updateConnectivityState

```go
func (ac *addrConn) updateConnectivityState(s connectivity.State, lastErr error) {
   if ac.state == s {
      return
   }
   ac.state = s
   channelz.Infof(logger, ac.channelzID, "Subchannel Connectivity change to %v", s)
   ac.cc.handleSubConnStateChange(ac.acbw, s, lastErr)
}
```

updateConnectivityState 会调用 `ClientConn.handleSubConnStateChange` 来通知 gRPC 子连接状态发生变化，那 ClientConn 又是如何通知 Balancer 的呢？

调用链如下：

``` go
addrConn.updateConnectivityState -> ClientConn.handleSubConnStateChange -> ccBalancerWrapper.handleSubConnStateChange -> ccb.scBuffer.Put(&scStateUpdate{
		sc:    sc,
		state: s,
		err:   err,
	}) // 发送到 channel
```

还记得 watcher 函数吗？在 Build 的时候会启动一个协程来执行这个方法：

``` go
// watcher balancer functions sequentially, so the balancer can be implemented
// lock-free.
func (ccb *ccBalancerWrapper) watcher() {
	for {
		select {
		// handleSubConnStateChange触发
		case t := <-ccb.scBuffer.Get():
			ccb.scBuffer.Load()
			if ccb.done.HasFired() {
				break
			}
			ccb.balancerMu.Lock()
			su := t.(*scStateUpdate)
			ccb.balancer.UpdateSubConnState(su.sc, balancer.SubConnState{ConnectivityState: su.state, ConnectionError: su.err})
			ccb.balancerMu.Unlock()
		case <-ccb.done.Done():
		}

		if ccb.done.HasFired() {
			ccb.balancer.Close()
			ccb.mu.Lock()
			scs := ccb.subConns
			ccb.subConns = nil
			ccb.mu.Unlock()
			for acbw := range scs {
				ccb.cc.removeAddrConn(acbw.getAddrConn(), errConnDrain)
			}
			ccb.UpdateState(balancer.State{ConnectivityState: connectivity.Connecting, Picker: nil})
			return
		}
	}
}
```

哈哈，我们又回到了这里。看来 watcher 方法的工作就是监听子连接的状态变化，并且调用 `balancer.UpdateSubConnState` 来执行更新操作。

### balancer.UpdateSubConnState

``` go
func (lb *lbBalancer) UpdateSubConnState(sc balancer.SubConn, scs balancer.SubConnState) {
	s := scs.ConnectivityState
	if logger.V(2) {
		logger.Infof("lbBalancer: handle SubConn state change: %p, %v", sc, s)
	}
	lb.mu.Lock()
	defer lb.mu.Unlock()

	oldS, ok := lb.scStates[sc]
	if !ok {
		if logger.V(2) {
			logger.Infof("lbBalancer: got state changes for an unknown SubConn: %p, %v", sc, s)
		}
		return
	}
	lb.scStates[sc] = s
	switch s {
	case connectivity.Idle:	// Idle，则重新建立连接
		sc.Connect()
	case connectivity.Shutdown:	// Shutdown， 删除当前连接
		// When an address was removed by resolver, b called RemoveSubConn but
		// kept the sc's state in scStates. Remove state for this sc here.
		delete(lb.scStates, sc)
	}
	// Force regenerate picker if
	//  - this sc became ready from not-ready
	//  - this sc became not-ready from ready
  // 更新状态，已经重新生成 picker
	lb.updateStateAndPicker((oldS == connectivity.Ready) != (s == connectivity.Ready), false)

	// Enter fallback when the aggregated state is not Ready and the connection
	// to remote balancer is lost.
	if lb.state != connectivity.Ready {
		if !lb.inFallback && !lb.remoteBalancerConnected {
			// Enter fallback.
			lb.refreshSubConns(lb.resolvedBackendAddrs, true, lb.usePickFirst)
		}
	}
}
```



## 6. 总结

设计者为了让 Resolver ，Balancer 实现解耦，可谓是煞费苦心，而且也实现的十分优秀。当然 dnsBalancer 的实现相对来说有点复杂，如果没有使用过 grpclb 的话，理解起来会有点困难，不过这并不影响我们学习 Balancer 的设计思想，下面是调用关系总结：

![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/resolver-balancer-clientconn%E4%BA%A4%E4%BA%92%E6%B5%81%E7%A8%8B.png)

咱们下篇文章见！