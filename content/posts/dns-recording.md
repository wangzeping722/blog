---
title: "DNS学习总结"
date: 2019-12-23T10:36:05+08:00
draft: false
---

DNS 有多种记录类型，每种记录都有不同的作用，这篇文章主要总结了常用的记录。

#### A

A（Address）记录用来指定主机名（域名）对应的 IPv4 地址记录。比如，当你浏览一个网页时候会，浏览器就会先去查找对应域名的 A 记录，来获取 IP 地址，获得了 IP 地址，才能与服务器建立连接。

>  我们可以为同一个域名添加多个 A 记录，解析的时候，会得到多个 IP，会随机选择一个使用。

*域名不区分大小写。记录的名称应该是由 ASCII 码字母、数字和 - 组成。*



#### AAAA

AAAA 记录用来指定主机名（域名）对应的 IPv6 地址记录，其他与 A 相同。



#### CNAME （canonical names）

CNAME（规范名称），也就是别名记录，它能够让我们把多个名字映射到同一个主机。

应用场景：

- 使用 CDN 服务时。
- 假如一个服务器运行着 100 个网站，这 100 个网站 CNAME 到 `a.example.com`。当该服务器 IP 改变时，你只需改变 `a.example.com`的 IP 就可以了。

注意：由于`CNAME`记录就是一个替换，所以域名一旦设置`CNAME`记录以后，就不能再设置其他记录了（比如`A`记录和`MX`记录），这是为了防止产生冲突。举例来说，`foo.com`指向`bar.com`，而两个域名各有自己的`MX`记录，如果两者不一致，就会产生问题。由于顶级域名通常要设置`MX`记录，所以一般不允许用户对顶级域名设置`CNAME`记录。

A记录是把域名解析到IP地址，而CNAME记录是把域名解析到另外一个域名，而这个域名最终会指向A记录，在功能实现在上A记录与CNAME记录没有区别。

例如：

``` dns
a.example.com       IN      A       192.168.1.101
b.example.com       IN      CNAME   a.example.com（规范名称）
```

你在请求 b.example.com 的时候，他会先返回给你一个 CNAME ，然后你在用 CNAME 去查询。

#### NS

NS（Name Server），域名服务器记录，返回保存下一级域名信息的服务器地址。该记录只能设置为域名，不能设置为IP地址。

一般来说，为了服务的安全可靠，至少应该有两条`NS`记录，而`A`记录和`MX`记录也可以有多条，这样就提供了服务的冗余性，防止出现单点失败。

``` shell
$ dig NS google.com
google.com.		777	IN	NS	ns4.google.com.
google.com.		777	IN	NS	ns3.google.com.
google.com.		777	IN	NS	ns2.google.com.
google.com.		777	IN	NS	ns1.google.com.
```

我们可以看见，google 的权威 NS 服务器有四个。



#### MX

MX（MX record），邮件交换记录，用于邮件服务器的地址。MX 记录允许设置优先级，当多个邮件服务器可用时，会根据该值决定投递邮件的服务器（值小的优先），下面的`20, 10, 40 ...`就表示优先级。

``` shell
$ dig MX google.com
google.com.		306	IN	MX	20 alt1.aspmx.l.google.com.
google.com.		306	IN	MX	10 aspmx.l.google.com.
google.com.		306	IN	MX	40 alt3.aspmx.l.google.com.
google.com.		306	IN	MX	30 alt2.aspmx.l.google.com.
google.com.		306	IN	MX	50 alt4.aspmx.l.google.com.
```



#### SOA

SOA 叫做起始授权机构记录，NS用于标识多台域名解析服务器，SOA记录用于在众多NS记录中的**主服务器**。SOA 记录表示此域名的权威解析服务器地址。 当要查询的域名在所有递归解析服务器中没有域名解析的缓存时，就会回源来请求此域名的SOA记录，也叫权威解析记录。

没有SOA记录的 zone 不符合 RFC 1035 要求的标准。

``` 
$TTL 86400
@   IN  SOA     startech60serve root.startech60serve.com. (
        2018110201  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
        IN  NS      startech60serve
        IN  A       192.168.1.3
        IN  MX 10   startech60serve
startech60serve     IN  A       192.168.1.3
```

`root.startech60serve.com. `其中第一个点表示是@

#### TXT

TXT（TXT record），文本记录，一般用来描述一个域名，或者用来做某种验证功能，例如：

``` sh
$ dig TXT google.com
google.com.		300	IN	TXT	"docusign=05958488-4752-4ef2-95eb-aa7ba8a3bd0e"
google.com.		3600	IN	TXT	"facebook-domain-verification=22rm551cu4k0ab0bxsw536tlds4h95"
google.com.		300	IN	TXT	"docusign=1b0a6754-49b1-4db5-8540-d2c12664b289"
google.com.		3600	IN	TXT	"globalsign-smime-dv=CDYX+XFHUw2wml6/Gb8+59BsH31KzUr6c1l2BPvqKX8="
google.com.		3600	IN	TXT	"v=spf1 include:_spf.google.com ~all"
```



#### PTR

PTR 记录是 A 记录的逆向记录，又称做 IP 反查记录或指针记录，负责将 IP 反向解析为域名，即反向域名解析。



参考：

[https://skyao.io/learning-dns/dns/](https://skyao.io/learning-dns/dns/)

[https://deepzz.com/post/dns-recording-type.html](https://deepzz.com/post/dns-recording-type.html)