# GoRoutine

1. 协程:

   创建一个线程需要4MB, 高并发场景下,  32位机器的4G虚拟内存明显不够, 同时大量的线程切换也会消耗过多资源, 于是出现了**用户级线程**, 即协程(co-routine)

   N:1: N个协程绑定一个内核线程, 多个协程切换时不会进入内核态, 切换比较快, 但是这样发挥不了多核的能力, 也可能会造成一个协程阻塞, 饿死其他协程的问题

    N:M: 实现复杂, 但是优点较多

   **协程是协作式的(只有一个协程主动让出cpu时才会进行切换), 而线程是抢占式的, 由cpu调度**

2. Go的线程

   goroutine为轻量级的线程抽象, 它们只存在程序的虚拟内存空间, 由go自己的调度器进行调度. 内存占用才几KB, 

3. goroutine切换:

   goroutine是协同调度的, 在这种调度中，Goroutine在空闲或逻辑阻塞时会定期产生控制权，以便同时运行多个Goroutine。因此它们只在特定情况下会进行切换:

   - channel因send或者receive时阻塞
   - go语句
   - 阻塞的系统调用(如文件, 网络操作)
   - gc的stw之后

4. 如何调度:

   1. 三个抽象:

      - P: process, 处理器,  由参数GOMSXPROCS控制
      - M: OSThread, 系统线程, M的数量小于等于P
      - G: Goroutines

      ![](/home/halo/notes/goroutine调度.png)

   2. OSthread(M)被CPU(P)调度, 而G(Goroutine)被OSthread(M)调度, 因此golang的调度器是M:N的, M个Goroutine会被N个系统线程调度(而java线程是1:1, 相当于映射到一个系统线程)

   3. OSthread如何调度goroutine?

      1. 每个P都有一个本地G队列, 同时还会有一个全局的G队列(数量小于256), 当本地G队列满了, 将其一半移动到全局G队, 当本地G队空了, 就从其他P的G队偷一半过来

      2. 每个M会被映射到一个P上,  但P可能也会空闲出来(如果M都被系统调用等阻塞时), 在任何时候, 最多出现GOMAXPROCS个P且只有一个M可运行在一个P上面, M可以被调度器创建

      3. 在每一轮调度, 调度器找到一个runnable的G, 然后运行, 知道它被阻塞

         <img src="/home/halo/notes/goroutine调度2.png" style="zoom:150%;" />

         ```
         Search in the local queue
                 if not found 
                     Try to steal from other Ps' local queue(1/2)
                     if not found 
                         Search in the global queue         Also periodically it searches in the global queue (every ~ 1/70)
         ```
      
   4. 如何调度M:

      1. 程序启动时, 系统线程就会被创建, P在启动时就会被创建
      2. 当有新的goroutine可以运行时会唤醒一个P, 或者没有足够M来运行P中的可运行G时 ,然后P会创建一个与之对应的M, 即系统线程
      3. 当P没事干时, 会进入空闲列表, 当M没goroutine可以运行时也会进入空闲列表, 
      4. 当一个G被放到空闲列表时, 需要保存其PC和栈

   5. 系统调用:

      go优化了系统调用, 不管此调用是否阻塞, 都断联此M和P, 然后把M(有系统调用的)和其他的空闲的P重新关联

      调度器持续调度直到满足以下规则:

      - 尝试获取队头的P, 恢复运行
      - 尝试在P空闲列表获取一个P, 恢复运行
      - 将此G放在全局G列表, 将M放回到空闲列表

      但是当系统调用完成, 但资源还未准备好(如http调用), 此时P不会和M断连, 而是迫使Go使用网络轮询器并挂起G, 直到网络轮询器通知资源已准备好, 此时M才会进行运行这个G