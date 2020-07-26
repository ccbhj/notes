# 设备管理
1. # I/O硬件原理

1. 块(block)设备: 设备把信息存在大小固定的块中,所有传输都以块为单位, 每个块都能独立于其他块进行读写, 如硬盘, usb

2. 字符设备: 设备以字符为单位发送或接受一个字符流, 字符设备不可寻址. 如打印机, 鼠标,网卡

3. 设备控制器(适配器): 把串行的位流转换为字节块, 并进行错误检查. 它首先在缓冲区进行数据的组装,  再校验成功之后转如入内存.

4. 内存映射IO: 每个控制器中都有几个寄存器, 操作系统通过向读写这些寄存器来与IO设备通信, 此外,还有数据缓冲区, 供操作系统读写, 

5. 那缓存区和寄存器如何通信? 

   1. 第一种: 为每一个设备寄存器分配一个端口号, 一个8或16位整数, 所有端口形成IO端口空间, 使得普通用户程序无法访问.

   2. 第二种: 将所有控制寄存器映射到内存空间, 并分配唯一的内存地址, 称为内存映射IO

      

2. ### I/O软件原理

   + 统一命名以解决软件通用性
   + 错误处理应该尽可能在接近硬件的层面得到处理
   + 异步/同步传输
   + 缓冲
   + 共享设备与独占设备

   1. #### 程序控制I/O: 让cpu做全部工作

      + 由操作系统控制I/O
      + 缺点: 直到I/O结束前CPU都给占用

   2. #### I/O软件层次

      ​	**用户级别I/O**

      ​		$\downarrow$

      ​	**与设备无关的操作系统软件**

      ​		$\downarrow$

      ​	**设备驱动程序**

      ​		$\downarrow$

      ​	**中断处理程序**

      ​		$\downarrow$

      ​		**硬件**
      1. 中断处理程序: 使阻塞的驱动程序不阻塞

      2. 设备驱动程序: 

         + 运行在内核层面或用户层面

         + 具有通用性

         + 结构: 检查输入参数$\rightarrow$ 检查当前设备是否在使用$\rightarrow$ 若忙, 则加入队列

           ​									   $\rightarrow$ 若空闲, 则就绪  $\rightarrow$ 写入控制器寄存器

           

   3. #### 磁盘

      1. ###### **重叠寻道**: 控制器可同时控制两个或多个驱动器进行寻道, 可以同时进行一个读写,但是不能多个同时读写

      2. **磁盘格式化**: 

         + 扇区格式: 前导码(柱面, 扇区号等)  +  数据  +  ECC(冗余信息, 用于恢复读错误)
         + 柱面斜进: 每个磁道的开始扇区(0扇区)不再同一个扇区, 其偏移量则为柱面斜进
         + 单交错/双交错: 为了给数据复制到主存中间以喘息的时间, 扇区交错编码, 而不用等待旋转一周的时间, 连续的扇区号中间隔一个扇区叫单交错, 隔两个叫双交错
         + 分区: 进行逻辑分区, 每个分区就像一个磁盘一样

      3. **磁盘臂调度算法**: 

         1. 性能因素: 寻道时间(最主要的), 旋转延迟, 实际传输时间
         2. **先来先服务(FCFS)**: 将柱面号排成一张表, 每个表项存放一个链表, 链表存放所有柱面上的请求, 当一个柱面请求完成后, 可以按请求顺序处理.
         3. **最短寻道优先算法(SSF)**: 改进的FCFS, 按离本次柱面最近的柱面先处理, 缺点是柱面表两端的请求可能需要等待很久
         4. **单向扫描**: 一个周期内, 磁头从柱面表0开始向后扫描, 遇到请求便处理, 直到表尾后再重新从表0开始扫描
         5. **双向扫描**: 一个周期内, 磁头从柱面表0开始向后扫描, 遇到表尾则从表尾开始向前处理请求
         6. **电梯算法**: 磁头按双向扫描方向扫描, 但维护一个方向位, 当后面/前面没有请求时, 则将位置位取反.

      4. **读取速度优化**

         1. 高速缓存: 每次读取多个扇区到高速缓存区, 即使没有请求那个扇区