---
title: "gRPC RoundRobin Picker 分析"
date: 2020-11-09T17:59:31+08:00
draft: false
---

## 前言

上一篇文章提到过：Picker 是负载均衡器里面的一个组件，主要作用就是客户端要进行 rpc 调用的时候，在可用的连接中根据负载均衡策略挑选出最时候的连接交个 gRPC 进行调用。本文简单的分析一下 Picker 的原理，以及官方实现的 RoundRobin Picker。

## 1. Picker

```go
// Picker is used by gRPC to pick a SubConn to send an RPC.
type Picker interface {
   Pick(info PickInfo) (PickResult, error)
}

// PickResult contains information related to a connection chosen for an RPC.
type PickResult struct {
	// SubConn is the connection to use for this pick, if its state is Ready.
	// If the state is not Ready, gRPC will block the RPC until a new Picker is
	// provided by the balancer (using ClientConn.UpdateState).  The SubConn
	// must be one returned by ClientConn.NewSubConn.
	SubConn SubConn

	// Done is called when the RPC is completed.  If the SubConn is not ready,
	// this will be called with a nil parameter.  If the SubConn is not a valid
	// type, Done may not be called.  May be nil if the balancer does not wish
	// to be notified when the RPC completes.
	Done func(DoneInfo)
}

// DoneInfo contains additional information for done.
type DoneInfo struct {
	// Err is the rpc error the RPC finished with. It could be nil.
	Err error
	// Trailer contains the metadata from the RPC's trailer, if present.
	Trailer metadata.MD
	// BytesSent indicates if any bytes have been sent to the server.
	BytesSent bool
	// BytesReceived indicates if any byte has been received from the server.
	BytesReceived bool
	// ServerLoad is the load received from server. It's usually sent as part of
	// trailing metadata.
	//
	// The only supported type now is *orca_v1.LoadReport.
	ServerLoad interface{}
}
```

Picker 接口只有一个 Pick 方法，用来返回一个可用的 `SubConn` 。需要注意的是，Pick 并不是直接返回一个 `SubConn`，而是返回了一个 `PickResult` ，里面包含了一个 `SubConn` ，以及一个 `Done func(DoneInfo)` 的闭包。从文档中可以看出，这个闭包是在 RPC 调用完成之后才被调用的。参数 `DoneInfo` 包含了调用以及服务器返回的信息。有经验的小伙伴肯定已经猜到了：`DoneInfo` 中的 Trailer 能够写到服务端信息（包括 CPU，内存，服务器状态等信息），这样就可以利用 DoneInfo 作为 Picker 以后 Pick 服务器的依据。

## 2. rrPicker 实现

官方的实现很简单，这里就直接贴出代码：

```go
type rrPickerBuilder struct{}

func (*rrPickerBuilder) Build(info base.PickerBuildInfo) balancer.Picker {
	logger.Infof("roundrobinPicker: newPicker called with info: %v", info)
  // 没有可用的连接，返回 errPicker
	if len(info.ReadySCs) == 0 {
		return base.NewErrPicker(balancer.ErrNoSubConnAvailable)
	}
	var scs []balancer.SubConn
	for sc := range info.ReadySCs {
		scs = append(scs, sc)
	}
	return &rrPicker{
		subConns: scs,
		// Start at a random index, as the same RR balancer rebuilds a new
		// picker when SubConn states change, and we don't want to apply excess
		// load to the first server in the list.
		next: grpcrand.Intn(len(scs)),
	}
}

type rrPicker struct {
   // subConns is the snapshot of the roundrobin balancer when this picker was
   // created. The slice is immutable. Each Get() will do a round robin
   // selection from it and return the selected SubConn.
   subConns []balancer.SubConn

   mu   sync.Mutex
   next int
}

func (p *rrPicker) Pick(balancer.PickInfo) (balancer.PickResult, error) {
   p.mu.Lock()
   sc := p.subConns[p.next]
   // 每次选择连接之后都会把 next 加 1，然后对长度取余，实现轮询所有连接
   p.next = (p.next + 1) % len(p.subConns)
   p.mu.Unlock()
   return balancer.PickResult{SubConn: sc}, nil
}
```

首先，负载均衡器会通过 `rrPickerBuilder.Build` 方法使用传入的 `SubConn` 来初始化 `rrPicker` 实例。当需要进行 RPC 调用时，gRPC 会调用 `Pick` 方法来获取一条可用连接。在 `rrPicker` 中，每次调用 Pick 都会把 `rrPicker.next` 值加1，然后对长度取余，实现轮询所有连接。

## 3. 总结

gRPC 实现的 RoundRobin 非常简洁易懂，如果我们有自己实现负载均衡策略的需求，也可以参考 RoundRobin 的写法来构建自己的负载均衡选择器。比如 go-zero 的 [p2c](https://github.com/tal-tech/go-zero/blob/master/zrpc/internal/balancer/p2c/p2c.go) 选择器。

