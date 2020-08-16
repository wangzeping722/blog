---
title: "TiDB学习课程 - week1"
date: 2020-08-16T11:58:45+08:00
draft: false
---

最近非常想学习关于分布式系统的知识，碰巧 TiDB 官方推出了《高性能 TiDB 系列课程》，我就像捡到宝一样高兴😄呀。希望能够通过这次的课程有所收获，我一定会坚持学完的。

## 题目描述

本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：

* 1 TiDB 
* 1 PD
*  3 TiKV  

改写后：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的 日志 。



## 源码编译

我是在本地直接编译运行的，过程有些繁琐，当然也可以写一个 Dockerfile 来自动化部署啦。

这三个项目的编译都非常简单，先把项目分别拉下来，然后分别进入每个项目下执行 make 就可以编译成功。编译后二进制的路径如下：

``` bash
pd/bin/
tidb/bin/
tikv/target/debug/
```



## 部署

参考 [TiKV](https://github.com/tikv/tikv/blob/master/docs/how-to/deploy/using-binary.md) 集群的部署方案就可以把 `PD` 和 `TiKV` 这两个组件部署成功，然后再单独部署一个 `TiDB` 节点就完成了作业的部署要求：

1. 启动 PD

   ``` sh
   ./pd-server --name=pd1 \
       --data-dir=pd1 \
       --client-urls="http://127.0.0.1:2379" \
       --peer-urls="http://127.0.0.1:2380" \
       --initial-cluster="pd1=http://127.0.0.1:2380" \
       --log-file=pd1.log
   ```

2. 启动 TiKV 集群：在部署 TiKV 的时候遇到一个小问题【单进程的最大文件描述符不满足 TiKV 的需求】，所以需要改一下系统配置（MacOS）

   ``` sh
   #修改文件描述符大小
   sudo launchctl limit maxfiles 65536 200000
   
   ./tikv-server --pd-endpoints="127.0.0.1:2379" \
       --addr="127.0.0.1:20160" \
       --data-dir=tikv1 \
       --log-file=tikv1.log
   
   ./tikv-server --pd-endpoints="127.0.0.1:2379" \
       --addr="127.0.0.1:20161" \
       --data-dir=tikv2 \
       --log-file=tikv2.log
   
   ./tikv-server --pd-endpoints="127.0.0.1:2379" \
       --addr="127.0.0.1:20162" \
       --data-dir=tikv3 \
       --log-file=tikv3.log
   ```

   

3. 启动 TiDB

   ``` sh
   ./tidb-server --store=tikv --path='127.0.0.1:2379' --log-file=tidb.log
   ```

部署完之后，我们查看 tidb.log 这个日志，出现 `server is running MySQL protocol` 后，就是部署完成啦。



### 修改源码

TiDB 的源码对新手来说算是非常友好的了，可以从`tidb-server`中的 main 函数开始一路追踪到连接建立和处理客户端请求的代码，简单易懂。下面给出流程，具体的以后找个时间分析分析：



直到遇到了`session.go`中的这段代码，加上日志就算大功告成啦。

``` go
// InitTxnWithStartTS create a transaction with startTS.
func (s *session) InitTxnWithStartTS(startTS uint64) error {
	if s.txn.Valid() {
		return nil
	}

	// no need to get txn from txnFutureCh since txn should init with startTs
  logutil.BgLogger().Info("hello transaction")
	txn, err := s.store.BeginWithStartTS(startTS)
	if err != nil {
		return err
	}
	txn.SetVars(s.sessionVars.KVVars)
	s.txn.changeInvalidToValid(txn)
	err = s.loadCommonGlobalVariablesIfNeeded()
	if err != nil {
		return err
	}
	return nil
}
```

重新编译运行 TiDB，你还会发现一个有意思的现象，那就是程序会狂打 `hello transaction`，这是因为 TiDB 后台会有一些定时任务在不断地开始事务执行一些操作。

到这里，week1 的的作用就算大功告成啦。希望自己能坚持下来💪~~~