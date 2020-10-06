---
title: "Kratos 框架学习——启动流程分析"
date: 2020-10-05T11:42:50+08:00
draft: false
---

Kratos 是 b 站开源的微服务框架，包含大量微服务相关框架及工具，同时整套Kratos框架也是不错的学习仓库，这个系列的文章将会对 Kratos 的源码进行学习和分析。

如何安装和使用 kratos 框架就不在本文详细描述，请参考：[go-kratos](https://go-kratos.github.io/kratos/#/quickstart)



## 剖析 demo 启动流程

通过 wiki 中的指示，我们生成了一个 **kratos-demo** 的项目，目录结构如下：

``` 
├── CHANGELOG.md 
├── OWNERS
├── README.md
├── api                     # api目录为对外保留的proto文件及生成的pb.go文件
│   ├── api.bm.go
│   ├── api.pb.go           # 通过go generate生成的pb.go文件
│   ├── api.proto
│   └── client.go
├── cmd
│   └── main.go             # cmd目录为main所在
├── configs                 # configs为配置文件目录
│   ├── application.toml    # 应用的自定义配置文件，可能是一些业务开关如：useABtest = true
│   ├── db.toml             # db相关配置
│   ├── grpc.toml           # grpc相关配置
│   ├── http.toml           # http相关配置
│   ├── memcache.toml       # memcache相关配置
│   └── redis.toml          # redis相关配置
├── go.mod
├── go.sum
└── internal                # internal为项目内部包，包括以下目录：
│   ├── dao                 # dao层，用于数据库、cache、MQ、依赖某业务grpc|http等资源访问
│   │   ├── dao.bts.go
│   │   ├── dao.go
│   │   ├── db.go
│   │   ├── mc.cache.go
│   │   ├── mc.go
│   │   └── redis.go
│   ├── di                  # 依赖注入层 采用wire静态分析依赖
│   │   ├── app.go
│   │   ├── wire.go         # wire 声明
│   │   └── wire_gen.go     # go generate 生成的代码
│   ├── model               # model层，用于声明业务结构体
│   │   └── model.go
│   ├── server              # server层，用于初始化grpc和http server
│   │   ├── grpc            # grpc层，用于初始化grpc server和定义method
│   │   │   └── server.go
│   │   └── http            # http层，用于初始化http server和声明handler
│   │       └── server.go
│   └── service             # service层，用于业务逻辑处理，且为方便http和grpc共用方法，建议入参和出参保持grpc风格，且使用pb文件生成代码
│       └── service.go
└── test                    # 测试资源层 用于存放测试相关资源数据 如docker-compose配置 数据库初始化语句等
    └── docker-compose.yaml
```

不难看出，`cmd/main.go` 便是程序的入口，我们来看看 `cmd/main.go` 的代码：

``` go
func main() {
	flag.Parse()
	log.Init(nil) // debug flag: log.dir={path}
	defer log.Close()
	log.Info("kratos-demo start")
	paladin.Init() // 配置文件初始化
	_, closeFunc, err := di.InitApp()	// 初始化 App(),可以看成是一个微服务的实体对象, 利用编译期依赖注入,来初始化 app 中的各个组件
	if err != nil {
		panic(err)
	}
	c := make(chan os.Signal, 1)	// 实现优雅的退出
	signal.Notify(c, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
	for {
		s := <-c
		log.Info("get a signal %s", s.String())
		switch s {
		case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
			closeFunc()
			log.Info("kratos-demo exit")
			time.Sleep(time.Second)
			return
		case syscall.SIGHUP:
		default:
			return
		}
	}
}
```

整个流程是非常的清楚：

1. 初始化日志；
2. 初始化配置；
3. 初始化 App（调用编译期依赖注入生成的方法 NewApp），这一步之后就能够对外提供服务了；
4. 最后实现了优雅的关闭和重启。



## 编译期依赖注入

Kratos 框架使用 [wire](https://github.com/google/wire) 进行编译期依赖注入，使得各个模块之间保持低耦合。如何使用 wire，本文不做说明，下面直接看 demo 是如何使用这个技术的。

`di.InitApp()` 就是利用编译期注入生成的函数，而这个函数的目的就是组合初始化 `App` 所需要的一切资源，初始化并返回 `App` 对象。

首先我们可以看见在 `internal/dao/dao.go` 和 `internal/service/service.go` 文件中各看见一个 Provider：

```go
var Provider = wire.NewSet(New, NewDB, NewRedis, NewMC)
var Provider = wire.NewSet(New, wire.Bind(new(pb.DemoServer), new(*Service)))
```

在 `internal/di/wire.go` 我们又看见了下面这段代码：

``` go
//go:generate kratos t wire
func InitApp() (*App, func(), error) {
	panic(wire.Build(dao.Provider, service.Provider, http.New, grpc.New, NewApp))
}
```

注意，我们看见了会使用`go:generate kratos t wire`这段生成命令来生成代码

``` go
func InitApp() (*App, func(), error) {
	redis, cleanup, err := dao.NewRedis()
	if err != nil {
		return nil, nil, err
	}
	memcache, cleanup2, err := dao.NewMC()
	if err != nil {
		cleanup()
		return nil, nil, err
	}
	db, cleanup3, err := dao.NewDB()
	if err != nil {
		cleanup2()
		cleanup()
		return nil, nil, err
	}
	daoDao, cleanup4, err := dao.New(redis, memcache, db)
	if err != nil {
		cleanup3()
		cleanup2()
		cleanup()
		return nil, nil, err
	}
	serviceService, cleanup5, err := service.New(daoDao)
	if err != nil {
		cleanup4()
		cleanup3()
		cleanup2()
		cleanup()
		return nil, nil, err
	}
	engine, err := http.New(serviceService)
	if err != nil {
		cleanup5()
		cleanup4()
		cleanup3()
		cleanup2()
		cleanup()
		return nil, nil, err
	}
	server, err := grpc.New(serviceService)
	if err != nil {
		cleanup5()
		cleanup4()
		cleanup3()
		cleanup2()
		cleanup()
		return nil, nil, err
	}
	app, cleanup6, err := NewApp(serviceService, engine, server)
	if err != nil {
		cleanup5()
		cleanup4()
		cleanup3()
		cleanup2()
		cleanup()
		return nil, nil, err
	}
	return app, func() {
		cleanup6()
		cleanup5()
		cleanup4()
		cleanup3()
		cleanup2()
		cleanup()
	}, nil
}
```

把上面的逻辑串起来就是：

1. dao.NewRedis()
2. dao.NewMC()
3. dao.NewDB()
4. dao.New(redis, memcache, db)
5. service.New(daoDao)
6. http.New(serviceService)
7. grpc.New(serviceService)
8. NewApp(serviceService, engine, server)

至此，依赖注入完成。

其实整个微服务的启动流程也是非常简单的，接下来的文章中我就重点分析比较感兴趣的点。