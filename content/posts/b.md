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

  - TCP（Transmission Control Protocol，传输控制协议）是面向连接的协议，也就是说，在收发数据前，必须和对方建立可靠的连接。
  - UDP是面向报文的传输协议，不保证数据不会丢失, 只能尽全力交付， quic 建立在 udp 之上，也能保证可靠传输

- 描述一下 TCP 四次挥手的过程?

  > ![](https://blog-wero.oss-cn-shanghai.aliyuncs.com/img/%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B.png)

- TCP 有哪些状态?

- TCP 的 LISTEN 状态是什么?

- TCP 的 CLOSE_WAIT 状态是什么?

  > close_wait 是服务器被动关闭产生的状态，如果服务器不重启，那么这条tcp 连接会一直存在，占用资源和套接字

- TCP 的 TIME_WAIT 状态是什么?

  > TIME_WAIT是因为客户端主动发起 close，在四次挥手之后还要保留 2 MSL 的时间，这个时间段内的状态就是 TIME_WAIT

- 建立一个 socket 连接要经过哪些步骤?

- 常见的 HTTP 状态码有哪些?

- 301和302有什么区别?

- 504和500有什么区别?

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

- select

- 线上问题一般怎么排查，比如oom

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

  

