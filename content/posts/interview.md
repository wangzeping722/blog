---
title: "Interview"
date: 2021-03-19T10:28:11+08:00
draft: true
---

## go 相关问题

1. 为什么需要 P 这个组件，直接把 runqueues 放到 M 不行吗？

   > You might wonder now, why have contexts at all? Can't we just put the runqueues on the threads and get rid of contexts? Not really. The reason we have contexts is so that we can hand them off to other threads if the running thread needs to block for some reason.
   >
   > An example of when we need to block, is when we call into a syscall. Since a thread cannot both be executing code and be blocked on a syscall, we need to hand off the context so it can keep scheduling.

   ### 原理

   scheduler是runtime的一部分，与go程序一起运行，OS scheduler是抢占式调度，而go scheduler是协作式调度

   协作式调度是用户来设置操作，而go的调度是由go runtime来控制，并不是由用户控制，所以我们也依然可以认为go scheduler是抢占式

   

   而go scheduler同上面的os scheduler一样也有三种状态

   waiting：等待状态(例如mutexes, atomic)

   runnable：就绪状态，只给M就可以执行 

   executing：运行状态，goroutine在M 上执行指令

   ![image.png](https://cdn.nlark.com/yuque/0/2021/png/5362565/1614399797422-785803a8-7f05-4b55-8203-09d6d3a00566.png?x-oss-process=image%2Fresize%2Cw_1500)

   

   #### 时机

   严谨地说，什么时候可能会发生调度

   1. go关键，此时新起goroutine，肯定会调度
   2. GC，执行GC时，肯定需要在M上运行，肯定会发生调度，remember GC只会回收堆上的内存
   3. 系统调用，此时goroutine发生系统调用，肯定会阻塞M，所以他会被调度走，调度上来一个新的goroutine 
   4. 内存同步访问，atomic/mutex/channel会发生goroutine阻塞，此时会发生调度，将阻塞的goroutine调度走，新的goroutine会调度上来，但是当满足条件后，依然会被调度起来

   

   #### 工作

   go程序启动时，会给每个逻辑核心数分配一个P(处理器核总数*2)，同时会给每个P分配一个M(内核线程)，这些内核线程是由Os scheduler调度

   sche的工作是什么呢

   其实就是找到可运行的goroutines，合理地分配到P上让他运行

   看下面的解释：运行可执行的goroutine直到阻塞，然后会从本地运行队列中找，本地没有则会去找全局队列，当全局没有的话，则回去steal其他P上的goroutine

   好处：干活儿快～

   ```
   runtime.schedule() {
       // only 1/61 of the time, check the global runnable queue for a G.
       // if not found, check the local queue.
       // if not found,
       //     try to steal from other Ps.
       //     if not, check the global runnable queue.
       //     if not found, poll network.
   }
   ```

   

   ![image.png](https://cdn.nlark.com/yuque/0/2021/png/5362565/1614403147879-56e48d80-ec9b-4adb-848f-bcdde37a67d9.png)

   

2. gpm到底是什么

3. scheduler是如何调度的

4. 什么时候会触发调度

5. 当在M上运行的goroutine发生阻塞时，会怎么工作

   当goroutine进行系统调用时，对于M来说根据调用的类型，有同步和异步两种执行方式

   同步：当有goroutine发生系统调用时，此时由于是同步操作，所以M发生阻塞，这个时候P是会重新找可运行的M的，新的M会接手P上的LRQ(本地队列)，继续运行runnable的goroutines，之前的M会被雪藏，需要时会被放出(什么时候是需要的？ //TODO)，goroutine在执行完系统调用后，依然会回到P的LRQ

   异步：当有goroutine发生系统调用时，此时由于是异步操作，goroutine会被network poller接手，M会继续执行LRQ其他的runnable的goroutines，goroutine在执行完系统调用后，依然会回到P的LRQ

   

   #### 10毫秒抢占标识位

   由于go的sche不是抢占式调度(时间片)，是协作式调度，go sche会随系统启动同时后台同时启动一个sys，他会检测全局运行超过10ms的goroutine，如果超过了10ms，会设置该goroutine的抢占标识位，之后sche则会处理这种goroutine，将他们放到全局P的goroutine，以示惩罚，就跟小妾似的，占用老公时间太久了～但是如果没有函数调用的时候，他是无法处理到的

   

   学的差不多了，我们来个例子

   ```
   func main(){
       var x int
       //得到系统逻辑核心数，eg: threads = 8
       //系统建立8个M绑定8个P
       threads := runtime.GOMAXPROCS(0)
   
       //启动8个goroutine, 同时还有一个main goroutine
       for i := 0; i < threads; i++ {
           go func() {
               for{ x++ }
           }()
       }
       //此时由于sleep, sche会将资源调度给还没得到运行的runnable goroutine
       //所以8个M正好对应的8个N，然而每个goroutine中是个无限死循环，出不来，则会一直卡死
       //答案知道了么
       time.Sleep(time.Second)
       fmt.Println("x =", x)
   }
   //解决方法，在for{ x++ }前sleep，或者i < threads - 1， 此时会有一个单独的M只运行main goroutine
   //在休眠后sche会唤醒goroutine，M继续执行
   ```

   

   #### goroutine

   goroutine可以说是thread的一层抽象，写代码时我们面对的不是thread，而是goroutine，操作系统看到的却是thread

   区别：从三个角度(内存消耗，创建与销毁，切换)

   内存消耗：每创建一个goroutine则会消耗2KB大小栈内存，当实际运行时如果栈空间不足，则会发生扩容，创建一个thread则会消耗1MB栈内存

   创建与销毁：thread的创建与销毁是与操作系统打交道，是内核级别，消耗是巨大的，而goroutine是用户级别，由go runtime管理，创建与销毁的代价是非常小的

   切换：thread切换大概需要1000-1500纳秒，而goroutine则需要200纳秒

   

   M：N

   runtime在启动的时候，会创建M个系统线程，之后创建的N个goroutine都会依附在这些M个线程上执行

   在同一时刻，一个线程上只能跑一个goroutine，当发生阻塞时，会把当前goroutine调度走，执行其他的goroutine

   同一时刻，一个P只能运行一个M

   

   ### 源码结构

   请先知道gpm的结构参数

   ```
   type schedt struct {
       
       midle        muintptr // 由空闲的工作线程组成的链表
       nmidle       int32 // 空闲的工作线程数量   
       nmidlelocked int32 // 空闲的且被 lock 的 m 计数
       mcount       int32  // 已经创建的工作线程数量 
       maxmcount    int32 // 表示最多所能创建的工作线程数量
           
       ngsys uint32 // goroutine 的数量，自动更新
       // 全局可运行的 G队列
       runqhead guintptr // 队列头
       runqtail guintptr // 队列尾
       runqsize int32 // 元素数量
       // Global cache of dead G's.dead G 的全局缓存   
       // 已退出的 goroutine 对象，缓存下来,避免每次创建 goroutine 时都重新分配内存
       gflock       mutex
       gfreeStack   *g
       gfreeNoStack *g
      
       ngfree       int32 // 空闲 g 的数量 
       
       pidle      puintptr  // 由空闲的 p 结构体对象组成的链表  
       npidle     uint32// 空闲的 p 结构体对象的数量
          
       // sudog 结构的集中缓存
       sudoglock  mutex
       sudogcache *sudog   
   }
   ```

   在程序运行时，全局只有一份sche实体，他维护了全局所有的调度器信息

   在runtime运行过程中，有几个比较重要的全局变量，程序初始化时。都会被初始化各自对应的零值

   ```
   allm   *m  //保存所有的 m
   
   allp  [_MaxGomaxprocs+ 1]*p  // 保存所有的 p，_MaxGomaxprocs = 1024
   
   gomaxprocs  int32  // p 的最大值，默认等于 ncpu
   
   ncpu        int32  // 程序启动时，会调用 osinit 函数获得此值
   
   sched       schedt // 调度器结构体对象，记录了调度器的工作状态
   
   m0           m   // 代表进程的主线程
   
   g0           g  // m0 的 g0，即 m0.g0 = &g0
   ```

   #### 程序初始化

   call osinit。初始化系统核心数。

   call schedinit。初始化调度器。

   make & queue new G。创建新的 goroutine。

   call runtime·mstart。调用 mstart，启动调度。

   The new G calls runtime·main。在新的 goroutine 上运行 runtime.main 函数

   

   #### 初始化g0

   g0栈为runtime代码提供了一个环境，会把g0的地址存入DI寄存器，此时主线程绑定了g0，g0的主要作用是为runtime提供运行时的栈，一个g0大约有64k，下图来自[阿波张的goroutine调度](https://mp.weixin.qq.com/s?__biz=MzU1OTg5NDkzOA==&mid=2247483769&idx=1&sn=3d77609a293d87e64639afc8d2219e1c&scene=19#wechat_redirect)

   ![image.png](https://cdn.nlark.com/yuque/0/2021/png/5362565/1615887059126-0b32989a-b558-4788-85ea-dc75b4c35e6a.png)

   #### 主线程绑定m0

   ![image.png](https://cdn.nlark.com/yuque/0/2021/png/5362565/1614412392947-28ecd79a-b481-4254-aadf-993bf474deea.png)

   保存在主线程本地存储中的值是 g0 的地址，也就是说工作线程的私有全局变量其实是一个指向 g 的指针而不是指向 m 的指针。目前这个指针指向g0，表示代码正运行在 g0 栈

   #### 初始化m0

   首先会调用getg获取当前正在运行的goroutine，得到是当前运行的goroutine的地址

   ```
   // 初始化 m
   func mcommoninit(mp *m) {
   
       // 初始化过程中_g_ = g0
       _g_ := getg()
   
       // g0 stack won't make sense for user (and is not necessary unwindable).
       if _g_ != _g_.m.g0 {
           callers(1, mp.createstack[:])
       }
   
       // random 初始化
       mp.fastrand = 0x49f6428a+ uint32(mp.id) + uint32(cputicks())
       if mp.fastrand == 0{
           mp.fastrand = 0x49f6428a
       }
   
       lock(&sched.lock)
   
       // 设置 m 的 id, m0的id递增
       mp.id = sched.mcount
       sched.mcount++
   
       // 检查已创建系统线程是否超过了数量限制（10000),程序会设置做多跑10000个工作线程
       checkmcount()
   
       // ……………省略了初始化 gsignal
   
       // Add to allm so garbage collector doesn't free g->m
       // when it is just in a register or thread-local storage.
       
       //将 m 挂到全局变量 allm 上，allm 是一个指向 m 的的指针
       mp.alllink = allm
       
       //形成链表
       atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))unlock(&sched.lock)
   }
   ```

   初始化后会将m0绑定到allm上，后续的m的创建如图

   ![image.png](https://cdn.nlark.com/yuque/0/2021/png/5362565/1614412634475-5f7b4133-10a2-49c4-a760-46a8733a8ba4.png)

   

   #### 

6. 为什么每个P都会对应一个g0，m0 (g0是用于调度每个线程中的goroutine，包括gc等等，拥有比较大的栈内存)

7. 什么时候会抢占P

   如果检查到P处于系统调用之中，则需要检查是否要抢占

   1. p 的运行队列里面有等待运行的 goroutine，可能原因是当前goroutine正在执行系统调用，其他的goroutine需要其他P来接管

   2. 没有无所事事的 p

   3. 从上一次监控线程观察到 p 对应的 m 处于系统调用之中到现在已经超过 10 毫秒

   如果P处于运行状态

   则检查M对应的一直运行的G是否超过了10毫秒

   

   ### g0与其他goroutine的切换

   当启动任何一个goroutine时，操作系统都会调用newproc函数

   g0负责调度，当有其他goroutine准备就绪后，会替换当前g0执行，curg存放的就是当前执行的goroutine

   

   ### main函数与其他goroutine的切换

   main goroutine 在执行完成后会执行exit(0)直接退出，对于普通goroutine，调用goexit(0), 会切换到g0，进行goroutine的一些数据，加入到goroutine缓存池，进入scheduler调度，至此完成

   

   ### sysmon都做了什么

   sysmon 中会进行 netpool（获取 fd 事件）、retake（抢占）、forcegc（按时间强制执行 gc），scavenge heap（释放自由列表中多余的项减少内存占用）等处理。

   

   

   ### newproc都做了什么

   ```
   //CALL  runtime·newproc(SB) # 创建goroutine
   //newproc有两个参数，第二个参数是要创建的函数的地址，第一个参数是第二个函数的参数大小
   
   func printNumber(a, b int){
       fmt.Println(a, b)
   }
   
   func main(){
       go printNumber(1,2)
   }
   
   //此时newproc的第一个参数就是16   runtime·newproc(16, printNumber)
   //在新建newg后会讲newg.buf.pc设置成函数的地址，被调度起来时，会把newg.buf.pc赋值给ip寄存器
   //设置g的状态为_Grunnable，代表可以运行
   
   //放入运行队列～本地，当本地满了放入全局队列 
   //save方法保存了正在运行的g的下一条将要执行的指令地址，以及栈顶地址
   func save(pc, sp uintptr) {
       _g_ := getg()
       _g_.sched.pc = pc //再次运行时的指令地址
       _g_.sched.sp = sp //再次运行时到栈顶
       _g_.sched.lr = 0
       _g_.sched.ret = 0
       _g_.sched.g = guintptr(unsafe.Pointer(_g_))
       // We need to ensure ctxt is zero, but can't have a write
       // barrier here. However, it should always already be zero.
       // Assert that.
       if _g_.sched.ctxt != nil {
           badctxt()
       }
   }
   ```

   #### 

8. 调度的本质

   > os scheduler有三种状态: 
   >
   > waiting：等待状态(例如mutexes, atomic)
   >
   > runnable：就绪状态，只给CPU资源就可以执行 
   >
   > executing：运行状态，线程在执行指令
   >
   > 线程一般做的两个事儿：IO型，计算型
   >
   > 线程切换：系统用一个runnable的线程讲一个executing的线程替换下来，状态也随之改变，当是计算型任务时，executing -> runnable， IO型则是executing -> waiting
   >
   > 对于os scheduler来说，最重要的不是让CPU闲着，而是让每个CPU核心都有任务可做

   调度的本质其实都是，修改寄存器得值来做到CPU切换系统进程/goroutine的调度

   

   go 程序执行是由program和runtime组成，用户进行的系统调用，都是由runtime来拦截，以此帮助他进行垃圾回收以及调度等工作，runtime维护了所有goroutine，并通过go scheduler调度，goroutine和thread是相互独立的，但是goroutine依赖于thread才能执行

   runtime起始时会启动一些g，比如垃圾回收的g，运行代码的g，执行调度的g，并且会创建一个M用来执行这些g

   

   ```
   #For scheduling goroutines onto kernel threads.
   #用于将goroutine调度到内核线程上
   ```

   ![image.png](https://cdn.nlark.com/yuque/0/2021/png/5362565/1614334133431-079f403e-fe25-4929-a7f5-eeb4bea70df9.png)

   

   #### 核心思想

   1. reuse threads
   2. 控制(正在运行的不包含阻塞)线程数为N的数量，N等于CPU核心数量
   3. 维护runqueues，并且可以stealing goroutine，程序阻塞后可以将runqueues传递给其他线程(M)

9. 多个线程与多个M如何一一对应？

   在每个线程创建的时候，利用TLS，为每个线程创建自己的私有全局变量M

10. 为什么要把工作线程与m对应

    每个工作线程对应一个m，m中保存了正在运行的栈顶，正在运行的goroutine以及p结构，因此m可以找到对应的g以及绑定的p.

    #### 初始化allp

    ```
    procs := ncpu //ncpu 这里已经被赋上了系统的核心数
    if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0{
        procs = n
    }
    //_MaxGomaxprocs = 1024
    if procs > _MaxGomaxprocs{
        procs = _MaxGomaxprocs
    }
    
    //procresize方法中的一部分截取
    func procresize(nprocs int32) *p {
        // 初始化所有的 P
        for i := int32(0); i < nprocs; i++ {
            pp := allp[i]
            if pp == nil {
                // 申请新对象
                pp = new(p)
                pp.id = i
                // pp 的初始状态为 stop
                pp.status = _Pgcstop
                pp.sudogcache = pp.sudogbuf[:0]
                for i := range pp.deferpool {
                    pp.deferpool[i] = pp.deferpoolbuf[i][:0]
                }
                // 将 pp 存放到 allp 处
                atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
            }
        }
    
        // 获取当前正在运行的 g 指针，初始化时 _g_ = g0
        _g_ := getg()
        if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
            // continue to use the current P
            // 继续使用当前 P
            _g_.m.p.ptr().status = _Prunning
        } else{
            // 初始化时执行这个分支
             _g_.m.p = 0
            _g_.m.mcache = nil
            // 取出第 0 号 p
            p := allp[0]
            p.m = 0
            p.status = _Pidle
            / 将 p0 和 m0 关联起来
            acquirep(p)
            if trace.enabled {
                traceGoStart()
            }
        }
        
        
        var runnablePs *p
        
        // 下面这个 for 循环把所有空闲的 p 放入空闲链表   
        for i := nprocs - 1; i >= 0; i-- {
            p := allp[i]    
            // allp[0] 跟 m0 关联了，不会进行之后的“放入空闲链表”   
            if _g_.m.p.ptr() == p {
                continue
            }
            // 状态转为 idle
            p.status = _Pidle
            // p 的 LRQ 里没有 G
            if runqempty(p) {
                // 放入全局空闲链表
                pidleput(p)
            } else{
                p.m.set(mget())
                p.link.set(runnablePs)
                runnablePs = p
            }    
        }
        //随机设置以后M来偷工作的时候，给一个随机的P
        stealOrder.reset(uint32(nprocs))
        //此时的runnablePs返回给调用者，其实就是runnable的所有p
        return runnablePs
    }   
    ```

    过程总结一下：

    使用 make([]p, nprocs) 初始化全局变量 allp，即 allp = make([]p, nprocs)

    循环创建并初始化 nprocs 个 p 结构体对象并依次保存在 allp 切片之中

    把 m0 和 allp[0] 绑定在一起，即 m0.p = allp[0]，allp[0].m = m0

    把除了 allp[0] 之外的所有 p 放入到全局变量 sched 的 pidle 空闲队列之中

    最后的gpm初始化后的联系

    ![image.png](https://cdn.nlark.com/yuque/0/2021/png/5362565/1614414467065-42c7b829-96e0-46fb-be1e-6cef9ff8093b.png)

    

    

    #### schedu function

    ```
    //never return 
    //先初始化g0,之后转换到main goroutine
    func schedule() {
        _g_ := getg()  //_g_ = 每个工作线程m对应的g0，初始化时是m0的g0
        //......
        var gp *g
      
        //......
        
        if gp == nil {
            //为了保证调度的公平性，每进行61次调度就需要优先从全局运行队列中获取goroutine，
            //因为如果只调度本地队列中的g，那么全局运行队列中的goroutine将得不到运行
            if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
                lock(&sched.lock) //所有工作线程都能访问全局运行队列，所以需要加锁
                gp = globrunqget(_g_.m.p.ptr(), 1) //从全局运行队列中获取1个goroutine
                unlock(&sched.lock)
            }
        }
        if gp == nil {
            //从与m关联的p的本地运行队列中获取goroutine
            gp, inheritTime = runqget(_g_.m.p.ptr())
            if gp != nil && _g_.m.spinning {
                throw("schedule: spinning with local work")
            }
        }
        if gp == nil {
            //如果从本地运行队列和全局运行队列都没有找到需要运行的goroutine，
            //则调用findrunnable函数从其它工作线程的运行队列中偷取，如果偷取不到，则当前工作线程进入睡眠，
            //直到获取到需要运行的goroutine之后findrunnable函数才会返回。
          gp, inheritTime = findrunnable() // blocks until work is available
        }
        //跟启动无关的代码.....
        //当前运行的是runtime的代码，函数调用栈使用的是g0的栈空间
        //调用execte切换到gp的代码和栈空间去运行
        execute(gp, inheritTime)  
    }
    
    func execute(gp *g, inheritTime bool) {
        _g_ := getg() //g0
        //设置待运行g的状态为_Grunning
        casgstatus(gp, _Grunnable, _Grunning)
     
        //把g和m关联起来，此时将g0的m绑定到gp上完成转换
        _g_.m.curg = gp 
        gp.m = _g_.m
        //......
        //gogo完成从g0到gp真正的切换,gogo做到CPU权利的转让以及栈的切换 
        gogo(&gp.sched)
    }
    
    //gogo是汇编方法，切换栈以及跳转到即将执行的goroutine的地址
    //执行main gorouinte
    // 主要做的事儿：
    //1. 创建sysmon负责整个程序的调度,gc以及epoll
    //2. runtime的初始化
    //3. import包的初始化
    //4. main程序的执行
    //5. exit
    func main() {
        g := getg()  // g = main goroutine，不再是g0了
       //64位系统上每个goroutine的栈最大可达1G
        if sys.PtrSize == 8 { 
            maxstacksize = 1000000000
        } else {
            maxstacksize = 250000000
        }
        // Allow newproc to start new Ms.
        mainStarted = true
        if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
            //现在执行的是main goroutine，所以使用的是main goroutine的栈，需要切换到g0栈去执行newm()
            systemstack(func() {
                //创建监控线程，该线程独立于调度器，不需要跟p关联即可运行
                 newm(sysmon, nil)
            })
        }
        
        ......
        //调用runtime包的初始化函数，由编译器实现
        runtime_init() 
        // Record when the world started.
        runtimeInitTime = nanotime()
        gcenable()  //开启垃圾回收器
        ......
        //main 包的初始化函数，也是由编译器实现，会递归的调用我们import进来的包的初始化函数
        fn := main_init // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
        fn()
        //调用main.main函数
        fn = main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
        fn()
        
        //进入系统调用，退出进程，可以看出main goroutine并未返回，而是直接进入系统调用退出进程了
        exit(0)
        
        //保护性代码，如果exit意外返回，下面的代码也会让该进程crash死掉
        for {
            var x *int32
            *x = 0
        }
    }
    ```

    首先从全局或者本地找，没有则偷取goroutine 然后调用execute从g0转换到goroutine执行, goroutine的切换其实就是CPU寄存器的切换以及栈内存的切换

11. 为什么在创建goroutine的newproc函数要传入参数大小

    当你创建一个新的goroutine时要使用一个全新的栈，而不能使用当前goroutine的栈，而newproc是不知道要拷贝多少数据到新的栈，所以需要指定

12. 什么时候调用的main函数？

    g0切换到main goroutine调用gogo函数时CPU就跳转到了runtime.main函数 schedu-->excute-->gogo此时跳转到runtime.main

13. g0到main goroutine的转换过程

    1. 保存g0的调度信息
    2. 寻找队列中需要执行的goroutine，此时是main goroutine
    3. g0到goroutine的栈切换JMP 跳到runtime.main
    4. goroutine的退出(exit)

    

    扩展所有程序执行时的系统操作

    从磁盘上读取可执行的文件，加载到内存

    创建进程和主线程

    为线程分配栈空间

    将用户输入的命令加载到主线程的栈空间

    将主线程分配到操作系统的运行队列等待其被调度





看完头都秃了，原文地址会有更详细的讲解包括编译什么的

https://mp.weixin.qq.com/s/3xXU_O8-ZiA1Tk1nCnkODA



https://mp.weixin.qq.com/mp/homepage?__biz=MzU1OTg5NDkzOA==&hid=1&sn=8fc2b63f53559bc0cee292ce629c4788&scene=1&devicetype=android-29&version=2800015d&lang=zh_CN&nettype=WIFI&ascene=7&session_us=gh_8b5b60477260&wx_header=1



# 其他问题

#### 1. 面对未知的流量暴增，可以预先怎么处理？

[如何应对网站流量暴增](https://www.cnblogs.com/dadonggg/p/8651909.html)

如果流量突然增大，那么总会有一个资源会出现瓶颈。按照经验，大概出问题的地方是 DB，磁盘 io，cpu，带宽，连接数，内存其中的一个或几个业务。

流量分为两种：

- 可预测流量（例如微博突然爆发的社会热点，运营的营销活动火爆）
- 不可预测流量（网站被恶意刷量；合作伙伴疯狂调平台接口）

预备方案：

1. 流量估算，性能预留

   一般来说，可以设计流量*3 作为系统压力的下限，压测时要达到流量\*3 的标准。提供一定的缓冲带宽，如果是云服务器，可以临时加带宽。

2. 提供降级方案，针对业务进行降级

   降级不能瞎降，能降级成啥样，显示成什么样子，都要预先设计好，做演练。作为后台服务器，可以自适应流量降级，也可以预先埋点做好开关，一旦设置，立刻进入降级方案。

    但是，如果核心服务就是热点本身，就没得降级，比如，电商的双十一，用户的购买，下单等行为，下单就是下单，不能下一半，不能砍掉支付，不能随机性有的能买有的不能买，是涉及到大量写操作，而且是核心链路，无法降级的，这个时候，限流就比较重要了。

3. 限流方案

   限流常用的方式有：计数器、令牌桶、漏桶、滑动窗口。

#### 2. 如何限流？有哪些限流算法？对于 ddos 攻击怎么处理？

限流常用的方式有：计数器、令牌桶、漏桶、滑动窗口。







## 收藏的面经

1. [字节PHP/Golang社招](https://www.nowcoder.com/discuss/638404?type=post&order=time&pos=&page=1&channel=-1&source_id=search_post_nctrack)



### 面试遇到的问题

##### 1. 服务注册与服务发现是如何实现的？

##### 2. cache server 的架构？

##### 3. binlog 日志不同，你们是如何解决的？

##### 4. 如何保证 binlog 的消费顺序？

[实时数据同步服务(canal+kafka)是如何保证消息的顺序性？](https://www.cnblogs.com/itdream/p/13510860.html)

> 1. 消息生产端将消息发送给同一个 MQ 的同一个 partition，并且按顺序发送
> 2. 消息消费端按照消息的发送顺序进行消费
>
> ![](https://img2020.cnblogs.com/blog/414002/202008/414002-20200815224725680-751244705.png)
>
> 如上图，我们可以在解析 binlog 之后按照数据行的主键 id 进行 hash 取模运算，分配到一个 partition，然后就能够保证消息的顺序消费。
>
> ##### 重要的一点是，kafka的同一个 partition 内的消息是有序的，所以只要按照顺序发送消息到同一个 topic，那么就能够保证顺序消费。
>
> - kafka 的同一个 partition 用一个write ahead log组织， 是一个有序的队列，所以可以保证FIFO的顺序；
> - 因此生产者按照一定的顺序发送消息，broker 就会按照这个顺序把消息写入 partition，消费者也会按照相同的顺序去读取消息；
> - kafka 的每一个 partition 不会同时被两个消费者实例消费，由此可以保证消息消费的顺序性。

##### 5. 蓄水池抽样算法（Reservoir Sampling）

##### 6. 如何用 mysql 实现分布式锁？

> mysql 有专门的原语函数

