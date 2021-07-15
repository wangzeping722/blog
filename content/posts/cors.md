---
title: "浏览器跨域的解决方案"
date: 2021-07-15T10:07:06+08:00
draft: false
---

前几天在工作中遇到了跨域问题，在博客里面简单记录一下。首先我们要知道为什么会产生跨域？

浏览器处于安全考虑，会限制脚本内发起的跨源HTTP请求。 例如，XMLHttpRequest和Fetch API遵循`同源策略`。 这意味着使用这些API的Web应用程序只能从加载应用程序的同一个域请求HTTP资源，除非响应报文包含了正确CORS响应头。

跨域资源共享（CORS）是一种基于HTTP头的机制，这种机制允许允许服务器标示除了它自己以外的其它origin（域，协议和端口），这样浏览器可以访问加载这些资源。跨源资源共享还通过一种机制来检查服务器是否会允许要发送的真实请求，该机制通过浏览器发起一个到服务器托管的跨源资源的"预检"（Option）请求。在预检中，浏览器发送的头中标示有HTTP方法和真实请求中会用到的头。

在CORS中，请求可以分为简单请求和非简单请求。

简单请求需要包含以下两大条件：

（1) 请求方法是以下三种方法之一：

- HEAD
- GET
- POST

（2）HTTP的头信息不超出以下几种字段：

- Accept
- Accept-Language
- Content-Language
- Last-Event-ID
- Content-Type：只限于三个值`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`

这是为了兼容表单（form），因为历史上表单一直可以发出跨域请求。AJAX 的跨域设计就是，只要表单可以发，AJAX 就可以直接发。

凡是不同时满足上面两个条件，就属于非简单请求。

浏览器对这两种请求的处理，是不一样的。

对于两种请求，浏览器有不一样的处理策略。

### 简单请求

浏览器会直接发起跨域请求，并且在http头部加上 Origin 字段：

```go
GET /cors HTTP/1.1
Origin: http://api.bob.com
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

如上的例子，Origin用来说明，本次请求来自哪个源（协议+域名+端口），服务器根据这个值来判断是否允许这次跨域请求。

如果`Origin`指定的源，不在许可范围内，服务器会返回一个正常的HTTP回应。浏览器发现，这个回应的头信息没有包含`Access-Control-Allow-Origin`字段（详见下文），就知道出错了，从而抛出一个错误，被`XMLHttpRequest`的`onerror`回调函数捕获。注意，这种错误无法通过状态码识别，因为HTTP回应的状态码有可能是200。

如果`Origin`指定的域名在许可范围内，服务器返回的响应，会多出几个头信息字段。

> Access-Control-Allow-Origin: http://api.bob.comAccess-Control-Allow-Credentials: true
> Access-Control-Expose-Headers: FooBar
> Content-Type: text/html; charset=utf-8

上面的头信息之中，有三个与CORS请求相关的字段，都以`Access-Control-`开头。

**（1）Access-Control-Allow-Origin**

该字段是必须的。它的值要么是请求时`Origin`字段的值，要么是一个`*`，表示接受任意域名的请求。

### 非简单请求

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json。这时服务器会有一个预检请求：

```
OPTIONS /cors HTTP/1.1
Origin:http://api.bob.comAccess-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

原理跟上面简单请求一样，要是返回的 `Access-Control-Allow-Origin` 允许服务器继续发起请求，那么跨域请求就能够成功。

### 后端代码

落实到后端代码，我们只需要在服务器请求到来的时候判断http header 中的 `origin` 和是否符合我们的要求。在gin框架中，实现一个简陋的cors中间件：

```go
func Cors() gin.HandlerFunc {
    return func(c *gin.Context) {
        method := c.Request.Method
        if origin != "" {
            c.Header("Access-Control-Allow-Origin", "*") 
            c.Header("Access-Control-Allow-Methods", "POST")
            c.Header("Access-Control-Allow-Headers", "*")
            c.Header("Access-Control-Expose-Headers", "Content-Length, Access-Control-Allow-Origin, Access-Control-Allow-Headers, Cache-Control, Content-Language, Content-Type")
            c.Header("Access-Control-Allow-Credentials", "true")
        }

        if method == "OPTIONS" {
            c.AbortWithStatus(http.StatusNoContent)
        }

        c.Next()
    }
}
```

参考：

[跨域资源共享 CORS 详解](https://www.ruanyifeng.com/blog/2016/04/cors.html)

