# channel

1. 特点:

   1. 先进先出
   2. 并发安全: 互斥锁

2. 数据结构:

   ```go
   type hchan struct {
   	qcount   uint  // 元素个数
   	dataqsiz uint  // 循环队列长度
   	buf      unsafe.Pointer  // 缓存区指针
   	elemsize uint16
   	closed   uint32
   	elemtype *_type
   	sendx    uint   // 发送位置
   	recvx    uint   // 接受位置
   	recvq    waitq  // 因缓冲区超过容量而阻塞发送的队列
   	sendq    waitq  // 因缓存没数据而阻塞接受队列
   
   	lock mutex
   }
   ```

3. 发送数据流程

   1. 先加锁
   
   2. 确认是否关闭
   
   3. - 当存在等待接收者时, 调用runtime.send发送数据:
   
        将数据拷贝到接收变量并将阻塞的goroutine标记为可运行状态, 放到运行队列里面去
   
      - 缓冲区有空间时拷贝到缓冲区, 并且递增qcount和sendx 
   
      - 不存在缓冲区或者已满了的话, 等待其他goroutine接受数据
   
4. 接受数据流程:

   1. 加锁

   2. 若发送队列不为空: 

      1. 当没有缓冲区: 将channel发送队列中的goroutine的elem地址直接拷贝到接受者变量内存中
      2. 有缓冲区: 
         1. 将队列中的数据拷贝到接受者的变量

      **无论如何都要取出发送队列头, 释放一个阻塞的发送者**

   3. 注意: channel为空直接挂起当前的goroutine

   4. 若channel关闭且没有数据时, 直接返回

   5. 当channel为空或者缓冲区没有数据且没发送者时会触发goroutine调度

   