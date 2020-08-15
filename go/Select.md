# Select 

1. ### 作用:

   - 在多个channel上进行非阻塞的收发
   - 在多个channel同时响应时随机选择(避免后面的channel被饿死)

2. ### 数据结构:

   ```go
   // 运行时的case
   type scase struct {
   	c           *hchan
   	elem        unsafe.Pointer
   	kind        uint16
   	pc          uintptr
   	releasetime int64
   }
   // kind种类
   const (
   	caseNil = iota
   	caseRecv
   	caseSend
   	caseDefault
   )
   
   ```

3. ### 持有channel的多个情况:

   1. 没有case: 直接阻塞(调用runtime.block直接挂起), 此goroutine不会被唤醒

   2. 只有一个case: 被改写成 

      ```go
      if ch == nil {
          block()
      }
      v, ok := <- ch
      ```

      

   3. 含有多个case且有default: 非阻塞操作使用if/else改写

4. 执行流程:

   1. 初始化: 执行必要的初始化并决定case的两个顺序, pollOrder/lockOrder

      pollOrder(轮询顺序): 引入随机性, 避免饥饿现象

      lockOrder(加锁顺序): 按channel地址排序, 避免死锁

   2. 循环(selectgo):

      1. 根据pollOrder遍历查找是否有就绪的channel:
         1. 有就返回对应的case索引
         2. 没有就将当前的goroutine加入到所有相关的channel相关队列上, 等待被唤醒
      2. 唤醒当前的goroutine之后按照lockOrder锁住channel并且遍历找到对应的channel