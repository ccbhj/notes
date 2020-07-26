# AbstractQueuedSynchronizer(AQS)

1. 定义: 队列同步器AbstractQueuedSynchronizer(以下简称同步器),是用来构建锁或其他同步组件的基础框架,**它使用了一个int成员变量(volatile int state)表示同步状态,通过内置的FIFO列来完成资源获取线程的排队工作, 用CAS来控制其状态改变**
2. LOCK接口: Java SE5提供的锁功能, 用于更加灵活的锁控制,虽然缺少了(通过synchronized块或者方法所提供的)隐式获取释放锁的便捷性,但是却有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized键字所不具备的同步特性。
3. 使用- 同步器的主要使用方式是继承(模板设计模式),子类通过继承同步器并实现它的抽象方法来管理同步态,在抽象方法的实现过程中免不了要对同步状态进行更改,这时就需要使用同步器提供的3个方法(getState()、setState(int newState) compareAndSetState(int expect,int update))来进行操作,因为它们能够保证状态的改变是安全的。
  - 子类推荐被定义为自定义同步组件的静态内部类,同步器自身有实现任何同步接口,它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步件使用,同步器既可以支持独占式地获取同步状态,也可以支持共享式地获取同步状态这样就可以方便实现不同类型的同步组件(ReentrantLock ReentrantReadWriteLock和CountDownLatch等)。
  - 需要重写方法
      - 独享类: tryAcquire(), tryRelease()
      - 共享类: tryAcquireShare(), tryReleaseShare()
      - 是否独享式: isHeldExclusively()
4. 以ReentrantLock为例, **volatile int state初始化为0, 表示未锁定.** 当有线程调用lock()时, 会用tryAcquire(), 使state += 1独占该锁, 当线程调用unlock时, 使state -= 1, 直到state=0,其他线程才能获取锁, 释放锁之前，线程自己是可以重复获取此锁的（state会累加, **这里把state看作成一个资源, 要获取锁就是要获取资源**
5. 以CountDownLatch为例, 任务分给N个线程去执行, state也初始化为N, 之后每个线程完成时调用countDown(), state会用CAS减1, 直到state = 0, 这时可以调用unpark()调用主线程, 主线程会从await()中返回, 

4. 实现:

   1. 依赖一个同步队列来完成状态的管理, 当前线程获取同步状态失败时, 将其信息加入队列, 并阻塞当前线程. 当同步状态释放时, 会将首节点唤醒, 便尝试获取同步状态.

   2. 队列节点:用于保存获取锁失败的线程的引用, 前后驱节点等

   3. 等待状态(waitStatus, 而不是state, state是全局的 ): 
        1. CANCELED: 值为1, 当某节点被中断或取消(超时)时, 进入此状态, 相当于废弃,**永远不会给阻塞**, 所有节点在都会被跳过, 队头不会处于这个状态, 它的前驱的next会指向下一个结点
        2. SIGNAL: 值为-1, 当前节点的后继被挂起, 需要被通知, 后继结点入队时, 所以当当前线程释放状态, 会通知后继节点, 使得后继节点可以开始运行
        3. CONDITION: 值为-2, 节点在等待队列中(不可被用于同步队列, 直到其状态被设为INITIAL(0 )), 节点线程等待在Condition上, 当其他线程对Condition类调用Signal方法时, 该节点会从等待队列移到同步队列, 加入到同步状态的获取中.
        4. PROPAGATE: 值为-3, 共享模式下, 不止可能唤醒他的后继结点, 他后面的其他结点都可能给唤醒
        5. INITAL: 值为0, 入队时的初始化状态,
        
   4. 注意，**负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0、<0来判断结点的状态是否正常**。

   5. 是prev, next: 前后驱, **节点是从尾部添加的, next, pre, waitStatus都是volatile的, 所以可以用CAS**

   6. nextWaiter: 等待队列中的后继节点(状态为CONDITION的节点), 如果当前的节点是共享的, 那么当前节点是SHARED变量, 也就是说节点类型(独占或者共享)和这个字段是同一个字段

   7. ### acquire(), 获取锁: 

        此方法为独占模式下线程获取资源的入口(可用于实现lock())

        1. tryAcquire() 尝试获取锁, 成功直接返回(体现了非公平锁的抢占语义, 每次获取时先抢一下试试), 这个方法需要由开发者自己去实现

        2. addWaiter()将线程加入等待队列尾部(线程将被添加的队列后, 此时需要用compareAndSetTail来设置尾节点),

           ```java
           private Node addWaiter(Node mode) {
               //以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享）
               Node node = new Node(Thread.currentThread(), mode);
           
               //尝试快速方式直接放到队尾。
               Node pred = tail;
               if (pred != null) {
                   node.prev = pred;
                   if (compareAndSetTail(pred, node)) {
                       pred.next = node;
                       return node;
                   }
               }
           
               //上一步失败则通过enq入队。
               enq(node);
               return node;
           }
           
           private Node enq(final Node node) {
               //CAS"自旋"，直到成功加入队尾	
               for (;;) {
                   Node t = tail;
                   if (t == null) { // 队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。
                       if (compareAndSetHead(new Node()))
                           tail = head;
                   } else {//正常流程，放入队尾
                       node.prev = t;
                       if (compareAndSetTail(t, node)) {
                           t.next = node;
                           return t;
                       }
                   }
               }
           }
           ```

           

        3. acquireQueued()使线程阻塞在等待队列中获取资源，一直获取到资源后才返回。若在等待过程被中断过, 则要返回True, 否则返回False

           ```java
           final boolean acquireQueued(final Node node, int arg) {
               boolean failed = true;//标记是否成功拿到资源
               try {
                   boolean interrupted = false;//标记等待过程中是否被中断过
           
                   //又是一个“自旋”！
                   for (;;) {
                       final Node p = node.predecessor();//拿到前驱
                       //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
                       if (p == head && tryAcquire(arg)) {
                           //拿到资源后，将head指向该结点。老大要出队, 所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                           setHead(node);
                           // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                           p.next = null; 
                           failed = false; // 成功获取资源
                           return interrupted;//返回等待过程中是否被中断过
                       }
           
                       // shouldPark... 用于告知有效的前驱在完成后通知自己(将前驱的waitStatus设为SIGNAL), 如果前驱无效(waitStatus>0), 就找前驱的前驱
                       // parkAndCheckInterrupt使线程休眠, 在唤醒后检查其中断位并重置
                       if (shouldParkAfterFailedAcquire(p, node) &&
                           parkAndCheckInterrupt())
                           interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
                   }
               } finally {
                   if (failed) // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
                       cancelAcquire(node);
               }
           }
           ```

           

        4. 若线程在等待过程被中断了, 他就不会响应acquiredQueued()的中断, 所以还要再中断一次

        ```java
        public final void acquire(int arg) {
            if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                selfInterrupt();
        }**当一个线程完成任务, 便唤醒下个节点, 使其成为首节点, 此时不需要CAS, 因为只有一个线程在使用头节点.**
        ```
        
        <img src="/home/halo/notes/java/aqs_acquire.png" style="zoom:150%;" />
        
   8. ### release(), 释放锁:
   
        此方法是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。
   
        ```java
        public final boolean release(int arg) {
            if (tryRelease(arg)) {
                Node h = head;//找到头结点
                // 若队列中没有节点, 直接返回True
                // 若队列中节点状态为init, 也返回True, 因为可以去抢占
                if (h != null && h.waitStatus != 0)
                    unparkSuccessor(h);//唤醒等待队列里的下一个线程
                return true;
            }
            return false;
        }
        ```
   
        1. tryRelease(arg): 由用户实现, 用于判断资源是否已经释放, state -= arg 
   
        2. unparkSuccessor(Node node):  用于唤醒下个节点持有的线程, **用unpark()唤醒等待队列中最前边的那个未放弃线程**
   
           ```java
           private void unparkSuccessor(Node node) {
               //这里，node一般为当前线程所在的结点。
               int ws = node.waitStatus;
               if (ws < 0)//置零当前线程所在的结点状态，允许失败。
                   compareAndSetWaitStatus(node, ws, 0);
           
               Node s = node.next;//找到下一个需要唤醒的结点s
               if (s == null || s.waitStatus > 0) {//如果为空或已取消
                   s = null;
                   for (Node t = tail; t != null && t != node; t = t.prev) // 从后向前找。
                       if (t.waitStatus <= 0)//从这里可以看出，<=0的结点，都是还有效的结点。
                           s = t;
               }
               if (s != null)
                   LockSupport.unpark(s.thread);//唤醒
           }
           ```
   
   9. ### aquiredShared(arg),
   
        共享模式获取锁, 获取成功则返回, 失败进入队列
   
        ```java
        public final void acquireShared(int arg) {
            if (tryAcquireShared(arg) < 0)
                doAcquireShared(arg);
        }
        ```
   
        1. tryAcquiredShared(arg) 由用户实现
   
        2. doAcquiredShared(arg)用于将线程加入等待队列尾部进行休眠, 直到其他线程唤醒自己
   
           ```java
           private void doAcquireShared(int arg) {
               final Node node = addWaiter(Node.SHARED);//加入队列尾部
               boolean failed = true;//是否成功标志
               try {
                   boolean interrupted = false;//等待过程中是否被中断过的标志
                   for (;;) {
                       final Node p = node.predecessor();//前驱
                       if (p == head) {//如果到head的下一个，因为head是拿到资源的线程，此时node被唤醒，很可能是head用完资源来唤醒自己的
                           int r = tryAcquireShared(arg);//尝试获取资源
                           if (r >= 0) {//成功
                               setHeadAndPropagate(node, r);//将head指向自己，还有剩余资源可以再唤醒之后的线程
                               p.next = null; // help GC
                               if (interrupted)//如果等待过程中被打断过，此时将中断补上。
                                   selfInterrupt();
                               failed = false;
                               return;
                           }
                       }
           
                       //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
                       if (shouldParkAfterFailedAcquire(p, node) &&
                           parkAndCheckInterrupt())
                           interrupted = true;
                   }
               } finally {
                   if (failed)
                       cancelAcquire(node);
               }
           }
           ```
   
           
   
   10. ### releaseShared(arg)
   
        释放共享资源, 并唤醒队列中的其他线程
   
        ```java
        public final boolean releaseShared(int arg) {
            if (tryReleaseShared(arg)) {//尝试释放资源
                doReleaseShared();//唤醒后继结点
                return true;
            }
            return false;
        }
        ```
   
        1. tryReleaseShared()由用户实现
   
        2. doReleaseShared() 用于唤醒其他进程
   
           ```java
           private void doReleaseShared() {
               for (;;) {
                   Node h = head;
                   if (h != null && h != tail) {
                       int ws = h.waitStatus;
                       if (ws == Node.SIGNAL) {
                           if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                               continue;
                           unparkSuccessor(h);//唤醒后继
                       }
                       else if (ws == 0 &&
                                !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                           continue;
                   }
                   if (h == head)// head发生变化
                       break;
               }
           }
           ```
   
   11. ### **Condition**:
   
         ```java
         public static void main(String... args) {
             ReentrantLock lock = new ReentrantLock();
             Condition cond = lock.newCondition();
             new Thread(() -> {
                 lock.lock();
                 try {
                 	System.out.println("1 lock");
                     cond.await(); // release and wait
                     System.out.println("1 waken");
                 } catch(Exception e) {
                     e.printStackTrace();
                 } finally {
                     lock.unlock();
                 }
             }).start();
             
                 new Thread(() -> {
                 lock.lock();
                 try {
                 	System.out.println("2 lock");
                     cond.signal(); 
                 } catch(Exception e) {
                     e.printStackTrace();
                 } finally {
                     lock.unlock();
                 }
             }).start();
         }
         ```
   
         cond.wait(): 解锁并挂起线程, 并加入condition等待队列
   
          cond.signal()唤醒在condition等待队列中的线程
