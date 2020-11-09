---
title: "gRPC Resolver 分析"
date: 2020-10-29T09:56:05+08:00
draft: false
---

## 1. 前言

gRPC Resolver 是 gRPC 的核心功能之一，与 gRPC Balancer 一起负责 gRPC 调用其它服务时的负载均衡。gRPC 负载均衡是针对每次请求，而不是连接，这样可以保证服务端负载的均衡性，所有 gRPC 负载均衡算法实现都在客户端。

## 2. Resolver

下图是 gRPC 客户端负载均衡的架构图：

<img src="https://raw.githubusercontent.com/grpc/proposal/master/L9_graphics/bar_after.png" />

从图中可以看出，Resolver 位于图片的左方，gRPC 负载与 Resolver 交互：

1. 首先会 Build 一个 Resolver 实例，并且 watch 后端服务列表的变化
2. 当服务列表发生变化后，Resolver 通过 gRPC 通知 Balancer



## 3. 基本概念

`ClientConn` 对象是连接管理的入口，表示到服务端的一个逻辑的连接，会做名字解析、负载均衡、KeepAlive 等连接管理方面的操作，是个线程安全的对象。

每个 `ClientConn` 对应有多个 `SubConn`，`ClientConn` 会基于服务发现（resolver）得到多个 SubConn，并在多个 `SubConn` 之间实现负载均衡（balancer）。



## 4. 源码分析

Resolver 的代码主要集中在 resolver 包中，里面主要包含了服务解析的接口定义，我们既可以自己通过实现 resolver 中的接口来自定义自己的 Resolver，也可以使用 gRPC 实现的 [DNSResolver](https://github.com/grpc/grpc-go/blob/master/internal/resolver/dns/dns_resolver.go)。

### Address

Address 用来代表一个当前客户端即将连接到的服务端的地址，一个服务一般会有多个地址，所有我们在监听的时候一般会获取到 Address 的切片。其中的 `Addr` 字段是服务器的地址, `Attributes`  和 `Metadata` 用来存储服务端的额外的信息，一般被负载均衡器用来决定 pick 哪一条连接：

``` go
// Address represents a server the client connects to.
//
// Experimental
//
// Notice: This type is EXPERIMENTAL and may be changed or removed in a
// later release.
type Address struct {
	// Addr is the server address on which a connection will be established.
	Addr string

	// ServerName is the name of this address.
	// If non-empty, the ServerName is used as the transport certification authority for
	// the address, instead of the hostname from the Dial target string. In most cases,
	// this should not be set.
	//
	// If Type is GRPCLB, ServerName should be the name of the remote load
	// balancer, not the name of the backend.
	//
	// WARNING: ServerName must only be populated with trusted values. It
	// is insecure to populate it with data from untrusted inputs since untrusted
	// values could be used to bypass the authority checks performed by TLS.
	ServerName string

	// Attributes contains arbitrary data about this address intended for
	// consumption by the load balancing policy.
	Attributes *attributes.Attributes

	// Type is the type of this address.
	//
	// Deprecated: use Attributes instead.
	Type AddressType

	// Metadata is the information associated with Addr, which may be used
	// to make load balancing decision.
	//
	// Deprecated: use Attributes instead.
	Metadata interface{}
}
```

### ClientConn

ClientConn 为 resolver 提供了通知 ClientConn 更新服务端列表的回调方法。这个接口不推荐用户自己实现：

``` go
// ClientConn contains the callbacks for resolver to notify any updates
// to the gRPC ClientConn.
//
// This interface is to be implemented by gRPC. Users should not need a
// brand new implementation of this interface. For the situations like
// testing, the new implementation should embed this interface. This allows
// gRPC to add new methods to this interface.
type ClientConn interface {
	// UpdateState updates the state of the ClientConn appropriately.
	UpdateState(State)
	// ReportError notifies the ClientConn that the Resolver encountered an
	// error.  The ClientConn will notify the load balancer and begin calling
	// ResolveNow on the Resolver with exponential backoff.
	ReportError(error)
	// NewAddress is called by resolver to notify ClientConn a new list
	// of resolved addresses.
	// The address list should be the complete list of resolved addresses.
	//
	// Deprecated: Use UpdateState instead.
	NewAddress(addresses []Address)
	// NewServiceConfig is called by resolver to notify ClientConn a new
	// service config. The service config should be provided as a json string.
	//
	// Deprecated: Use UpdateState instead.
	NewServiceConfig(serviceConfig string)
	// ParseServiceConfig parses the provided service config and returns an
	// object that provides the parsed config.
	ParseServiceConfig(serviceConfigJSON string) *serviceconfig.ParseResult
}
```

一般我们的 Resolver 中都会用一个成员变量来存储 ClientConn，然后再需要更新服务端地址的时候调用 `UpdateState`。例如 dnsResolver 中存储了 `cc`：

``` go
type dnsResolver struct {
	...
	cc       resolver.ClientConn
	...
}
```

### Target

Target 是一个存储注册中心信息和后端服务信息的结构体, 并且通过 Scheme 指定了 gRPC 使用的解析器的名称：

``` go
// Target represents a target for gRPC, as specified in:
// https://github.com/grpc/grpc/blob/master/doc/naming.md.
// It is parsed from the target string that gets passed into Dial or DialContext by the user. And
// grpc passes it to the resolver and the balancer.
type Target struct {
	Scheme    string	// 注册在 gRPC 中的名称
	Authority string	// 服务发现的权威服务器
	Endpoint  string	// 一般是服务名或者HOST
}
```

例如我们有一个 Target 是 `dns://some_authority/foo.bar`， 那么对应的被解析出来就应该是

``` go
Scheme = dns
Authority = some_authority
Endpoint = foo.bar
```

### Builder

Builder 接口为我们提供了创建 Resolver 的 Build 方法，并且持续监听服务端地址的变化：

``` go
// Builder creates a resolver that will be used to watch name resolution updates.
type Builder interface {
	// Build creates a new resolver for the given target.
	//
	// gRPC dial calls Build synchronously, and fails if the returned error is
	// not nil.
	Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
	// Scheme returns the scheme supported by this resolver.
	// Scheme is defined at https://github.com/grpc/grpc/blob/master/doc/naming.md.
	Scheme() string
}
```

从注释中，我们可以理解到：当向 gRPC 注册解析器的时候，实际上注册的是 Builder，通过 Build 方法来创建 Resolver，并且一般会在 Build 方法中开启 goroutine 来 watch 服务端的变化。

Build 方法的参数就包含了上节提到的 `cc ClientConn`，然后如上文说的，把 cc 存储到 Resolver 中，在观察到服务地址发生变化的时候

通过 `cc.UpdateState(resolver.State{Addresses: addr})` 来通知 ClientConn 服务器列表发生变化。

### Resolver

Resolver 被 Builder 创建出来，实现了监听服务名映射到的地址变化的逻辑：

``` go
// Resolver watches for the updates on the specified target.
// Updates include address updates and service config updates.
type Resolver interface {
	// ResolveNow will be called by gRPC to try to resolve the target name
	// again. It's just a hint, resolver can ignore this if it's not necessary.
	//
	// It could be called multiple times concurrently.
	ResolveNow(ResolveNowOptions)
	// Close closes the resolver.
	Close()
}
```

### 小结

resolver 包的使用流程是，通过 Builder 接口来创建 Resolver，我们可以再 Resolver 中实现自己服务发现的逻辑，并且更新到 ClientConn 中。



## 5. Resolver 应用

下面我们写一个完整的例子，用来分析 Resolver 的工作流程，我们程序的目的就是客户端通过 rpc 调用服务器的接口并打印出结果。为了减少篇幅，我就不贴出所有的代码，完整的代码可以在我的 [github](https://github.com/wangzeping722/gRPC-resolver-demo) 获取。

### server/main.go

通过 NewGreeterService 向 etcd 注册了服务的地址：

``` go
func (g *greeterService) Register() {
	target := fmt.Sprintf("%s", serviceName)
	ctx, _ := context.WithTimeout(context.Background(), 5*time.Second)

	_, err := g.cli.Put(ctx, target, "127.0.0.1:8000")
	if err != nil {
		panic(err)
	}
}
```

然后，监听在 8000 这个端口上，等待提供服务。

### client/main.go

由于本文的重心在于分析 Resolver，那么 client 的代码实现才是我们需要详细解析的。为了能够从 etcd 获取我们后端服务的地址，并且 gRPC 只提供了 dnsResolver，那么我们就要实现自己自定义的 Resolver，这里我们只实现了一个功能非常简单，并且没有 watch（即持续的服务发现）的 Resolver：

``` go
type exampleResolver struct {
	target resolver.Target
	cc     resolver.ClientConn
	cli    *clientv3.Client
}

func (e *exampleResolver) ResolveNow(_ resolver.ResolveNowOptions) {
	fmt.Println("ResolverNow")
}

func (e *exampleResolver) Close() {
	fmt.Println("Close")
}

func (e *exampleResolver) resolve() {
	once.Do(func() {
		clientConfig := clientv3.Config{
			Endpoints:   []string{e.target.Authority},
			DialTimeout: 120 * time.Second,
		}
		cli, err := clientv3.New(clientConfig)
		if err != nil {
			panic(err)
		}
		e.cli = cli
	})
	addList := make([]resolver.Address, 0)
  // 获取服务对应的所有的后端地址，并且添加到 addList
	if result, err := e.cli.Get(context.Background(), e.target.Endpoint, clientv3.WithPrefix()); err == nil {
		for _, kvs := range result.Kvs {
			addList = append(addList, resolver.Address{
				Addr: string(kvs.Value),
			})
			fmt.Printf("resolved addr: %s\n", string(kvs.Value))
		}
	}
	e.cc.UpdateState(resolver.State{Addresses: addList})
}

```

实现 exampleResolver， 可以看到，我们在 resolver 方法中实现了服务发现的代码，获得了服务对应的后端地址，然后通过 `cc.UpdateState(resolver.State{Addresses: addList})` 更新（添加、删除）本地的后端服务器列表。

同时，我们还需要实现一个 Builder，用来创建 Resolver 实例，同时我们还需要把 Builder 注册到 gRPC 中，通过 `Scheme` 来标识我们自定义的 Resolver：

``` go
func init() {
	resolver.Register(newBuilder())
}

type exampleBuilder struct {
}

func newBuilder() *exampleBuilder {
	return &exampleBuilder{}
}

func (b *exampleBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	r := &exampleResolver{cc: cc, target: target}
	r.resolve()
	return r, nil
}

func (b *exampleBuilder) Scheme() string {
	return scheme
}
```

再来看看 main 函数做了什么工作：

``` go
func main() {
	target := fmt.Sprintf("%s://%s/%s", scheme, "127.0.0.1:2379", serviceName)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
	defer cancel()
	clientConn, err := grpc.DialContext(ctx, target, grpc.WithInsecure())
	if err != nil {
		panic(err)
	}

  // 调用远程服务
	reply, err := hello.NewGreeterClient(clientConn).SayHello(context.Background(), &hello.HelloRequest{
		Name: "wero",
	})
	if err != nil {
		panic(err)
	}
	fmt.Println("get reply:", reply.Message)
}
```

首先，我们通过 `target` 指出需要调用的远程服务，然后调用  `grpc.DialContext`，gRPC 便可以自动发现服务的地址了，并在我们调用远程服务的时候实现了负载均衡。不过我们不仅要知其然，更要知其所以然。

### DialContext 流程分析

最重要的便是 DialContext 这个函数了，那我们便从这个函数入手：

``` go
func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		target:            target,
		csMgr:             &connectivityStateManager{},
		conns:             make(map[*addrConn]struct{}),	// 存储连接的 map
		dopts:             defaultDialOptions(),
		blockingpicker:    newPickerWrapper(),				// 负载均衡的选择器
		czData:            new(channelzData),
		firstResolveEvent: grpcsync.NewEvent(),
	}
  
	...
	// 初始化拦截器
	chainUnaryClientInterceptors(cc)
	chainStreamClientInterceptors(cc)
	
  ...
  
	// Determine the resolver to use.
	// 根据 target 上面置顶的 scheme 指定需要使用的 resolver
	cc.parsedTarget = grpcutil.ParseTarget(cc.target)
	unixScheme := strings.HasPrefix(cc.target, "unix:")
	channelz.Infof(logger, cc.channelzID, "parsed scheme: %q", cc.parsedTarget.Scheme)
	// NameResolver 核心逻辑, 初始化 resolverBuilder, 如果传入的 scheme 找不到对应的 resolverBuilder, 就使用默认的
	resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)
	if resolverBuilder == nil {
		// If resolver builder is still nil, the parsed target's scheme is
		// not registered. Fallback to default resolver and set Endpoint to
		// the original target.
		channelz.Infof(logger, cc.channelzID, "scheme %q not registered, fallback to default scheme", cc.parsedTarget.Scheme)
		cc.parsedTarget = resolver.Target{
			Scheme:   resolver.GetDefaultScheme(),
			Endpoint: target,
		}
		resolverBuilder = cc.getResolver(cc.parsedTarget.Scheme)
		if resolverBuilder == nil {
			return nil, fmt.Errorf("could not get resolver for default scheme: %q", cc.parsedTarget.Scheme)
		}
	}
  
	...
  
	cc.balancerBuildOpts = balancer.BuildOptions{
		DialCreds:        credsClone,
		CredsBundle:      cc.dopts.copts.CredsBundle,
		Dialer:           cc.dopts.copts.Dialer,
		ChannelzParentID: cc.channelzID,
		Target:           cc.parsedTarget,
	}

	// Build the resolver.
	// 使用上面初始化的 resolverBuilder 构建 resolver
	rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)
	if err != nil {
		return nil, fmt.Errorf("failed to build resolver: %v", err)
	}
	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()
  
	...
  
	return cc, nil
}
```

代码中省略了一部分目前不关注的内容，可以看见的是，我们需要先通过 `target` 指定的 `scheme` 来获取 resolverBuilder， 然后再通过 builder 来构建出 resolver 实例。在我们的例子中 `scheme = test` ，其对应的 builder 就是上面的 `exampleBuilder`，代码如下：

``` go
func (cc *ClientConn) getResolver(scheme string) resolver.Builder {
	for _, rb := range cc.dopts.resolvers {
		if scheme == rb.Scheme() {
			return rb
		}
	}
	return resolver.Get(scheme)
}
```

在获取到 builder 之后，紧接着就需要构建出用于服务发现的 `Resolver`，我们接往下看，发现 `newCCResolverWrapper(cc, resolverBuilder)` 会把获取到的 builder 传入函数中：

``` go
func newCCResolverWrapper(cc *ClientConn, rb resolver.Builder) (*ccResolverWrapper, error) {
	ccr := &ccResolverWrapper{
		cc:   cc,
		done: grpcsync.NewEvent(),
	}
	...
	rbo := resolve r.BuildOptions{
		DisableServiceConfig: cc.dopts.disableServiceConfig,
		DialCreds:            credsClone,
		CredsBundle:          cc.dopts.copts.CredsBundle,
		Dialer:               cc.dopts.copts.Dialer,
	}

	var err error
	// We need to hold the lock here while we assign to the ccr.resolver field
	// to guard against a data race caused by the following code path,
	// rb.Build-->ccr.ReportError-->ccr.poll-->ccr.resolveNow, would end up
	// accessing ccr.resolver which is being assigned here.
	ccr.resolverMu.Lock()
	defer ccr.resolverMu.Unlock()
	ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, rbo)
	if err != nil {
		return nil, err
	}
	return ccr, nil
}
```

果然，这个函数会调用传入的 Builder，而这个 Builder 是一个接口，具体的实现是我们自定义的 `exampleBuilder`，在 Build 方法里面，我们一般会开启一个协程持续的获取后端服务器列表的状态，并通过 `	UpdateState(State)` ，来更新到 ClientConn 中（不过，在我这个建议的实现中，并没有开启协程来处理服务发现，而是一次性的获取，假设后端服务永远不出故障）。

### 总结

到这里，对 Resolver 的分析基本完成，下一篇文章会分析 gRPC Balancer。

