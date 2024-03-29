# Volatile

1. ### Java内存模型
    Java中, 所有实例域, 静态对象和数组都存储在堆中, **堆在线程间共享. 局部变量, 方法定义参数, 和异常处理器不会在线程间共享**. 线程间通信通过JMM, JMM规定: **每个线程有一个本地内存, 本地内存是个抽象概念**, 

  **Java内存模型是共享内存的并发模型，线程之间主要通过读-写共享变量（堆内存中的实例域，静态域和数组元素）来完成隐式通信。**

2. ### 指令重排序
    为提高性能, 编译器会与处理器会进行指令重排序, 3种情况下会有重排序:
    
      1. 在不改变单线程语义的前提下, 编译器优化的重排序
    
      2. 现代处理器采用的指令流水线并行技术, 若不存在数据依赖性则可以进行重排序
    
      3. 由于处理器使用缓存, 内存系统会**被动地**进行重排序.
    
         
    
         以上2, 3为jvm不可知的重排序, 会造成内存可见性问题, 因此编译器会在生成指令序列时插入内存屏障
    
         屏障类型如下:
+ Load-Lood: Load1;LoadLoad;Load2 该屏障(lfense)确保Load1数据的装载先于Load2及其后所有装载指令的的操作 
+ Load-Store:Load1;LoadStore;Store2 确保Load1的数据装载先于Store2及其后所有的存储指令刷新数据到内存的操作 
+ Store-Store: Store1;StoreStore;Store2	该屏障(sfense)确保Store1立刻刷新数据到内存(使其对其他处理器可见)的操作先于Store2及其后所有存储指令的操作
+ Store-Load:Store1;StoreLoad;Load2	该屏障(mfense)确保Store1立刻刷新数据到内存的操作先于Load2及其后所有装载装载指令的操作。它会使该屏障之前的所有内存访问指令(存储指令和访问指令)完成之后,才执行该屏障之后的内存访问指令


3. ### happens-before 规则
    在JMM中, 如果一个操作执行的结果需要对另一个操作可见, 那么这两个操作必须存在happens-before关系, 可以在不同线程间生效. JMM规定的规则有以下几个:

    
      1. 程序顺序规则: 一个线程中的每个操作提前发生于后续所有操作
      2. 监视器锁规则: 加锁提前发生于解锁
      3. volatile规则: 一个volatile域的写提前发生于它的读 
      4. 传递性: A 提前发生与 B, B提前发生于C, 于是A提前发生于C
    
4. ### 数据依赖性:
两个操作同时访问同一个变量, 且这两个其中有一个为写操作, 则两个操作存在数据依赖性, 包括: 读后写, 写后写, 读后写

5. ### as-if-serial语义
不管怎么排序, 程序的结果必须不能被改变. 但是如果两个操作之间不存在数据依赖性, 则可以进行重排序而不影响程序结果.

6. ### 总线事务
    总线事务包括读事务与写事务, 读事务从把内存送到处理器, 写事务从处理器写到内存. **总线会同步试图并发使用总线的事务, 在处理器执行事物期间, 总线会禁止其他处理器与IO设备读写内存, 保证在任意时间点, 最多只有一个处理器可访问操作**
    
7. ### MESI缓存一致性协议:

    四个状态: Modified(脏缓存), Exclusive(独占缓存), Share(共享), Invalid(过时)

    **MESI协议保证了每个缓存中使用的共享变量的副本是一致的。**它核心的思想是：当CPU写数据时，如果发现操作的变量是共享变量，即在其他CPU中也存在该变量的副本，会发出信号通知其他CPU将该变量的缓存行置为无效状态，因此当其他CPU需要读取这个变量时，发现自己缓存中缓存该变量的缓存行是无效的，那么它就会从内存重新读取。

7. ## volatile的内存语义
    1. volatile变量自身有三个特性:
    - 可见性: 对一个volatile变量的读, 总是能被任意线程看到这个变量最后的写操作 (lock指令前缀锁内存总线, 缓存一致性协议)
    - 原子性: 对单个volatile变量的读写有原子性, 但是类似++这种操作没原子性 
    - 有序性: 写发生在读之前.

    2. volatile读写的内存语义(用了**总线锁和缓存一致性协议**):
    **当写一个volatile变量时, JMM会把该线程对应的本地内存中的共享变量刷新到内存进去**
    **当读一个volatile变量时, JMM会把该线程的**所有**本地内存置为无效, 然后从主内存读取值**

    3. 内存语义的实现: 
       - 禁止指令重排序: 
       - 当第一个操作为普通变量读写时, 第二个操作为volatile写, 不可重排序
           - 当第一个操作为volatile读时, 不管第二个操作是什么, 都不能重排序
           - 当第一个操作为volatile写时, 第二个操作是volatile读时, 不能重排序
       - 实现: 
           - 在volatile写前插入StoreStore屏障(保证写顺序)
           - 写后插入StoreLoad屏障(保证先写后才能读)
           - 在volatile读后插入LoadLoad屏障和LoadStore屏障(保证读先)
    4. 通过内存屏障的插入, 使得volatile变量有了锁一样的语义, 且比锁更轻量级. 因为在volatile写必须在普通变量的读写之后

    5. 总结:

       - 线程A写一个volatile变量,实质上是线程A向接下来将要读这个volatile变量的某
       线程发出了(其对共享变量所做修改的)消息。 
       - 线程B读一个volatile变量,实质上是线程B接收了之前某个线程发出的(在写这
        volatile变量之前对共享变量所做修改的)消息。 
       - 线程A写一个volatile变量,随后线程B读这个volatile变量,这个过程实质上是线
         A通过主内存向线程B发送消息。


8. ## 锁的内存语义:

   1. 锁的加锁happens-before解锁
   2. ** 当线程释放锁时, 会将线程的本地内存的共享变量写入主内存**
   3. ** 当线程加锁时, JMM 会把线程的本地内存置为无效, 必须从主内存重新读取**

9. ## CAS的内存语义:
- CAS具有volatile读写的内存语义, ** 即编译器不能对CAS和CAS前后的任意内存操作进行重排序**
- 编译器会根据处理器类型决定是否在cmpxchg指令前添加lock前缀:
    1. 添加lock前缀会锁住内存总线. 若只有单个内存处理器则不会添加lock
    2. 禁止该指令与其前后的读写指令进行重排序
    3. 把写缓冲区的所有数据全部刷新到内存中
 以上2, 3 点便可以实现volatile语义
- CAS使用现代处理器提供的原子指令

10. ## final的内存语义
- final遵守两个重排序规则:
  1. 在构造函数里对一个final域的写入, 与随后把这个域的持有对象的引用赋值给另一个引用, 这两个操作不能重排序
  2. 初次读这个final域的持有对象, 与随后初次读这个final域, 这两个操作不能重排序.
- 写final域的规则, **确保在任意对象对任意线程可见之前(被引用), 就已经被正确初始化了**:
  1. JMM禁止编译器的写重排序到构造函数之外
  2. 编译器会在final写后, 构造函数的return之前插入一个StoreStore屏障, 将final域刷新进主内存
- 读final域的规则: 在一个线程中, 初次读对象引用和初次读该对象包含的final域, 这两操作禁止重排序,编译器会在读final之前插入LoadLoad屏障, 
                   尽管这两个操作有间接数据依赖, 但是有些处理器允许对这种进行重排序, 于是有这个规则
                   **这个规则可确保: 在读一个对象的final域之前, 一定会想读这个持有对象的引用 **
