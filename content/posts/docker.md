---
title: "Docker"
date: 2021-04-20T18:07:00+08:00
draft: true
---

本文介绍 Docker 使用到的核心技术，以便对 Docker 的 有深入的理解。

## NameSpaces

Linux 的命名空间机制提供了一下七种不同的命名空间，包括`CLONE_NEWCGROUP`、`CLONE_NEWIPC`、`CLONE_NEWNET`、`CLONE_NEWNS`、`CLONE_NEWPID`、`CLONE_NEWUSER`、`CLONE_NEWPUTS`，通过这七个选项我们能在创建新的进程时设置新进程应该在*哪些资源*上与宿主机进行隔离。

