# Redis

1. ## 字符串
    1. 底层实现:  **SDS (simple dynamic string)**

    2. ```c
        struct sdshdr {
        	int len;
        	int free;
        	char buf[];
        }
        ```
    3. 优点: 
        - 常数时间获得长度
        - 杜绝缓冲区溢出
        - 减少修改时的内存分配
        - 可保存二进制或者文本数据
        - 兼容部分C函数
        
    4. 优化策略:
        1. 空间预分配:
            增长时若len小于1MB, buf将多分配和len一样的大小, 此时len = free
            
            大于等于1MB, 分配1MB给buf, 此时 free = 1MB
            
        2. 惰性空间释放:

            并不缩小buf的大小, 而是减少len, 增多free

2. ## 列表
   实现:  #### 链表
	```c
	typedef struct list {
		listNode* head;
		listNode* tail;
		unsigned long len;
		void *(*dup)(void *ptr); // 复制函数
		void *(*free)(void *ptr); // 释放函数
		int (*match)(void *ptr, void *key); // 节点对比函数
        } list;
	```
	特点: 无环, 双端, 有头尾指针, 带计数器, **多态(函数节点利用void指针保存节点)**
	
3. ## 字典
    实现: #### 哈希表
  ```c
  // 哈系表
  typedef struct dictht
  {
  	dictEntry *table;
  	unsigned long size;
  	unsigned long sizemask; // 大小掩码, 用于计算索引, 总是的等于size-1
  	unsigned long used; // 节点已使用数量
  } dictht;
  // 表项
  typedef struct dictEntry
  {
  	void* key; // 键
  	union {  // 值
  		void* val;
  		uint64_t u64;
  		int64_t s64;
  	}
  	struct  dictEntry* next; // 使用链地址法解决冲突
  } dictEntry;
  ```
  sizemask 总是等于size - 1(-1是为了让1出现的次数更多, 使得hash值更加分散);

  **目前redis使用MurmurHash2算法计算哈希值** 

  redis的字典实现:

  ```c
  typedef struct dict {
      dictType *type;  // 类型特定函数
      void *privdata;
      dictht ht[2];  // 另一个表存要迁移的表
      int rehashidx; // rehash进度, 默认为-1
  }
  ```

- 拉链法解决hash冲突: 在每个entry中有next字段, 但hash值产生碰撞时形成链表

- **rehash**: redis要将负载因子(大于1和大于5时)维持在一个范围内, 但键值对过多或过少时进行收缩或扩展, **若此时没有进行BGSAVE, BGREWRITEAOF**, 需要进行迁移:

  1. 为字典ht[1]分配空间, 扩张时分配used * 2的空间, 缩小时分配used*1/2
  2. 将ht[0]上的键值对重新计算hash并且移到ht[1]
  3. 当迁移都完成时释放ht[0]并将ht[0] 设为ht[1], ht[1]置空

-  渐进式rehash: 为保证其他操作不被影响, rehash迁移不会一次性完成:

   1. 分配ht[1]
   2. 将dict->rehashidx 设为0, 表示开始rehash
   3. 在对dict进行操作时, 会顺带进行一次键值对迁移, 并将reahshidx+1 (lazy rehash)
   4. 当比较空闲时, 则每100ms耗费1ms来进行rehash(active rehash)
   5. 当rehash完成后, rehashidx=-1

   优点: 避免集中式rehash带来的巨大计算量

4. ## 跳跃表(skiplist)

   #### 优点: 

   - 平均查找时间复杂度O(logn), 最坏O(n), 媲美平衡树
   - 支持顺序遍历操作
   - 实现简单

   #### 实现:

   ```c
   typedef zskiplistNode struct {
       struct zskiplistLevel {
           struct zskiplistNode *forward;
           unsigned int span;
       } level[];
       
       struct zskiplistNode *backward;
       double score;
       robj *obj;
   } zskiplistNode;
   
   typedef struct zskiplist {
       struct skiplistNode *header, *tail;
       unsigned long length; // 节点数量
       int level;  // 最大层数
   } zskiplist;
   ```

- 层: 每次创建节点时, 会根据幂次定律随机生成一个1-32大小的level数组, 这个大小为高度
- 前进指针: 指向同一层的下一个节点
- 后退指针: 用与从后到前遍历节点(与层无关)
- 跨度(span): 记录同一层的两个节点之间的距离, **span于计算排位**(Rank)
- 分值(score): 所有节点以分值大小从小到大排序

5. #### 整数集合(intset)

   **当一个集合只包含整数时, 集合会用intset来实现**

   实现:

   ```c
   typedef struct intset {
       uint32_t encoding; // 编码方式
       int8_t contents; // 保存元素
   } intset;
   ```

   - encoding 支持INTSET_ENC_INT16(int16), int32, int64的类型
   - 升级: 当添加的元素大于当前编码的最大值, 集合类型要进行升级, 并根据当前元素个数重新分配数组空间, 
   - 降级: 不支持降级

6. #### 压缩列表(ziplist)

   为了节约内存,  有多个节点组成, 可以保存字节数组或者整数. **当一个字典只包含少数的键值对, 且值和键为整数或者短字符串时, 会使用压缩列表**

   **保存的元素有大小/长度限制**

   ```c
   typedef ziplist struct {
       uint32_t zlbytes; // 整个表所占的字节数
       uint32_t zltail: // 记录节点起始到末端有多少个字节
       uint16_t zllen; // 节点数量
      	// 节点
       uint8_t zlend; // 特殊值255, 用于标记结尾
   }ziplist;
   
   typedef ziplistEntry struct {
        previous_entry_length; // 前一节点大小(可以占1或5字节)
        encoding; // 类型及长度
       // content 内容
}
   ```

   
   
   - **previous_entry_length以字节为单位, 前一节点大小小于254时则占为1字节, 大于占5字节(其中首个字节被设为0xFE(254), 后4个字节表示长度) **
   - **连锁更新**:前一个节点大小升级会让后面节点更新previous_entry_length所占大小, 从而让整个节点大小增加4, 如果增加后节点大小大于254, 则又会引起后面的节点更新, 时间复杂度最坏O(n^2), 但是不会造成许多性能问题: 
     - 只有大小介于250-253之间的节点才会引起更新, 很少会同时连续的出现很多这样的节点
     - 就算出现连锁更新, 也不会有很多节点参与到其中
   
7. ## 对象编码

   1. 字符串:

      1. 整数字符串: 保存为long
      2. 长度大于32字节的字符串: raw(SDS)
      3. 长度小于等于32字节: embstr(redisObject连续的SDS)

   2. 列表:

      1. 所有字符串小于64字节且元素数量小于512个,  使用**ziplist**
      2. 否则使用**linkedlist**

   3. 哈希表:

      1. 所有字符串小于64字节且键值对数量小于512个,  使用**ziplist**
      2. 否则使用**hashtable**

   4. 集合:

      1. 当所有元素都为整数且元素数量小于512时, 使用**intset**
      2. 否则使用**hashtable**

   5. 有序集合:

      1. 所有成员长度小于64且元素集合小于128用**ziplist**
      2. 否则使用**skiplist**

      注意: **当使用skiplist时,  还会同时用到一个字典, 用字典是为了在O(1)时间内获得score, 两个数据结构用指针共享数据, 不会造成内存浪费 **
   
8. #### RDB持久化:

   1. 生成一个经过压缩的二进制文件, 可以用它还原生成RDB文件时的状态
   2. 保存用save/bgsave, bgsave会用子进程来生成rdb文件, 不会阻塞, 而save则会阻塞
   3. 启动时自动载入rdb文件,  载入时会阻塞, 如果开启了AOF功能, 则优先用AOF来还原.
   4. 自动间隔保存, save time n表示在一段时间time内, 若进行了至少n次修改, 则执行bgsave

9. #### AOF持久化:

   1. 以redis命令形式保存
   2. 命令追加到缓冲区, 在每个事件循环结束之前, 会判断是否需要进行fsync, 是否需要fsync由appendfsync决定(默认为everysec)
      - always: 将aof_buf内容全部写入并sync到AOF文件(最安全可靠, 但是最慢)
      - everysec: 将aof_buf写入到aof文件, 如果距上次fsync超过一秒时进行fsync(足够快, 最多丢失一秒的命令)
      - no: 只写不fsync, 由操作系统决定何时进行fsync(效率最快, 但是会丢失较多的数据)
   3. 载入时, 创建一个不带网络链接的伪客户端(因为命令的执行来源于aof文件, 不关网络连接), 然后读取aof中的一条写命令, 使用伪客户端执行
   4. 但aof文件太大时, 会新开一个子进程进行AOF重写(bgrewriteaof),  新建一个新的aof文件, 并将一些命令进行合并(通过读取当前服务器状态来实现, 因为在重写aof文件时会丢失当前进来的命令, 于是每一条命令都会写入到两个缓冲区(AOF缓冲, AOF重写缓冲区), 在AOF重写完成后, 所有重写缓冲区的内容会被写入一个新文件, 然后将新AOF文件改名覆盖旧的AOF文件

10. ### 多机复制:

    1. 同步:

       **旧版复制:**

       过程:

       1. 从服务器向主服务器发送sync命令
       2. 主服务器收到sync后执行bgsave
       3. bgsave结束后发送给从服务器载入还原
       4. 主服务器还要将缓冲区里的写命令发给从服务器(传播执行bgsave时执行的命令)

       缺陷:

       1. 断线后需要恢复一小部分数据, 但是却要传输一整个rdb文件与一小部分命令, 需要fsync两次
       2. 浪费网络带宽
       3. 从服务器收到rdb时需要载入新的rdb文件, 这段时间无法处理请求

       **新版复制**

       过程:

       1. 用psync代替sync命令, 有部分重同步(用于断线复制)和完整重同步模式(用于初次复制
       2. 初次复制和旧版的一样
       3. 部分重同步只发送断开后主服务器执行的命令

       部分重同步实现:

       1. 主从服务器维护一个复制偏移量: 主服务器发送n个字节数据时, 复制偏移量加上n, 从服务器收到复制偏移量时加上n
       2. 复制积压缓冲区: 主服务器维护的一个固定的先进先出队列, 默认大小为1Mb, 在发送命令给从服务器时, 还要将命令放入队列, 并且还会记录相应的偏移量, 这样主服务器只要知道从服务器的偏移量就可以知道要发送多少数据给从服务器了(如果偏移量已经不在缓冲区, 则需要进行完整重同步)
       3. 服务器运行ID: 每个服务器都会生成一个40长度的随机字符串

       从服务器默认每秒一次向主服务器发送心跳以检查网络状态

11. ### sentinel 哨兵

    1. 监控（Monitoring）：哨兵会不断地检查主节点和从节点是否运作正常。

       1. 默认每10秒向主节点节点发生info请求, 主节点再发info获取从节点信息
       2. 每隔2s向redis数据节点的指定频道发送哨兵对主节点的主观判断, 每个哨兵都会订阅这个频道
       3. 每隔1s每个哨兵对主从节点和其他哨兵发送心跳, 判断节点是否正常
       4. 主观下线: 当一个哨兵在一段时间内收不到某个节点的心跳回应时, 可主观地判断这个节点下线了
       5. 客观判断: 当超过一半的哨兵认为这个节点下线了, 此时则为客观下线

    2. 自动故障转移（Automatic Failover）：当主节点不能正常工作时，哨兵会开始自动故障转移操作，它会将失效主节点的其中一个从节点升级为新的S主节点，并让其他从节点改为复制新的主节点。 **高可用**

       **哨兵通过选举一个leader来负责故障转移**

    3. 配置提供者（Configuration Provider）：客户端在初始化时，通过连接哨兵来获得当前Redis服务的主节点地址。

    4. 通知（Notification）：哨兵可以将故障转移的结果发送给客户端。
    
    实现: 
    
    1. 创建两个对主从服务器的异步连接, 一个用于命令交流, 一个用于订阅主服务器的__sentinel__:hello频道
    
    2. 通过监听服务器sentinel:hello频道来发现其他sentinel,  当一个sentinel向某个服务器的频道发送消息时, 所有sentinel都会收到.
    
       **sentinel之间是没有订阅连接的**