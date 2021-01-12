---
title: "go-zero源码分析： P2C 负载均衡算法"
date: 2020-01-02T16:30:18+08:00
draft: false
---

看完文章[基于gRPC的注册发现与负载均衡的原理和实战](https://mp.weixin.qq.com/s/olPGfrFczo22rhpPLBmpNw)后大呼过瘾，但是文章中对负载均衡算法—— p2c 算法的介绍不是很详细，于是乎就去学习了一下。

## 1. P2C 负载均衡算法介绍

P2C 的全称是 **Power of Two Choices (P2C，两次随机选择) 负载均衡算法**，相比其他负载均衡算法有着更加科学的 LB 策略，它通过随机选择出两个节点，然后根据一些条件计算出两个节点的负载情况，选择出负载小的那个节点。

伪代码如下：

```  go
node1 := randomNode()
node2 := randomNode()
best := choiceTheBest(node1, node2)
```

P2C 是在客户端侧实现的负载均衡算法，所以不可避免的要有评判节点连接负载高低的方法，只要知道负载率，那么就能够选出最优的节点。在 go-zero 中每个节点的负载情况都是由几个统计字段记录的，并通过这几个字段能够计算出节点连接当前的负载情况。

### 1.1 lag

lag 是节点延迟，根据*EWMA（[指数加权移动平均算法](https://www.cnblogs.com/jiangxinyang/p/9705198.html)）*计算出来，此算法，是对观察值分别给予不同的权数，按不同权数求得移动平均值，并以最后的移动平均值为基础，确定预测值的方法。采用加权移动平均法，是因为观察期的近期观察值对预测值有较大影响，它更能反映近期变化的趋势。

- 指数移动加权平均法，是指各数值的加权系数随时间呈指数式递减，越靠近当前时刻的数值加权系数就越大
- 指数移动加权平均较传统的平均法来说，一是不需要保存过去所有的数值；二是计算量显著减小

其中 EWMA 公式如下：
$$
X_i=w*X_{i-1}+(1-w)*X_{cur}
$$
EWMA 算法中 β 的计算公式（[牛顿冷却定律](http://www.ruanyifeng.com/blog/2012/03/ranking_algorithm_newton_s_law_of_cooling.html)）也在这里一并给出，其中 decayTime 可根据自己的业务需求进行调整：

```
w := math.Exp(float64(-td) / float64(decayTime))
```

### 1.2 inflight

代表节点的当前正在处理的请求数，即反应了节点的拥塞程度。

### 1.3 success

节点请求的成功率，初始化为 1000，当成功率低于 500 的时候，当前节点就会被判定为不健康。success 同样是根据 EWMA 计算出来的。

## 2. go-zero 中对 P2C 算法的实现

我们可以看见在 p2c 包里面对 gRPC 的 subConn 进行了包装，每个连接都有自己的统计数据：

``` go
type subConn struct {
	addr     resolver.Address
	conn     balancer.SubConn
	lag      uint64	// 延迟
	inflight int64  // 节点当前正在处理的请求数
	success  uint64 // 请求成功率
	requests int64
	last     int64
	pick     int64
}
```

当要选择一个节点进行请求的时候就会调用 Pick 函数：

``` go
func (p *p2cPicker) Pick(ctx context.Context, info balancer.PickInfo) (
	conn balancer.SubConn, done func(balancer.DoneInfo), err error) {
	p.lock.Lock()
	defer p.lock.Unlock()

	var chosen *subConn
	switch len(p.conns) {
	case 0:
		return nil, nil, balancer.ErrNoSubConnAvailable
	case 1:
		chosen = p.choose(p.conns[0], nil)
	case 2:
		chosen = p.choose(p.conns[0], p.conns[1])
	default:
		var node1, node2 *subConn
    // 循环 pickTimes 次，如果还没有取到健康的节点，就取最后一次的结果
		for i := 0; i < pickTimes; i++ {
			a := p.r.Intn(len(p.conns))
			b := p.r.Intn(len(p.conns) - 1)
			if b >= a {
				b++
			}
			node1 = p.conns[a]
			node2 = p.conns[b]
			if node1.healthy() && node2.healthy() {
				break
			}
		}

		chosen = p.choose(node1, node2)
	}

	atomic.AddInt64(&chosen.inflight, 1)
	atomic.AddInt64(&chosen.requests, 1)
  // 回调函数
	return chosen.conn, p.buildDoneFunc(chosen), nil
}

// 判断节点是否健康
func (c *subConn) healthy() bool {
	return atomic.LoadUint64(&c.success) > throttleSuccess
}
```

这个函数实现了负载均衡的逻辑：

- 首先会随机选择出两个节点；
- 然后分别计算出两个节点的负载；
- 选择出负载最小的节点。

再来看看具体的 `choose` 方法：

``` go
func (p *p2cPicker) choose(c1, c2 *subConn) *subConn {
	start := int64(timex.Now())
	if c2 == nil {
		atomic.StoreInt64(&c1.pick, start)
		return c1
	}

	if c1.load() > c2.load() {
		c1, c2 = c2, c1
	}

	pick := atomic.LoadInt64(&c2.pick)
	if start-pick > forcePick && atomic.CompareAndSwapInt64(&c2.pick, pick, start) {
		return c2
	} else {
		atomic.StoreInt64(&c1.pick, start)
		return c1
	}
}

// 节点的负载情况
func (c *subConn) load() int64 {
	// plus one to avoid multiply zero
	lag := int64(math.Sqrt(float64(atomic.LoadUint64(&c.lag) + 1)))
	load := lag * (atomic.LoadInt64(&c.inflight) + 1)
	if load == 0 {
		return penalty
	} else {
		return load
	}
}
```

可以看见，choose 做的工作也很少，就是比较两个节点的负载情况，然后选择负载最低的那个节点。看到这里，不免心生疑问，前面提到的 lag，success 究竟是在哪里完成的计算呀。其实是在节点完成 rpc 调用之后，会调用一个回调函数：

``` go
func (p *p2cPicker) buildDoneFunc(c *subConn) func(info balancer.DoneInfo) {
	start := int64(timex.Now())
	return func(info balancer.DoneInfo) {
		atomic.AddInt64(&c.inflight, -1)
    // 当前时间
		now := timex.Now()
    // 上一次调用的时间
		last := atomic.SwapInt64(&c.last, int64(now))
    // 获取时间间隔
		td := int64(now) - last
		if td < 0 {
			td = 0
		}
    // 获取时间衰减系数
		w := math.Exp(float64(-td) / float64(decayTime))
    // 获取调用延时
		lag := int64(now) - start
		if lag < 0 {
			lag = 0
		}
		olag := atomic.LoadUint64(&c.lag)
		if olag == 0 {
			w = 0
		}
    // 保存本地计算出的延迟数据
		atomic.StoreUint64(&c.lag, uint64(float64(olag)*w+float64(lag)*(1-w)))
    // success 的计算逻辑同上
		success := initSuccess
		if info.Err != nil && !codes.Acceptable(info.Err) {
			success = 0
		}
		osucc := atomic.LoadUint64(&c.success)
		atomic.StoreUint64(&c.success, uint64(float64(osucc)*w+float64(success)*(1-w)))

		stamp := p.stamp.Load()
		if now-stamp >= logInterval {
			if p.stamp.CompareAndSwap(stamp, now) {
				p.logStats()
			}
		}
	}
}
```



## 3. 总结

以上，便是对于 go-zero 中 P2C 算法的分析。



参考:

- [负载均衡-P2C算法](https://exceting.github.io/2020/08/13/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1-P2C%E7%AE%97%E6%B3%95/)

