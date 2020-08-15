# HTTP

1. ### HTTP的特点

   1. 超文本传输协议 -- **H**yperText **T**ransfer **P**rotocol。
   2. 明文传输
   3. 面向报文
   4. 无状态
   5. 端口:80

2. ### URI(统一资源定位符):

   有以下两种形式

   1. URL(统一资源定位器):**它是一种具体的URI，即URL可以用来标识一个资源，而且还指明了如何locate这个资源。**

      格式:`<协议>://<用户名>:<密码>@主机:端口/路径?参数1&参数2#片断`

      + ①协议(或称为服务方式)
      + ②存有该资源的主机IP地址(有时也包括端口号)
      + ③主机资源的具体地址。如目录和文件名等

   2. URN(统一资源名): **统一资源命名，是通过名字来标识资源，比如mailto:java-net@java.sun.com。**

3. ### HTTP状态码:

   ![](/home/halo/notes/http状态码.jpg)

- 100(Continue):  服务端已收到请求头, 客户端需要继续发送请求体

- 101( Switching Protocols):  切换协议。服务器根据客户端的请求切换协议。只能切换到更高级的协议

- 200: OK

- 201(Created): 已创建。成功请求并创建了新的资源, 且其URI在响应的Location字段中.

- 202(Accepted): 已经接受请求, 但未处理

- 203(Non-Authoritative Information): 响应被代理服务器修改, 非权威的消息

- 204(No Content): 已经成功处理请求, 但是没有返回任何内容

- 206(Partial Content): 表明此内容是range指定的资源的一部分

- 301(Moved Permanently)：永久性重定向, 新的URI在Location字段中(可缓存的)

- 302(Found)：临时重定向, 客户端应当继续向原有地址发送以后的请求。只有在Cache-Control或Expires中进行了指定的情况下，这个响应才是可缓存的。**就是請求的資源暫時駐留在不同的URI下(在Location字段里)**

- 303(See Other): 用于在收到HTTP Post后, 进行URL重定向GET的操作(因为POST没有幂等性)

- 304(Not Modified):  表示资源在由请求头中的If-Modified-Since或If-None-Match参数指定的这一版本之后，未曾被修改。在这种情况下，由于客户端仍然具有以前下载的副本，因此不需要重新传输资源。

- 400(Bad Request)：请求报文错误(语法错误, 请求太大,)，服务器无法识别

- 401(Unauthorized): 未认证, 返回时会在WWWAuthenticate首部附带用户信息的请求, 当初次收到此请求时, 会弹出认证窗口

- 403(Forbidden):  服务器已经理解请求, 但是拒绝执行, 禁止访问

- 404(Not found): 找不到资源

- 405(Method Not Allowed): 某个请求所针对的资源不支持对应的请求方法, 并在Allow字段中附上可以获取此URI的请求方式

- 500: 服务器内部错误

- 501(Not Implement): 服务器不认识或者不支持对应的请求方法

- 502:  Bad Gateway, 作为[网关](https://zh.wikipedia.org/wiki/网关)或者[代理](https://zh.wikipedia.org/wiki/代理服务器)工作的服务器尝试执行请求时，从上游服务器接收到无效的响应

- 503: 服务器忙(临时维护或者过载), 若服务器知道什么时候可以恢复正常, 则可以在响应头加入retry-after告诉客户端什么时候再重试

  

4. ### 报文格式:

   <img src="/home/halo/notes/http报文头.jpg" style="zoom:150%;" />

   - 请求:  
     1. 起始行: **方法   \s  路径URL  \s  协议版本 **
     
     2. 首部字段:  可以有0个或多个首部, 以键值对形式, 每个键值对以**CRLF(空行)**结束.
     
     3. 请求体: 首部与请求体之间以CRLF隔开
     
        

   ![](/home/halo/notes/响应报文.png)

   

   - 响应:
     1. 起始行: 版本  \s  响应状态码  \s  描述信息
     2. 首部字段: 同请求首部字段一样
     3. 响应体

5. ### 请求方法:

   1. GET: 用于请求服务器的某个资源
   2. POST: 用于向服务器传输数据
   3. HEAD: 与get类似, 不过服务器只返回首部
   4. PUT: 将文件传输给服务器, 并存入对应的URI位置
   5. TRACE: 用于在服务器发起环回诊断
   6. OPTIONS: 返回服务器支持的服务或HTTP方法
   7. DELETE: 删除服务器的某个资源

6. POST与GET的区别:

   1. GET用于获取资源(幂等), POST用于创建/更新资源(非幂等)
   2. **GET产生一个TCP数据包；POST产生两个TCP数据包。 ** POST的第一个数据包先发送header, 服务器响应100, 然后再发送数据.
   3. GET产生的URL地址可以被记录，而POST 不可以。
   4. **GET请求会被浏览器主动cache，而POST不会，除非手动设置。**
   5. GET请求只能进行url编码，而POST支持多种编码方式。
   6. GET请求参数会被完整保留在浏览器历史记录里，而POST中的参数不会被保留。
   7. GET请求在URL中传送的参数是有长度限制的，而POST没有。
   8. 对参数的数据类型，GET只接受ASCII字符，而POST没有限制
   9. GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息.
   10. **GET参数通过URL传递，POST放在Request body中。**

7. 常见首部字段:

   1. 通用首部:

      + Date: 报文日期
      + Connection：连接的管理
   + Cache-Control：缓存的控制
      + Transfer-Encoding：报文主体的传输编码方式

   2. **请求首部字段（请求报文会使用的首部字段）**:

      1. Content-Type: 报文主体的对象类型
      2. Accept: 用户代理可处理的媒体类型
      3. Authorization: Web认证信息
      4. If-Modified-Since/ if-Unmodified-Since: 比较资源的更新时间
      5. If-Match / If-None-Match: 比较实体标记(Etag)
      6. Referer: 从哪个页面进行请求
      7. User-Agent: http客户端的信息
      8. **if-range, range: if-range中指定了某个资源的etag, 如果服务器找到了, 则发送此资源由range指定的byte范围(用于断点续传)**

   3. 响应首部:

      1. Location: 用于重定向
      2. Proxy-Authenticate: 代理服务器对客户端的认证信息
      3. Server: HTTP服务器的安装信息

   4. 实体字段(用于补充说明实体的信息):

      1. Allow: 资源支持的HTTP方法
      2. Content-Encoding:资源编码类型(gzip/deflate/compress/identity)
      3. Content-Length: 实体大小, 单位字节
      4. Content-Type:表示实体类型(text/html)
      5. Content-Range: 范围请求(range)指定的字节范围
      6. Expires: 实体过期的日期 

8. HTTP 1.1 新特性

   + 缓存处理(cache-control)
   + 带宽优化及网络连接的使用(keep-alive)
   + 错误通知的管理
   + 消息在网络中的发送
   + 互联网地址的维护(host)
   + 安全性及完整性

   1. keep-alive持久连接(1.1默认特性):

      - 每次http请求都会进行一次TCP握手,  同时一个网页除了页面的HTML之外还会有很多静态资源以及诸多的API调用，如果每个请求都一个连接，势必网页的一次加载就会和服务器创建多次连接，这是非常浪费服务器资源的，同时也让客户端的访问速度慢了不少。
      - 持久连接也不宜一直保持，毕竟每个连接都会占用服务器资源，如果打开网页的人太多，那服务器资源也会紧张，所以一般服务器都会配置一个KeepAlive Timeout参数和KeepAlive Requests参数限制单个连接持续时长和最多服务的请求次数。
      - 由请求字段Connection控制, 当Connection:close时, 为短链接.
      - 持久连接还需要一个Content-Length来表明响应体的长度, 以此来让浏览器判断HTTP响应是否已经结束

   2. 断点续传:

      发送请求时利用range字段将资源字节区间请求, 响应时用content-range告诉客户端发送的字节范围. 资源tag用if-range字段指定.
      
   3. 分块传输:

      1. 工作在长连接状态, 需要在Transfer-Encoding中指定传输方式为chunked
      2. **chunked表示输出内容长度不确定, 所以用于动态内容的传输, 否则请用content-length和range**
      3. 分块传输允许服务器在最后发送消息头字段, 对于头字段无法确定的内容十分有用, 比如消息需要摘要来签名, 得等消息完整后才能生成
      4. 服务器有时用gzip/deflate来进行压缩以缩短传输时间, 这时文件可以用分块传输来一边压缩一边发送
      5. 每个非空的块包含数据的字节数, 最后一个块大小为0

   4. Pipeline管道化:

      HTTP1.0不支持管线化，同一个连接处理请求的顺序是逐个应答模式，处理一个请求就需要耗费一个TTL，也就是客户端到服务器的往返时间，处理N个请求就是N个TTL时长。当页面的请求非常多时，页面加载速度就会非常缓慢。

      可以同时将多个请求发送到服务器，然后逐个读取响应。这个管线化和Redis的管线化原理是一样的，响应的顺序必须和请求的顺序保持一致。

   5. 带宽优化:

      - 使用range/content-range字段指定每次发送的实体字节范围, 降低负载
      - 增加100响应码, 允许先发送头部, 再发送请求体

   6. Host域:

      支持一个服务器上的多个虚拟主机, 即多个url对应一个服务器ip

      

9. HTTP/2:

   1. 多路复用:

      浏览器对同一域名下的并发连接数量有限制，一般为6个, 而HTTP2的请求与响应以二进制帧的形式交错进行，只需建立一次连接，即一轮三次握手，实现多路复用。

   2. 压缩消息头:

      HTTP1的消息头很大冗余，而HTTP2.0利用HPACK对消息头进行压缩传输，将常用的消息头用索引(两端共同维护的静态表[存储常见头部名和头部]和动态表)表示, 即是将消息头中的不同的部分分别用不用的索引进行表示，且会用**哈夫曼编码压缩字符串**，最后封装成frame。索引表分为动态索引和静态索引，动态索引表在客户端和服务器端共同维护，静态索引采用硬编码形式。

   3.  HTTP2.0服务端推送

      HTTP2.0中服务器会主动将资源推送给客户端，例如把js和css文件主动推送给客户端而不用客户端解析HTML后请求再响应
      
   4. 二进制分帧

      在二进制分帧层中， HTTP/2 会将所有传输的信息分割为更小的消息和帧（frame）,并对它们采用二进制格式的编码 ，其中 HTTP1.x 的首部信息会被封装到 HEADER frame，而相应的 Request Body 则封装到 DATA frame 里面。将一个TCP连接分为若干个流（Stream），每个流中可以传输若干消息（Message），每个消息由若干最小的二进制帧（Frame）组成。

10. ## HTTPS:

    1.  要解决的问题:

       - HTTP明文传输, 不安全
       - 传输内容可能被篡改
       - 通信双方可能被伪装

    2. TLS/SSL层(安全传输层): 传输层安全, 位于会话层

    3. 端口: 443

    4. 加密方式: 混合加密, 用非对称加密传输密钥, 用对称加密传输内容

    5. 数字证书: 解决了双方的身份的验证问题, 客户端很少有交换证书. 获取步骤:

       1. 服务器向认证机构发送自己的公钥
       2. **认证机构用机构的私钥对服务器公钥进行数字签名生成了数字证书发送给服务器**
       3. 客户端有了证书后可以用证书机构的公钥来验证服务器公钥的真实性

    6. 握手:

       1. 客户端服务器相互发送hello报文, 交换ssl版本号,加密算法等
       2. 服务器发送数字证书与公钥(第一次握手结束)
       3. 客户端用证书机构的公钥对证书进行验证, 验证成功后生成随机密码串(Pre-master secret)同时用服务器公钥签名并发给服务器
       4. 最后客户端发送Finished报文, 这次报文包含全部报文的整体校验值, 服务器必须能解密此报文才能算握手成功.
       5. 之后的每次http报文都要用随机密码串进行加密
       6. 报文附加上MAC(Message Authentication code 消息认证码), 以检查报文有无被篡改

    7. 缺点:

       - 处理速度变慢, 因为每次传输都要加密, 消耗资源
       - 证书不便宜
       - 证书机构可能会被黑

       

11. ### Cookies:

    1. 用于管理服务端和客户端的状态, 存储在客户端

       1. Set-Cookie: 响应头字段,用于设置开始时cookie的状态

           - NAME=VALUE: 设置cookie的名称和值
           - expires=DATE: 过期时间
           - path=PATH:限制cookie发送的路径范围
           - domain=PATH: 限制cookie对应的域名
           - Secure: 当用HTTPs时才可以发送cookie
           - HttpOnly: 加以限制, 使cookie不能被js访问
           
       2. Cookie: 请求首部字段, 告诉服务器客户端的请求附带了cookie

    2. Session与cookie管理:

       ![](/home/halo/notes/session_cookie.png)

       1. **session存在服务器, cookie存在客户端, 而session机制依赖与cookie**

       2. 过程:

          1. 客户端把id, 密码放在请求体内并post
          2. 服务器生成一个session_id, 并在响应头的Set-Cookie中写入
          3. 客户端收到session_id将其作为cookie并保存起来, 下次发送请求的时候将cookie(附带有session_id)发送给服务器 

       3. 分布式session四种方案:

          1. 客户端存储, 将用户状态信息直接存储cookie上

             - 不安全
             - cookie存储大小限制
             - 网络开销大

          2. session复制: 搭建集群, 但任何一个服务器上session发生改变, 便将session的改变广播同步到所有服务器上

             - 容错性强
             - 网络负载大
             - 存储需求大

          3. session绑定: 

             利用nginx反向代理, 根据session将请求分布散列到某个服务器

             - 可用性低, 当某个服务器挂掉时, 这个session便丢失了
             - 配置简单

          4. session共享:

             用redis/memcached集群存储session

             - 高可用
             - 持久化

             
