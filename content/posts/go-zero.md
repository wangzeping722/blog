---
title: "go-zero 源码分析计划"
date: 2020-11-06T13:03:49+08:00
draft: false
---

[go-zero](https://github.com/tal-tech/go-zero) 是一个集成了各种工程实践的 web 和 rpc 框架。通过弹性设计保障了大并发服务端的稳定性，经受了充分的实战检验。

go-zero 包含极简的 API 定义和生成工具 goctl，可以根据定义的 api 文件一键生成 Go, iOS, Android, Kotlin, Dart, TypeScript, JavaScript 代码，并可直接运行。

使用 go-zero 的好处：

- 轻松获得支撑千万日活服务的稳定性
- 内建级联超时控制、限流、自适应熔断、自适应降载等微服务治理能力，无需配置和额外代码
- 微服务治理中间件可无缝集成到其它现有框架使用
- 极简的 API 描述，一键生成各端代码
- 自动校验客户端请求参数合法性
- 大量微服务治理和并发工具包



## 计划

go-zero 是一个集大成者，但又不失优雅的微服务框架，我计划在业余时间研究它的源码。我把感兴趣的模块都写在了下面的 TODO 列表：

- Balancer--基于 p2c 负载均衡
- Bloom 布隆过滤器的实现
- go-zero 的熔断器
- go-zero 的服务注册与发现

