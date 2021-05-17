---
title: "B"
date: 2021-04-21T08:58:20+08:00
draft: true
---

- 链表和数组相比, 有什么优劣？

  > 链表更加灵活， 可动态添加删除  大小可变  
  >
  > 链表更加消耗空间
  >
  > 数据可以利用下标快速访问元素

- TCP 和 UDP 有什么区别?

  - TCP有超时重传机制，如果发送方一段时间内没有收到 ACK，就知道**很可能**是数据包丢失了，紧接着就重发该数据包，直到收到 ACK 为止。
  - TCP（Transmission Control Protocol，传输控制协议）是面向连接的协议，也就是说，在收发数据前，必须和对方建立可靠的连接。
  - UDP是面向报文的传输协议，不保证数据不会丢失, 只能尽全力交付， quic 建立在 udp 之上，也能保证可靠传输

- 描述一下 TCP 四次挥手的过程?

  > ![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B.png)

- [TCP 有哪些状态?](https://blog.mimvp.com/article/44678.html)

- TCP 的 LISTEN 状态是什么?

- TCP 的 CLOSE_WAIT 状态是什么?

  > close_wait 是服务器被动关闭产生的状态，如果服务器不重启，那么这条tcp 连接会一直存在，占用资源和套接字

- TCP 的 TIME_WAIT 状态是什么?

  > TIME_WAIT是因为客户端主动发起 close，在四次挥手之后还要保留 2 MSL 的时间，这个时间段内的状态就是 TIME_WAIT
  >
  > MSL（Maximum Segment Lifetime），TCP允许不同的实现可以设置不同的MSL值。**第一，保证客户端发送的最后一个ACK报文能够到达服务器，因为这个ACK报文可能丢失。**站在服务器的角度看来，我已经发送了FIN+ACK报文请求断开了，客户端还没有给我回应，应该是我发送的请求断开报文它没有收到，于是服务器又会重新发送一次，而客户端就能在这个2MSL时间段内收到这个重传的报文，接着给出回应报文，并且会重启2MSL计时器。如果客户端收到服务端的FIN+ACK报文后，发送一个ACK给服务端之后就“自私”地立马进入CLOSED状态，可能会导致服务端无法确认收到最后的ACK指令，也就无法进入CLOSED状态，这是客户端不负责任的表现。**第二，防止失效请求。**防止类似与“三次握手”中提到了的“已经失效的连接请求报文段”出现在本连接中。客户端发送完最后一个确认报文后，在这个2MSL时间中，就可以使本连接持续的时间内所产生的所有报文段都从网络中消失。这样新的连接中不会出现旧连接的请求报文。
  >
  >  在TIME_WAIT状态无法真正释放句柄资源，在此期间，Socket中使用的本地端口在默认情况下不能再被使用。该限制对于客户端机器来说是无所谓的，但对于高并发服务器来说，会极大地限制有效连接的创建数量，称为性能瓶颈。所以建议将高并发服务器TIME_WAIT超时时间调小。RFC793中规定MSL为2分钟。但是在当前的高速网络中，2分钟的等待时间会造成资源的极大浪费，在高并发服务器上通常会使用更小的值。

  作用：

  1. 保证客户端发送的最后一个 ACK 报文能够到达服务器，因为这个 ACK 报文可能丢失。
  2. 

  遇到大量 TIME_WAIT 怎么处理？

  1. 解决的方法是要么使用连接池，要么在同一个连接做尽可能多的事情，避免频繁开关.
  2. 缩短服务器 time_wait 的等待时间

- 建立一个 socket 连接要经过哪些步骤?

- 常见的 HTTP 状态码有哪些?

- 301和302有什么区别?

  1. 302重定向表示临时性转移(Temporarily Moved )，当一个网页URL需要短期变化时使用。
  2. 301重定向是永久的重定向，搜索引擎在抓取新内容的同时也将旧的网址替换为重定向之后的网址。

- 504和500有什么区别?

  1. 500 Internal Server Error 内部服务错误：顾名思义500错误一般是服务器遇到意外情况，而无法完成请求。
  2. 504 Bad Gateway timeout 网关超时

- 403是什么？

  1. 服务器收到请求，但是拒绝提供服务，也就是没有访问权限。

- HTTPS 和 HTTP 有什么区别?

- 写一个算法题: 手写快排?

- B-树和B+树区别，数据库索引原理，组合索引怎么使用？最左匹配的原理

- linux 常用指令

  1. uptime：可以快速查看机器的负载情况，在 Linux 系统中，这些数据**表示等待 CPU 资源的进程和阻塞在不可中断 IO 进程（进程状态为 D）的数量**。这些数据可以让我们对系统资源使用有一个宏观的了解。

     ``` sh
     [root@f-135 ~]# uptime
      15:31:54 up 128 days,  3:29,  3 users,  load average: 6.81, 4.59, 2.46
     ```

     

  2. **dmesg | tail**：输入系统日志，示例中的输出，可以看见一次内核的 oom kill 和一次 TCP 丢包。这些日志可以帮助排查性能问题。

     ``` sh
     $ dmesg | tail
     [1880957.563150] perl invoked oom-killer: gfp_mask=0x280da, order=0, oom_score_adj=0
     [...]
     [1880957.563400] Out of memory: Kill process 18694 (perl) score 246 or sacrifice child
     [1880957.563408] Killed process 18694 (perl) total-vm:1972392kB, anon-rss:1953348kB, file-rss:0kB
     [2320864.954447] TCP: Possible SYN flooding on port 7001. Dropping request.  Check SNMP counters.
     
     ```

  3. **vmstat 1**: vmstat(8) 命令，每行会输出一些系统核心指标，这些指标可以让我们更详细的了解系统状态。后面跟的参数 1，表示每秒输出一次统计信息，表头提示了每一列的含义，这几介绍一些和性能调优相关的列：

     - r：等待在 CPU 资源的进程数。这个数据比平均负载更加能够体现 CPU 负载情况，数据中不包含等待 IO 的进程。如果这个数值大于机器 CPU 核数，那么机器的 CPU 资源已经饱和。
     - free：系统可用内存数（以千字节为单位），如果剩余内存不足，也会导致系统性能问题。下文介绍到的 free 命令，可以更详细的了解系统内存的使用情况。
     - si, so：交换区写入和读取的数量。如果这个数据不为 0，说明系统已经在使用交换区（swap），机器物理内存已经不足。
     - us, sy, id, wa, st：这些都代表了 CPU 时间的消耗，它们分别表示用户时间（user）、系统（内核）时间（sys）、空闲时间（idle）、IO 等待时间（wait）和被偷走的时间（stolen，一般被其他虚拟机消耗）。

  4. **mpstat -P ALL 1**：查看每个 cpu 的负载情况

     ``` sh
     Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
     Average:     all   34.54    0.00    5.10    0.01    0.00    1.33    0.00    0.00    0.00   59.02
     Average:       0   46.29    0.00    7.62    0.03    0.00    7.41    0.00    0.00    0.00   38.64
     Average:       1   28.88    0.00    3.68    0.00    0.00    0.27    0.00    0.00    0.00   67.17
     Average:       2   48.79    0.00    5.97    0.00    0.00    2.63    0.00    0.00    0.00   42.62
     Average:       3   26.33    0.00    3.54    0.00    0.00    0.10    0.00    0.00    0.00   70.02
     Average:       4   47.53    0.00    7.29    0.00    0.00    5.67    0.00    0.00    0.00   39.52
     Average:       5   27.87    0.00    3.55    0.03    0.00    0.03    0.00    0.00    0.00   68.52
     Average:       6   48.05    0.00    7.30    0.00    0.00    5.31    0.00    0.00    0.00   39.34
     Average:       7   29.49    0.00    3.69    0.00    0.00    0.03    0.00    0.00    0.00   66.79
     Average:       8   35.63    0.00    6.28    0.00    0.00    0.00    0.00    0.00    0.00   58.08
     Average:       9   29.68    0.00    3.55    0.00    0.00    0.00    0.00    0.00    0.00   66.77
     Average:      10   36.54    0.00    6.02    0.00    0.00    0.03    0.00    0.00    0.00   57.40
     Average:      11   27.91    0.00    3.58    0.00    0.00    0.00    0.00    0.00    0.00   68.51
     Average:      12   34.82    0.00    5.90    0.00    0.00    0.00    0.00    0.00    0.00   59.28
     Average:      13   25.28    0.00    4.06    0.07    0.00    0.00    0.00    0.00    0.00   70.60
     Average:      14   33.45    0.00    6.10    0.00    0.00    0.00    0.00    0.00    0.00   60.46
     Average:      15   26.81    0.00    3.66    0.00    0.00    0.03    0.00    0.00    0.00   69.49
     ```

     

  5. **iostat -xz 1**：iostat 命令主要用于查看机器磁盘 IO 情况。该命令输出的列，主要含义是：

     - r/s, w/s, rkB/s, wkB/s：分别表示每秒读写次数和每秒读写数据量（千字节）。读写量过大，可能会引起性能问题。

     - await：**IO 操作的平均等待时间，单位是毫秒**。这是应用程序在和磁盘交互时，需要消耗的时间，包括 IO 等待和实际操作的耗时。如果这个数值过大，可能是硬件设备遇到了瓶颈或者出现故障。

     - avgqu-sz：向设备发出的请求平均数量。如果这个数值大于 1，可能是硬件设备已经饱和（部分前端硬件设备支持并行写入）。

     - %util：设备利用率。这个数值表示设备的繁忙程度，经验值是如果超过 60，可能会影响 IO 性能（可以参照 IO 操作平均等待时间）。如果到达 100%，说明硬件设备已经饱和。

     - [iowait](http://linuxperf.com/?p=33)： **表示在一个采样周期内有百分之几的时间属于以下情况：CPU空闲、并且有仍未完成的I/O请求。**

       > Percentage of time that the CPU or CPUs were idle during which the system had an outstanding disk I/O request.

       对 %iowait 常见的误解有两个：一是误以为 %iowait 表示CPU不能工作的时间，二是误以为 %iowait 表示I/O有瓶颈。

       1. iowait 的首要条件就是 CPU 空闲，才会有 iowait，既然空闲当然就可以接受运行任务，只是因为没有可运行的进程，CPU才进入空闲状态的。那为什么没有可运行的进程呢？因为进程都处于休眠状态、在等待某个特定事件：比如等待定时器、或者来自网络的数据、或者键盘输入、或者等待I/O操作完成，等等。

       2. iowait 不一定就是 io 有瓶颈，

       这就是为什么说 %iowait 所含的信息量非常少的原因，它是一个非常模糊的指标，如果看到 %iowait 升高，还需检查I/O量有没有明显增加，avserv/avwait/avque等指标有没有明显增大，应用有没有感觉变慢，如果都没有，就没什么好担心的。

     如果显示的是逻辑设备的数据，那么设备利用率不代表后端实际的硬件设备已经饱和。值得注意的是，即使 IO 性能不理想，也不一定意味这应用程序性能会不好，可以利用诸如预读取、写缓存等策略提升应用性能。

     ``` sh
     avg-cpu:  %user   %nice %system %iowait  %steal   %idle
                7.83    0.00   25.33   66.84    0.00    0.00
     
     Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
     vda               0.50     7.50  133.00  209.50 13846.00 94554.00   632.99   117.76  583.47  347.52  733.27   2.74  93.95
     vdb               0.00     2.50  486.50    1.50 33824.00    16.00   138.69     4.50    9.28    9.31    0.33   0.21  10.20
     ```

  6. **free –m** 查看内存使用情况

  7. **sar -n DEV 1**：查看网络情况，sar 命令在这里可以查看网络设备的吞吐率。在排查性能问题时，可以通过网络设备的吞吐量，判断网络设备是否已经饱和。如示例输出中，eth0 网卡设备，吞吐率大概在 22 Mbytes/s，既 176 Mbits/sec，没有达到 1Gbit/sec 的硬件上限。

     ``` sh
     
     $ sar -n DEV 1
     Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015     _x86_64_    (32 CPU)
     12:16:48 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
     12:16:49 AM      eth0  18763.00   5032.00  20686.42    478.30      0.00      0.00      0.00      0.00
     12:16:49 AM        lo     14.00     14.00      1.36      1.36      0.00      0.00      0.00      0.00
     12:16:49 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
     12:16:49 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
     12:16:50 AM      eth0  19763.00   5101.00  21999.10    482.56      0.00      0.00      0.00      0.00
     12:16:50 AM        lo     20.00     20.00      3.25      3.25      0.00      0.00      0.00      0.00
     12:16:50 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
     ^C
     ```

     

- 如何计算 qps

- https://zhuanlan.zhihu.com/p/117333338

- 数组和切片的区别

  > 1. 数组是固定长度的，切片可以动态扩容
  > 2. 底层数据结构不一样
  > 3. 数组和字符串的一些简单越界错误都会在编译期间发现，例如：直接使用整数或者常量访问数组；但是如果使用变量去访问数组或者字符串时，编译器就无法提前发现错误，我们需要 Go 语言运行时阻止不合法的访问
  > 4. 数组传参会 copy 整个数组一次

- slice 的扩容操作

  > 1. 如果容量在1024以下，那么直接 double
  > 2. 如果容量在1024之上，每次把容量增加1/4倍，直到达到 cap 的需求，再进行扩容

- map

- chan

- goroutine

- schedule

- select：在通常情况下，`select` 语句会阻塞当前 Goroutine 并等待多个 Channel 中的一个达到可以收发的状态。select 可以分四种情况来处理：

  1. 没有case：select 调用 `block()` 会一直阻塞

  2. `select` 只存在一个 `case`：阻塞在这个 case 上面，直到 chan 中有数据

     > 编译期会把 select 语句优化成 if 语句
     >
     > ```go
     > // 改写前
     > select {
     > case v, ok <-ch: // case ch <- v
     >     ...    
     > }
     > 
     > // 改写后
     > if ch == nil {
     >     block()
     > }
     > v, ok := <-ch // case ch <- v
     > ...
     > ```

  3. `select` 存在两个 `case`，其中一个 `case` 是 `default`；

  4. `select` 存在多个 `case`；

- panic

  panic 可以被多次调用，他们之间用 link 进行连接

- 线上问题一般怎么排查，比如oom

  1. 

- 介绍GMP模型、说说go并发GMP版本迭代的过程？说说如何避免全局队列饥饿？

- https://www.bilibili.com/read/cv11005564

- CPU 爆表怎么排查

  1. 用 top 命令查看 cpu 使用率最高的进程
     1. top -Hp pid查看进程下的各线程pid
  2. [使用 pprof 查看热点函数](https://studygolang.com/articles/21841)
     1. go tool pprof im_gate cpu.prof
     2. top 查看 cpu 占用前十的函数和
     3. 针对性的查找代码排查问题
  3. [实战](https://www.infoq.cn/article/f69uvzjuomq276hbp1qb)

- io 爆表怎么查

  1. Iostat -X 1 来查看哪块磁盘的 iowait 比较高

     ``` sh
     Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
     vda               0.00     1.00    2.00  238.00   156.00 108592.00   906.23   126.20  904.16  227.00  909.85   4.11  98.60
     vdb               0.00     0.00   19.00    0.00    80.00     0.00     8.42     0.02    1.00    1.00    0.00   0.16   0.30
     
     avg-cpu:  %user   %nice %system %iowait  %steal   %idle
                9.57    0.00   21.81   68.62    0.00    0.00
     ```

  2. 用 `iotop -Po` 找到频繁 io 的进程

  3. 

- **慢查询如何处理**

  > 磁盘 IO 和预读：磁盘 IO 慢，操作系统会利用局部性原理，预加载一些页到内存中。

  建索引的原则：

  1. 最左前缀匹配原则，mysql 会一直向右匹配，直到遇到范围查询(>、<、between、like)就停止匹配
  2. =和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式。
  3. 尽量选择区分度高的列作为索引，区分度的公式是count(distinct col)/count(*)，表示字段不重复的比例
  4. 索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’)。
  5. 尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。

- [流量控制与拥塞控制](https://zhuanlan.zhihu.com/p/37379780)

  > 流量控制：通过滑动窗口（接收窗口）来控制对端的发送速率，TCP为它的应用程序提供了**流量控制服务(flow-control service)**以消除发送方使接收方缓存溢出的可能性。流量控制因此是一个**速度匹配服务**，即发送方的发送速率与接收方应用程序的读取速率相匹配。发送方把自己已发送但未被确认的消息控制住rwnd的值范围内，这样就可以避免接收方缓存溢出。
  >
  > 拥塞控制：在 TCP 协议中，分组丢失一般是当网络变得拥塞时由路由器缓存溢出引起的。与流量控制一样，在发送方的TCP拥塞控制机制中跟踪了一个额外的变量，即**拥塞窗口**(congestion window)。拥塞窗口表示为cwnd,通过这个拥塞窗口，我们就能够对发送方向其连接发送数据的速率进行限制。具体的措施是:**让一个发送方中未确认的数据量不会超过cwnd和rwnd的最小值**。

  拥塞控制：拥塞控制是作用于网络的，它是防止过多的数据注入到网络中，避免出现网络负载过大的情况；常用的方法就是：（ 1 ）慢开始、拥塞避免（ 2 ）快重传、快恢复。

  流量控制：流量控制是作用于接收者的，它是控制发送者的发送速度从而使接收者来得及接收，防止分组丢失的。

  拥塞控制算法：

  1. **慢启动**：发送方维持一个叫做拥塞窗口cwnd（congestion window）的状态变量。拥塞窗口的大小取决于网络的拥塞程度，并且动态地在变化。发送方让自己的发送窗口等于拥塞窗口，另外考虑到接受方的接收能力，发送窗口可能小于拥塞窗口。

     为了防止cwnd增长过大引起网络拥塞，还需设置一个慢开始门限ssthresh状态变量。ssthresh的用法如下：当cwnd<ssthresh时，使用慢开始算法。
     当cwnd>ssthresh时，改用拥塞避免算法。
     当cwnd=ssthresh时，慢开始与拥塞避免算法任意

  2. **拥塞避免**：拥塞避免算法让拥塞窗口缓慢增长，即每经过一个往返时间RTT就把发送方的拥塞窗口cwnd加1，而不是加倍。这样拥塞窗口按线性规律缓慢增长。

  3. 快重传：快重传要求接收方在收到一个失序的报文段后就立即发出重复确认（为的是使发送方及早知道有报文段没有到达对方，可提高网络吞吐量约20%）而不要等到自己发送数据时捎带确认。快重传算法规定，发送方只要一连收到三个重复确认就应当立即重传对方尚未收到的报文段，而不必继续等待设置的重传计时器时间到期。

  4. 快恢复：快重传配合使用的还有快恢复算法，有以下两个要点：

     当发送方连续收到三个重复确认时，就执行“乘法减小”算法，把ssthresh门限减半（为了预防网络发生拥塞）。但是接下去并不执行慢开始算法
     考虑到如果网络出现拥塞的话就不会收到好几个重复的确认，所以发送方现在认为网络可能没有出现拥塞。所以此时不执行慢开始算法，而是将cwnd设置为ssthresh减半后的值，然后执行拥塞避免算法，使cwnd缓慢增大。如下图：TCP Reno版本是目前使用最广泛的版本。

- golang 中 CAS 是如何实现的？

  首先，我们使用 atomic 包中的 cas 函数的时候，这些函数只是一个声明，真正的实现还是在包 `src/runtime/internal/atomic/asm_amd64.s` 中。例如 doc.go 中的`CompareAndSwapInt64` 就对应`runtime/internal/atomic/Cas64`，具体对应关系如下：

  ![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/cas%E6%B1%87%E7%BC%96%E5%87%BD%E6%95%B0.png)

  ``` asm
  TEXT runtime∕internal∕atomic·Cas64(SB), NOSPLIT, $0-25
  	MOVQ	ptr+0(FP), BX
  	MOVQ	old+8(FP), AX
  	MOVQ	new+16(FP), CX
  	LOCK
  	CMPXCHGQ	CX, 0(BX)
  	SETEQ	ret+24(FP)
  	RET
  ```

  可以看见，asm 汇编代码使用了 x86 的 lock(一个命令前缀，在这里用于CMPXCHGQ)可以锁住总线保证多次内存操作的原子性。然后执行CMPXCHGQ

  > 1. 1. 拿AX(old) 与 BX(共享数据ptr) 做对比。
  >    2. 相等，则修改BX的值(共享数据ptr)为 CX，状态码ZX设置为 1 。
  >    3. 不相等，状态码ZX设置为 0

- Mutex 是如何实现的？

  1. mutex 中的 lock 回首先进行 cas 原子替换，如果 cas 成功，那么加锁成功，否则会降级到`lockSlow()`

  2. 进入 slow 之后也会进行 4 次 cas 自旋操作，避免 goroutine 切换带来的开销

     ``` asm
     TEXT runtime·procyield(SB),NOSPLIT,$0-0
     	MOVL	cycles+0(FP), AX
     again:
     	PAUSE //调用 pause
     	SUBL	$1, AX
     	JNZ	again
     	RET
     ```

  3. 尝试 cas 指令获取锁，如果超过 1ms，那么会进入饥饿状态，通过信号量获取锁

- 如何防止消费者重复消费？

  1. 可以发消息的时候带一个全局唯一的 id，处理消息之后把唯一 id 放入 redis 中，并设置过期时间
  2. 利用数据库唯一键保证幂等性

- [Rocketmq 如何保证消息不丢失？](https://www.cnblogs.com/goodAndyxublog/p/12563813.html)

  若要严格保证消息不丢失：

  ``` ini
  ## master 节点配置
  flushDiskType = SYNC_FLUSH
  brokerRole=SYNC_MASTER
  
  ## slave 节点配置
  brokerRole=slave
  flushDiskType = SYNC_FLUSH
  ```

- mysql 翻页如何优化？

  1. 对于只有一个条件的查询，就直接使用唯一索引，用 id 去 limit 

  2. 利用覆盖索引，先获取id，这一步就是索引覆盖扫描，不需要回表，然后通过id跟原表进行关联，改写后的SQL如下：

     ``` sql
     select * from trade_info a ,
     
     (select  id from trade_info where status = 0 and create_time >= '2020-10-01 00:00:00' and create_time <= '2020-10-07 23:59:59' order by id desc limit 102120, 20) as b   //这一步走的是索引覆盖扫描，不需要回表
      where a.id = b.id;
     ```

     

- select 和 epoll 的区别

  > select 的时间复杂度是 O(N)，并且每次都要拷贝传入的fd，有最大连接数的限制
  >
  > poll 的时间复杂度是 O(N)，然后查询每个fd对应的设备状态， **但是它没有最大连接数的限制**，原因是它是基于链表来存储的.
  >
  > epoll 的时间复杂度是 O(1)

  **select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的**，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。 

  **表面上看epoll的性能最好，但是在连接数少并且连接都十分活跃的情况下，select和poll的性能可能比epoll好，毕竟epoll的通知机制需要很多函数回调。**

- 瞬时写入量很大可能会打挂存储,怎么保护?

  先写内存或者消息队列，然后再写入存储

- 饥饿模式下如何保证队列里的协程一定拿得到锁，或者说协程是根据什么判断进入这个队列的？

  饥饿模式下, 锁资源直接从解锁的 goroutine 递交给队列中的第一个等待的 goroutine.

  新来的 goroutine 不会去尝试获取锁, 而是直接入队

- Channel 底层如何实现阻塞的？

  1. lock
  2. 看看 recv 队列有没有阻塞的 sudog，有就直接 handoff
     1. 如果是无缓冲 chan，阻塞等待一个接收者
     2. 如果是缓冲 chan，直接放入 buffer 中，如果 buffer 满了，阻塞

  - 同步 Channel — 不需要缓冲区，发送方会直接将数据交给（Handoff）接收方；
  - 异步 Channel — 基于环形缓存的传统生产者消费者模型；

  ```go
  type hchan struct {
  	qcount   uint
  	dataqsiz uint
  	buf      unsafe.Pointer
  	elemsize uint16
  	closed   uint32
  	elemtype *_type
  	sendx    uint
  	recvx    uint
  	recvq    waitq
  	sendq    waitq
  
  	lock mutex
  }
  ```

  在发送数据的逻辑执行之前会先为当前 Channel 加锁，防止多个线程并发修改数据。如果 Channel 已经关闭，那么向该 Channel 发送数据时会报 “send on closed channel” 错误并中止程序。

  1. 向 nil 的 chan 中发送数据，会导致阻塞
  2. 向 close 的 chan 中发送数据，会 panic

- zrange 的时间复杂度？

  O(log(N)+M)

- etcd 如何实现分布式锁？

  1. 利用 etcd 的 concurrency 包实现
  2. newsession, newmutex, lock
  3. 成功获取所的节点就可以去跑定时任务
  4. 其它没有获取到锁的 task job 就阻塞住

  > 1. 假设分布式锁的 Name 为 `/root/lockname`，用来控制某个共享资源，`concurrency` 会自动将其转换为目录形式：`/root/lockname/`
  > 2. 客户端 A 连接 Etcd，创建一个租约 Leaseid_A，并设置 TTL（以业务逻辑来定 TTL 的时间）, 以 `/root/lockname` 为前缀创建全局唯一的 Key，该 Key 的组织形式为 `/root/lockname/{leaseid_A}`，客户端 A 将此 Key 绑定租约写入 Etcd，同时调用 TXN 事务查询写入的情况和具有相同前缀 `/root/lockname/` 的 `Revision` 的排序情况
  > 3. 客户端 A 判断自己是否获得锁，以前缀 `/root/lockname/` 读取 keyValue 列表（keyValue 中带有 Key 对应的 `Revision`），判断自己 Key 的 `Revision` 是否为当前列表中最小的，如果是则认为获得锁；否则阻塞监听列表中前一个 `Revision` 比自己小的 Key 的删除事件，一旦监听到删除事件或者因租约失效而删除的事件，则自己获得锁
  > 4. 执行业务逻辑，操作共享资源
  > 5. 释放分布式锁，现网的程序逻辑需要实现在正常和异常条件下的释放锁的策略，如捕获 `SIGTERM` 后执行 `Unlock`，或者异常退出时，有完善的监控和及时删除 Etcd 中的 Key 的异步机制，避免出现死锁现象
  > 6. 当客户端持有锁期间，其它客户端只能等待，为了避免等待期间租约失效，客户端需创建一个定时任务进行续约续期。如果持有锁期间客户端崩溃，心跳停止，Key 将因租约到期而被删除，从而锁释放，避免死锁

- syn flood攻击如何处理？

  最基本的DoS攻击就是利用合理的服务请求来占用过多的服务资源，从而使合法用户无法得到服务的响应。syn flood属于Dos攻击的一种。

  如果恶意的向某个服务器端口发送大量的SYN包，则可以使服务器打开大量的半开连接，分配**TCB（Transmission Control Block）**, 从而消耗大量的服务器资源，同时也使得正常的连接请求无法被相应。当开放了一个TCP端口后，该端口就处于Listening状态，不停地监视发到该端口的Syn报文，一 旦接收到Client发来的Syn报文，就需要为该请求分配一个TCB，通常一个TCB至少需要280个字节，在某些操作系统中TCB甚至需要1300个字节，并返回一个SYN ACK命令，立即转为**SYN-RECEIVED即半开连接状态**。系统会为此耗尽资源。

  1. 监视系统的半开连接和不活动连接，当达到一定阈值时拆除这些连接，从而释放系统资源。这种方法对于所有的连接一视同仁，而且由于SYN Flood造成的半开连接数量很大，正常连接请求也被淹没在其中被这种方式误释放掉，因此这种方法属于入门级的SYN Flood方法。
  2. 消耗服务器资源主要是因为当SYN数据报文一到达，系统立即分配TCB，从而占用了资源。而SYN Flood由于很难建立起正常连接，因此，当正常连接建立起来后再分配TCB则可以有效地减轻服务器资源的消耗。常见的方法是使用Syn Cache和Syn Cookie技术。
  3. Sync Cache: 系统在收到一个SYN报文时，在一个专用HASH表中保存这种半连接信息，直到收到正确的回应ACK报文再分配TCB。这个开销远小于TCB的开销。当然还需要保存序列号。
  4. Syn Cookie技术则完全不使用任何存储资源，这种方法比较巧妙，它使用一种特殊的算法生成Sequence Number，这种算法考虑到了对方的IP、端口、己方IP、端口的固定信息，以及对方无法知道而己方比较固定的一些信息，如MSS(Maximum Segment Size，最大报文段大小，指的是TCP报文的最大数据报长度，其中不包括TCP首部长度。)、时间等，在收到对方 的ACK报文后，重新计算一遍，看其是否与对方回应报文中的（Sequence Number-1）相同，从而决定是否分配TCB资源。

- 如何获取 redis 里面所有的 key? 能用 `keys *` 吗？http://jinguoxing.github.io/redis/2018/09/04/redis-scan/

  一定不能用 `keys *`。

  1. 用 scan 来扫描， 直到没有数据返回。
  2. 把 redis 的内存数据 dump 下来，解析协议重放，然后就能获得当前 dump 的所有 key。

- 服务如何平滑重启？

  1. 滚动发布，下线服务的时候，先发送信号，应用捕捉到这个信号，然后在注册中心注销自己，不再提供服务，等待几秒，这个时候其他节点还在继续提供服务，然后下线
  2. 机器起来之后，注册自己，jenkins 睡眠 n 秒，

- 全局二级索引？

- mysql mvcc？

  对于 RC(READ COMMITTED) 和 RR(REPEATABLE READ) 隔离级别的实现就是通过上面的版本控制来完成。两种隔离界别下的核心处理逻辑就是判断所有版本中哪个版本是当前事务可见的处理。针对这个问题InnoDB在设计上增加了ReadView的设计，ReadView中主要包含当前系统中还有哪些活跃的读写事务，把它们的事务id放到一个列表中，我们把这个列表命名为为m_ids。

  对于查询时的版本链数据是否看见的判断逻辑：

  如果被访问版本的 trx_id 属性值小于 m_ids 列表中最小的事务id，表明生成该版本的事务在生成 ReadView 前已经提交，所以该版本可以被当前事务访问。

  如果被访问版本的 trx_id 属性值大于 m_ids 列表中最大的事务id，表明生成该版本的事务在生成 ReadView 后才生成，所以该版本不可以被当前事务访问。

  如果被访问版本的 trx_id 属性值在 m_ids 列表中最大的事务id和最小事务id之间，那就需要判断一下 trx_id 属性值是不是在 m_ids 列表中，如果在，说明创建 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建 ReadView 时生成该版本的事务已经被提交，该版本可以被访问。

- Redis 的跳表实现原理？

  为了在链表里二分查找设计出来的

- 分布式事务是什么？

- [gRPC负载均衡的基本原理？](https://pandaychen.github.io/2019/07/11/GRPC-SERVICE-DISCOVERY/)

- nginx 有哪些负载均衡算法？

  1. RoundRobin（轮询）
  2. Weight-RoundRobin（加权轮询）
  3. 源地址哈希法(ip hash)

- Zset 和 hash 在底层元素较小的时候使用的是什么数据结构？

  1. ziplist

     因为 ziplist 都是紧凑存储，没有冗余空间 (对比一下 Redis 的字符串结构)。意味着插入一个新的元素就需要调用 realloc 扩展内存。取决于内存分配器算法和当前的 ziplist 内存大小，realloc 可能会重新分配新的内存空间，并将之前的内容一次性拷贝到新的地址，也可能在原有的地址上进行扩展，这时就不需要进行旧内容的内存拷贝。

     如果 ziplist 占据内存太大，重新分配内存和拷贝内存就会有很大的消耗。所以 ziplist 不适合存储大型字符串，存储的元素也不宜过多。
  
- [海量数据下的一些常见问题？](https://wizardforcel.gitbooks.io/the-art-of-programming-by-july/content/06.02.html)

  1. 给定一个文件，包含1亿个ip，每个ip一行，可能是无效ip，要求统计出现次数最多的ip地址，只有1GB内存?
     1. 因为要统计重复出现的 ip，所以可以先对这个文件进行 hash 切割，每个文件占用的内存的大小在 1GB 一下
     2. 然后再用 hashmap 来统计，分别求出每个文件中出现频率最高的 ip
     3. 再排序取出出现频率最高的 ip 地址

- redis 大 key ， 大 value

  1. [redis bigkey 如何优化?](https://molin.cool/2020/06/10/%E8%AE%B0%E4%B8%80%E6%AC%A1Redis-Big-Key%E4%BC%98%E5%8C%96/)
  2. [Redis的大value有什么危害？](https://www.cnblogs.com/cxy2020/p/13748658.html)

  > 为什么要优化？
  >
  > 1. bigkey 会引起节点之间内存使用不均匀，间接影响到节点之间的负载不均
  > 2. 由于 redis 是单线程模型，会阻塞其他客户端的请求，导致延时增大
  > 3. key由于比value需要做更多的操作如hashcode、链表中比较等操作，所以会比value更多一些内存相关开销。以及 cpu 方面的开销
  >
  > 大 value
  >
  > 1. 内存不均：单value较大时，可能会导致节点之间的内存使用不均匀，间接地影响key的部分和负载不均匀；
  > 2. **阻塞请求：redis为单线程，单value较大读写需要较长的处理时间，会阻塞后续的请求处理；**
  > 3. 阻塞网络：单value较大时会占用服务器网卡较多带宽，可能会影响该服务器上的其他Redis实例或者应用。

- [如何解决跨域问题？](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)

  在响应头中，把 `Access-Control-Allow-Origin` 改成 * ，允许全部域名跨域，可以指定特点域名，逗号分隔。

  规范要求，对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 以外的 HTTP 请求，或者搭配某些 MIME 类型的 [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 请求），浏览器必须首先使用 [`OPTIONS`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/OPTIONS) 方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨源请求。

  - 某些请求不会触发 [CORS 预检请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS#preflighted_requests)。本文称这样的请求为“简单请求”，请注意，该术语并不属于 [Fetch](https://fetch.spec.whatwg.org/) （其中定义了 CORS）规范。若请求满足所有下述条件，则该请求可视为“简单请求”：
    - 使用下列方法之一：
      - [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET)
      - [`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD)
      - [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST)

  

- mysql 意向锁

  InnoDB 支持`多粒度锁`，特定场景下，行级锁可以与表级锁共存。

  意向锁之间互不排斥，但除了 IS 与 S 兼容外，`意向锁会与 共享锁 / 排他锁 互斥`。

  IX，IS是表级锁，不会和行级的X，S锁发生冲突。只会和表级的X，S发生冲突。

  意向锁在保证并发性的前提下，实现了`行锁和表锁共存`且`满足事务隔离性`的要求。

- [mysql next-key 锁？](https://www.cnblogs.com/zhoujinyi/p/3435982.html)

  数据库使用锁是为了支持更好的并发，提供数据的完整性和一致性。InnoDB是一个支持行锁的存储引擎，锁的类型有：共享锁（S）、排他锁（X）、意向共享（IS）、意向排他（IX）。为了提供更好的并发，InnoDB提供了非锁定读：不需要等待访问行上的锁释放，读取行的一个快照。该方法是通过InnoDB的一个特性：MVCC来实现的。

  1，Record Lock：单个行记录上的锁。

  2，Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况。

  3，Next-Key Lock：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。

  ​	在可重复读隔离级别下，InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围（GAP）。上面索引值有1，3，5，8，11，其记录的GAP的区间如下：是一个**左开右闭**的空间（原因是默认主键的有序自增的特性，结合后面的例子说明）

- fork 子进程，套接字的引用计数会+1 (https://blog.csdn.net/gatieme/article/details/50615112)

  ![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/fork%E8%BF%9B%E7%A8%8B%E5%90%8E%E5%A5%97%E6%8E%A5%E5%AD%97%E5%8E%9F%E7%90%86.png)

- [万级TPS亿级流水-中台账户系统架构设计](https://www.cnblogs.com/wangiqngpei557/p/13169705.html)

- 一条 sql 执行很慢，怎么回事？
  
  1. 系统正在刷脏页，占用系统I/O，会导致读操作耗时长
  
- kafka 为何这么快？
  
  1. 直接 append 数据，利用磁盘的顺序 io
  
- [守护进程是啥？](https://www.zhihu.com/question/38609004)

  1. Fork 出一个进程
  2. 退出父进程
  3. 当前进程 setsid
  4. 二次 fork，确保没有控制终端， 可以通过使进程不再成为会话组长来禁止进程重新打开控制终
  5. 改变当前目录为根目录
  6. 把标准输入，输出，错误都重定向到 /dev/null

- tidb 数据如何存储？

  1. TIDB 会为每个表分配一个表 id，tableid，表 id 在集群范围内唯一，整数

  2. tidb 为每一行分配一个行 id，rowid，行 id 在表内唯一，整数，如果是整数主键，那么就是这个主键作为 rowid

  3. 每行数据根据如下规则进行编码：

     ``` bash
     Key: tablePrefix{TableId}_recordPrefixSep{RowId}
     Value: [col1,col2...]
     ```

     其中 `tablePrefix` 和 `recordPrefixSep` 都是特定的字符串常量，用于在 Key 空间内区分其他数据。其具体值在后面的小结中给出。

- tidb 中索引的存储方式

  1. TiDB 同时支持主键和二级索引（包括唯一索引和非唯一索引）。与表数据映射方案类似，TiDB 为表中每个索引分配了一个索引 ID，用 `IndexID` 表示。

  2. 对于主键和唯一索引，我们需要根据键值快速定位到对应的 RowID，因此，按照如下规则编码成 (Key, Value) 键值对：

     ```  bash
     Key:   tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue
     Value: RowID
     ```

  3. 对于不满足唯一性约束的普通二级索引，一个键可能对应多个行，我们需要根据键值范围查询对应的 RowID。 因此，按照如下规则编码成 (Key, Value) 键值对：

     ```bash
     Key:   tablePrefix{TableID}_indexPrefixSep{IndexID}_indexedColumnsValue_{RowID}
     Value: null
     ```

- namespace 技术是实现容器的关键？

  1. UTS：不同的 namespace 可以配置不同的 hostname
  2. USER，不同的 namespace 可以配置不同的用户和组
  3. Mount，不同的 namespace 的文件系统挂载点是隔离的
  4. PID，不同的 name 有完全独立的 pid
  5. NETWORK，不同的 namespace 有独立的网络协议栈

  有 clone，setns，unshare 系统调用

  应用场景：

  1. K8S 容器化之后应用里几乎没有什么可用的调试工具，可以利用容器 NAmespace 共享的思路，启动一个包含各种调试工具的容器，加入到 pod 的 pid、net 等 namespace 中，实现都任意 pod 的 debug 功能

- 301和 302 的区别？

  1. 301表示永久重定向，表示客户端以后都应该访问这个新的地址，搜索引擎也会抓取新网址的内容
  2. 302 临时重定向，旧地址A的资源仍可访问，这个重定向只是临时从旧地址A跳转到B地址，这时搜索引擎会抓取B网址内容，但是会将网址保存为A的。

- 500和 504 的区别？

  1. 服务器遇到了一个未曾预料的状况，导致了它无法完成对请求的处理。一般来说，这个问题都会在服务器的程序码出错时出现。
  2. 作为网关或者代理工作的服务器尝试执行请求时，未能及时从上游服务器（URI标识出的服务器，例如HTTP、FTP、LDAP）或者辅助服务器（例如DNS）收到响应。　　注意：某些代理服务器在DNS查询超时时会返回400或者500错误

- restful 接口幂等性

  1. HTTP GET 方法，用于获取资源，不管调用多少次接口，结果都不会改变，所以是幂等的。只是查询数据，不会影响到资源的变化，因此我们认为它幂等。
  2. HTTP POST 方法是一个非幂等方法，因为调用多次，都将产生新的资源。因为它会对资源本身产生影响，每次调用都会有新的资源产生，因此不满足幂等性。
  3. HTTP PUT 方法，因为它直接把实体部分的数据替换到服务器的资源，我们多次调用它，只会产生一次影响，但是有相同结果的 HTTP 方法，所以满足幂等性。
  4. HTTP DELETE 方法用于删除资源，会将资源删除。调用一次和多次对资源产生影响是相同的，所以也满足幂等性。

- 为什么是 TCP/IP 协议？

  1. TCP/IP（Transmission Control Protocol/Internet Protocol，传输控制协议/网际协议）是指能够在多个不同网络间实现信息传输的协议簇。TCP/IP协议不仅仅指的是[TCP](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/TCP/33012) 和[IP](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/IP/224599)两个协议，而是指一个由[FTP](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/FTP/13839)、[SMTP](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/SMTP/175887)、TCP、[UDP](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/UDP/571511)、IP等协议构成的协议簇， 只是因为在TCP/IP协议中TCP协议和IP协议最具代表性，所以被称为TCP/IP协议。

- Sync.Map 的实现原理？

  1. 有 read 和 dirty 这两个 map 来实现了读写分离，并且中间用 entry 存储 value 的指针，抽象出了三个状态
  2. 当 p == expunged 的时候说明 dirty != nil， 并且数据已被删除
  3. 当 p == nil 时说明 dirty == nil ，并且数据已经被删除
  4. `sync.map` 是线程安全的，读取，插入，删除也都保持着常数级的时间复杂度。
  5. 通过读写分离，降低锁时间，来提高效率，适用于读多写少的场景
  6. 新写入的 key 会保存到 dirty 中，如果这时 dirty 为 nil，就会先新创建一个 dirty，并将 read 中未被删除的元素拷贝到 dirty。
  7. 当 dirty 为 nil 的时候，read 就代表 map 所有的数据；当 dirty 不为 nil 的时候，dirty 才代表 map 所有的数据。
  
- [如何保证缓存与数据库的一致性？](https://xie.infoq.cn/article/47241d099404a1565e168fad4)

  1. 不更新缓存，而是删除缓存
  2. 订阅 binlog，用 task job 来重做缓存

- 两台机器网络不通，如何排查？

  1. 查网卡是否正常工作
  2. 查看机器是否与网关相连
  3. 如果无法 ping 通，可能是网关限制了 ICMP 数据包
  4. traceroute 来查看当前 icmp 协议包的经过的全部路由
  5. 排查远程主机是否开放了端口
  6. 查看本机是否监听端口， netstat -lnp