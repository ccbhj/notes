# RabbitMQ

1. ### 简介:

   RabbitMQ 是采用 Erlang 语言实现 AMQP(Advanced Message Queuing Protocol，高级消息队列协议）的消息中间件，它最初起源于金融系统，用于在分布式系统中存储转发消息。

   **消息队列可以用于:**

   - 流量削峰
   - 降低系统耦合度, 跨语言, 跨应用
   - 消息驱动架构
   - 异步通信

2. ### 特点:

   1. **可靠性：** RabbitMQ使用一些机制来保证消息的可靠性，如持久化、传输确认及发布确认等。
   2. **灵活的路由：** 在消息进入队列之前，通过交换器来路由消息。对于典型的路由功能，RabbitMQ 己经提供了一些内置的交换器来实现。针对更复杂的路由功能，可以将多个交换器绑定在一起，也可以通过插件机制来实现自己的交换器。这个后面会在我们将 RabbitMQ 核心概念的时候详细介绍到。
   3. **扩展性：** 多个RabbitMQ节点可以组成一个集群，也可以根据实际业务情况动态地扩展集群中节点。
   4. **高可用性：** 队列可以在集群中的机器上设置镜像，使得在部分节点出现问题的情况下队列仍然可用。
   5. **支持多种协议：** RabbitMQ 除了原生支持 AMQP 协议，还支持 STOMP、MQTT 等多种消息中间件协议。
   6. **多语言客户端：** RabbitMQ几乎支持所有常用语言，比如 Java、Python、Ruby、PHP、C#、JavaScript等。
   7. **易用的管理界面：** RabbitMQ提供了一个易用的用户界面，使得用户可以监控和管理消息、集群中的节点等。在安装 RabbitMQ 的时候会介绍到，安装好 RabbitMQ 就自带管理界面。
   8. **插件机制：** RabbitMQ 提供了许多插件，以实现从多方面进行扩展，当然也可以编写自己的插件。感觉这个有点类似 Dubbo 的 SPI机制

3. ### 核心概念:

   ![S](/home/halo/notes/rabbitmq.jpeg)

   1. Producer/Consumer

      生产消息一方/投递消息一方

   2. Channel（信道）：消息推送使用的通道。

   3. 消息: 一般由 2 部分组成：**消息头**（或者说是标签 Label）和 **消息体**。消息体也可以称为 payLoad ,消息体是不透明的，而消息头则由一系列的可选属性组成，这些属性包括 routing-key（路由键）、priority（相对于其他消息的优先权）、delivery-mode（指出该消息可能需要持久性存储), TTL(队列过期时间，在消息入队列开始计算时间，只要超过了队列的超时时间配置，那么消息就会自动的清除)等。生产者把消息交由 RabbitMQ 后，RabbitMQ 会根据消息头把消息发送给感兴趣的 Consumer(消费者)。

   4. RoutingKey（路由键）：用于把生成者的数据分配到交换器上。

   5. BindingKey（绑定键）：用于把交换器的消息绑定到队列上。

   6. vhost: 可以理解为虚拟 broker ，即 mini-RabbitMQ server。

      1. host本质上是一个mini版的RabbitMQ服务器，拥有自己的队列、绑定、交换器和权限控制；
      2. vhost通过在各个实例间提供逻辑上分离，允许你为不同应用程序安全保密地运行数据；
      3. vhost是AMQP概念的基础，必须在连接时进行指定，RabbitMQ包含了默认vhost：“/”；
      4. 当在RabbitMQ中创建一个用户时，用户通常会被指派给至少一个vhost，并且只能访问被指派vhost内的队列、交换器和绑定，vhost之间是绝对隔离的。

   7. Exchange(交换机):

      在 RabbitMQ 中，消息并不是直接被投递到 **Queue(消息队列)** 中的，中间还必须经过 **Exchange(交换器)** 这一层，**Exchange(交换器)** 会把我们的消息转发到对应的 **Queue(消息队列)** 中。

      + 生产者指定Message的routing key，并指定Message发送到哪个Exchange
      + Queue会通过binding key绑定到指定的Exchange
      + Exchange根据对比Message的routing key和Queue的binding key，然后按一定的分发路由规则，决定Message发送到哪个Queue

      四种exchange类型:

      - #####  fanout

        将消息发送到绑定到交换机的所有队列上, 所以 fanout 类型是所有的交换机类型里面速度最快的。

        **fanout 类型常用来广播消息。**

      - ##### direct

        把消息路由到那些Bindingkey (消息头中的)与 RoutingKey(队列的) 完全匹配的 Queue 中。

        **direct 类型常用在处理有优先级的任务，根据任务的优先级把消息发送到对应的队列，这样可以指派更多的资源去处理高优先级的队列。**

      - ##### topic

        前面讲到direct类型的交换器路由规则是完全匹配 BindingKey 和 RoutingKey ，但是这种严格的匹配方式在很多情况下不能满足实际业务的需求。topic类型的交换器在匹配规则上进行了扩展，它与 direct 类型的交换器相似，也是将消息路由到 BindingKey 和 RoutingKey 相匹配的队列中，但这里的匹配规则有些不同，它约定：

        + RoutingKey 为一个点号“．”分隔的字符串（被点号“．”分隔开的每一段独立的字符串称为一个单词），如 “com.rabbitmq.client”、“java.util.concurrent”、“com.hidden.client”;
        + BindingKey 和 RoutingKey 一样也是点号“．”分隔的字符串；
        + BindingKey 中可以存在两种特殊字符串“*”和“#”，用于做模糊匹配，其中“*”用于匹配一个单词，“#”用于匹配多个单词(可以是零个)。

      - ##### headers(不推荐)

        headers 类型的交换器不依赖于路由键的匹配规则来路由消息，而是根据发送的消息内容中的 headers 属性进行匹配。在绑定队列和交换器时制定一组键值对，当发送消息到交换器时，RabbitMQ会获取到该消息的 headers（也是一个键值对的形式)'对比其中的键值对是否完全匹配队列和交换器绑定时指定的键值对，如果完全匹配则消息会路由到该队列，否则不会路由到该队列。headers 类型的交换器性能会很差，而且也不实用，基本上不会看到它的存在。

      生产者将消息发给交换器的时候，一般会指定一个 **RoutingKey(路由键)**，用来指定这个消息的路由规则，而这个 **RoutingKey 需要与交换器类型和绑定键(BindingKey)联合使用才能最终生效**。

      ![](/home/halo/notes/exchange_binding.jpeg)

      RabbitMQ 中通过 **Binding(绑定)** 将 **Exchange(交换器)** 与 **Queue(消息队列)** 关联起来，在绑定的时候一般会指定一个 **BindingKey(绑定建)** ,这样 RabbitMQ 就知道如何正确将消息路由到队列了

   8. Queue(消息队列):

      Queue(消息队列)** 用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。

      **RabbitMQ** 中消息只能存储在 **队列** 中，这一点和 **Kafka** 这种消息中间件相反。Kafka 将消息存储在 **topic（主题）** 这个逻辑层面，而相对应的队列逻辑只是topic实际存储文件中的位移标识。 RabbitMQ 的生产者生产消息并最终投递到队列中，消费者可以从队列中获取消息并消费。

      **多个消费者可以订阅同一个队列**，这时队列中的消息会被平均分摊（Round-Robin，即轮询）给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理，这样避免的消息被重复消费。

      **RabbitMQ 不支持队列层面的广播消费,如果有广播消费的需求，需要在其上进行二次开发,这样会很麻烦，不建议这样做。**

4. ### 消息是怎么发送的?

   首先客户端必须连接到 RabbitMQ 服务器才能发布和消费消息，客户端和 rabbit server 之间会创建一个 tcp 连接，一旦 tcp 打开并通过了认证（认证就是你发送给 rabbit 服务器的用户名和密码），你的客户端和 RabbitMQ 就创建了一条 amqp 信道（channel），信道是创建在“真实” tcp 上的虚拟连接，amqp 命令都是通过信道发送出去的，每个信道都会有一个唯一的 id，不论是发布消息，订阅队列都是通过这个信道完成的

5. ### rabbitmq 怎么保证消息的可靠投递？

   1. 可能在推送消息到broker时因为网络等问题投送失败, 导致消息丢失

      解决方法: 

      1. 事务模式: 在通过channel.txSelect方法开启事务之后，我们便可以发布消息给RabbitMQ了，如果事务提交成功，则消息一定 到达了RabbitMQ中，如果在事务提交执行之前由于RabbitMQ异常崩溃或者其他原因抛出异常，这个时候我们便可以将其捕获，进而通过执行channel.txRollback方法来实现事务回滚。使用事务机制的话会“吸干”RabbitMQ的性能，一般不建议使用
      2. confirm（确认）模式:生 产者通过调用channel.conﬁrmSelect方法（即Conﬁrm.Select命令）将信道设置为conﬁrm模式。一旦消息被投递到所有匹配的队列之后，RabbitMQ就会发送一个确认（Basic.Ack）给生产者（包含消息的唯一ID），这就使得生产者知晓消息已经正确到达了目的地了。**而发布者确认有三种编程方式**:
         1. 普通confirm模式：每发送一条消息后，调用waitForConfirms()方法，等待服务器端confirm。实际上是一种串行confirm了。
         2. 批量confirm模式：每发送一批消息后，调用waitForConfirms()方法，等待服务器端confirm。
         3. 异步confirm模式：提供一个回调方法，服务端confirm了一条或者多条消息后Client端会回调这个方法。(性能最高)

   2. 可能因为路由关键字错误，或者队列不存在，或者队列名称错误导致exchange无法路由到队列

      1. 使用mandatory参数和ReturnListener，可以实现消息无法路由的时候返回给生产者
      2. 另一种方式就是使用备份交换机（alternate-exchange），无法路由的消息会发送到这个交换机上。

   3. 可能因为系统宕机、重启、关闭等等情况导致存储在队列的消息丢失

      1. 队列持久化: durable = true

      ```text
      // channel.queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments) 
      channel.queueDeclare(QUEUE_NAME, true, false, false, null);	
      ```

      2. 交换机持久化:

         ```text
         // channel.exchangeDeclare(String exchange, boolean durable)
         channel.exchangeDeclare("MY_EXCHANGE","true");
         ```
         
        3. 消息持久化:
      
        ```text
          AMQP.BasicProperties properties = new AMQP.BasicProperties
      	.Builder() 
           // 2代表持久化，其他代表瞬态 
          .deliveryMode(2) 
          .build(); 
          channel.basicPublish("", QUEUE_NAME, properties, msg.getBytes());
        ```
      
    4. **确保消息从队列正确地投递到消费者**:

       如果消费者收到消息后未来得及处理即发生异常，或者处理过程中发生异常，会导致消费者无法收到消息.
       
       为了保证消息从队列可靠性到达消费者，**RabbitMQ提供了消息确认机制（message acknowledgement）,消费者在订阅队列时，可以指定autoAck参数**，当autoAck等于false时，RabbitMQ会等待消费者显示地回复确认消息才从队列中删除该消息。
   如果消息消费失败，也可以调用Basic.Reject或者BasicNack来拒绝当前消息而不是确认，如果requere参数为true，可以把这条消息重新存入队列，以便发送给下一个消费者。

   5. 消费者回调

      消费者处理消息之后，可以再发送一条消息给生产者，或者调用生产者地API，告知消息处理完毕。

   6. 补偿机制

      对于一定时间没有响应地消息，可以设置一个定时重发地机制，但是要控制次数，比如最多重复三次，否则会造成消息堆积。

6. ### **如何避免消息的重复消费？**:

   **消息重复消费可能会有两个原因：**

   1. 生产者的问题。生产者重复发送消息，比如在开启Confirm模式但未收到确认
   2. 消费者出了问题，由于消费者未发送ACK或者其它原因，消息重复投递

   **对于重复发送的消息，可以对每一条消息生成一个唯一的业务id，通过日志或者建表来做重复控制。**

   1. 唯一id+加指纹码，利用数据库主键去重。
      优点：实现简单
      缺点：高并发下有数据写入瓶颈。
   2. 利用Redis的原子性来实现。(setnx)
      使用Redis进行幂等是需要考虑的问题
      + 是否进行数据库落库，落库后数据和缓存如何做到保证幂等（Redis 和数据库如何同时成功同时失败）？
      + 如果不进行落库，都放在Redis中如何这是Redis和数据库的同步策略？还有放在缓存中就能百分之百的成功吗？

7. ### 要保证消息持久化成功的条件有哪些?

   1. 声明队列必须设置持久化 durable 设置为 true.
   2. 消息推送投递模式必须设置持久化，消息的deliveryMode 设置为 2（持久）。
   3. 消息已经到达持久化交换器。
   4. 消息已经到达持久化队列。

   以上四个条件都满足才能保证消息持久化成功。

8. ### **rabbitmq 持久化有什么缺点？**

   持久化的缺点就是降低了服务器的吞吐量，因为使用的是磁盘而非内存存储，从而降低了吞吐量。可尽量使用 ssd 硬盘来缓解吞吐量的问题。

9. ### **rabbitmq 怎么实现延迟消息队列？**

   1. 通过消息过期(消息头TTL)后进入死信交换器，再由交换器转发到延迟消费队列，实现延迟功能；

   2. 使用 RabbitMQ-delayed-message-exchange 插件实现延迟功能.

      **死信队列(DLX-dead letter exchange):利用DLX，当消息在一个队列中变成死信 `(dead message)` 之后，它能被重新publish到另一个Exchange，这个Exchange就是DLX, **消息变为死信队列的情况**:

      1. 消息被拒绝（basic.reject/basic.nack）同时requeue=false（不重回队列）
      2. TTL过期
      3. 队列达到最大长度

10. ### **rabbitmq 节点的类型有哪些？**

    磁盘节点：消息会存储到磁盘。

    内存节点：消息都存储在内存中，重启服务器消息丢失，性能高于磁盘类型。

11. ### 消费端限流:

    当队列中有上万条消息未消费, 一个消费者一连接就会有许多消息被推送过去, 导致消费者压力过大:

    + rabbitMQ提供了一种qos（服务质量保证）的功能，即非自动确认消息的前提下，如果有一定数目的消息（通过consumer或者Channel设置qos）未被确认，不进行新的消费。

    > void basicQOS(unit prefetchSize,ushort prefetchCount,Boolean global)方法。

    + prefetchSize:0 单条消息的大小限制。0就是不限制，一般都是不限制。
    + prefetchCount: 设置一个固定的值，告诉rabbitMQ不要同时给一个消费者推送多余N个消息，即一旦有N个消息还没有ack，则consumer将block掉，直到有消息ack
    + global：truefalse 是否将上面的设置用于channel，也是就是说上面设置的限制是用于channel级别的还是consumer的级别的。

12. ### 集群

    **集群模式下RabbitMQ不会默认会将消息冗余到所有节点**

<img src="/home/halo/notes/rabbitmq_cluster.jpg" style="zoom:150%;" />

三个节点组成了一个RabbitMQ的集群，Exchange A的元数据信息在所有节点上是一致的，**而Queue的完整数据则只会存在于它所创建的那个节点上，其他节点只知道这个queue的metadata信息和一个指向queue的owner node的指针。**

1. 数据同步:

   **集群元数据的同步**

   rabbitmq考虑存储空间、性能的原因，所以集群内部仅同步元数据

   RabbitMQ 内部有各种基础构件，包括队列、交换器、绑定、虚拟主机等，这些构件以元数据的形式存在

   > Queue元数据：队列的名称和声明队列时设置的属性(是否持久化、是否自动删除、队列所属的节点)
   > Exchange元数据：交换机的名称、类型、属性(是否持久化等)
   > Bindding元数据：一张简单的表格展示了如何将消息路由到队列。包含的列有 Exchange名称、Exchange类型、routing_key、queue_name等
   > vhost元数据：为vhost内队列、交换机和绑定提供命名空间和安全属性
2. 集群消息转发

例如当消息进入A节点的Queue中后，consumer却从B节点拉取时，集群内部做了什么？

> RabbitMQ会临时在A、B间进行消息传输，把A中的消息实体取出并经过B发送给consumer，所以consumer应平均连接每一个节点，从中取消息
>

3. 集群节点故障:

如果特定队列的所有者节点发生了故障，那么该节点上的队列和关联的绑定都会消失吗？

> 分两种情况，看集群内节点是内存模式还是磁盘模式：
> 如果是内存节点，那么附加在该节点上的队列和其关联的绑定都会丢失，并且消费者可以重新连接集群并重新创建队列；
> 如果是磁盘节点，重新恢复故障后，该队列又可以进行传输数据了，并且在恢复故障磁盘节点之前，不能在其它节点上让消费者重新连到集群并重新创建队列，如果消费者继续在其它节点上声明该队列，会得到一个 404 NOT_FOUND 错误，这样确保了当故障节点恢复后加入集群，该节点上的队列消息不回丢失，也避免了队列会在一个节点以上出现冗余的问题。

在单节点 RabbitMQ 上，仅允许该节点是磁盘节点，这样确保了节点发生故障或重启节点之后，所有关于系统的配置与元数据信息都会重磁盘上恢复；

而**在 RabbitMQ 集群上，要求集群里至少有一个磁盘节点，所有其他节点可以是内存节点，当节点加入或者离开集群时，必须要将该变更通知到至少一个磁盘节点。**

如果是唯一的磁盘节点也发生故障了，集群可以继续路由消息，但是不可以做以下操作了：

+ 创建队列
+ 创建交换器
+ 创建绑定
+ 添加用户
+ 更改权限
+ 添加或删除集群节点

