<!--
 * @Author: 黄遥
 * @Date: 2020-05-30 14:04:26
 * @LastEditors: 黄遥
 * @LastEditTime: 2020-05-30 15:14:05
 * @Description: file content
--> 
## HTTP1.0
1.0 的 HTTP 版本，是一种无状态、无连接的应用层协议。

1. 每次请求都需要建立 TCP 连接
2. 队头阻塞。由于 1.0 规定下一个请求必须在前一个请求响应到达之前才发送，假设前一个请求响应一直不到达，那么下一个请求就不发送，同样的后面的请求也给阻塞了

## HTTP1.1
1. 支持长连接，设置 Connection 字段为 keep-alive 可以保持 HTTP 连接不断开，避免每次重复建立释放建立 TCP 连接，如果客户端想关闭 HTTP 连接，可以在请求头中携带 connection: false 来告知服务器关闭请求
2. 管道化。允许请求“并行”传输，即我同时请求html和css，假如服务器的css资源先准备就绪，服务器也会先发送html再发送css，换句话来说，只有等到html响应的资源完全传输完毕后，css响应的资源才能开始传输。也就是说，不允许同时存在两个并行的响应。
3. 现阶段的浏览器厂商采取了另外一种做法，它允许我们打开多个TCP的会话，也就是说，我们平时在 chrome 上看到的并行下载资源，其实是不同的TCP连接上的HTTP请求和响应。这也就是我们所熟悉的浏览器对同域下并行加载6~8个资源的限制。而这，才是真正的并行！
4. 加入缓存处理（强缓存和协商缓存），新的字段如 cache-control，支持断点传输，以及增加了HOST字段（使得一个服务器能够用来创建多个Web站点）

## HTTP2.0
### 二进制分帧
HTTP2.0通过在应用层和传输层之间增加一个二进制分帧层，突破了HTTP1.1的性能限制、改进传输性能。
可见，虽然HTTP2.0协议和HTTP1.x协议之间的规范完全不同了，但是实际上HTTP2.0并没有改变HTTP1.x的语义
简单来说，HTTP2.0只是把原来HTTP1.x的header和body部分用frame重新封装了一层而已
### 多路复用（连接共享）
- 流（stream）：已建立连接上的双向字节流
- 消息：与逻辑消息对应的完整的一系列数据帧
- 帧（frame）：HTTP2.0 通信的最小单位，每个帧包含帧头部，标识出当前帧所属的流（stream id）
所有的HTTP2.0通信都在一个TCP连接上完成，这个连接可以承载任意数量的双向数据流。
每个数据流以消息的形式发送，而消息由一或多个帧组成。这些帧可以乱序发送，然后再根据每个帧头部的流标识符（stream id）重新组装。
举个例子，每个请求是一个数据流，数据流以消息的方式发送，而消息又分为多个帧，帧头部记录着 stream id 用来标识所属的数据流，不同属的帧可以在连接中随机混杂在一起。接收方可以根据 stream id 将帧再归属到各自不同的请求当中去。

另外，多路复用（连接共享）可能会导致关键请求被阻塞。HTTP2.0里每个数据流都可以设置优先级和依赖，优先级高的数据流会被服务器优先处理和返回给客户端，数据流还可以依赖其他的子数据流。

可见，HTTP2.0实现了真正的并行传输，它能够在一个TCP上进行任意数量HTTP请求。而这个强大的功能则是基于“二进制分帧”的特性。
### 头部压缩
在HTTP1.x中，头部元数据都是以纯文本的形式发送的，通常会给每个请求增加500~800字节的负荷。

比如说cookie，默认情况下，浏览器会在每次请求的时候，把cookie附在header上面发送给服务器。（由于cookie比较大且每次都重复发送，一般不存储信息，只是用来做状态记录和身份认证）

HTTP2.0使用encoder来减少需要传输的header大小，通讯双方各自cache一份header fields表，既避免了重复header的传输，又减小了需要传输的大小。高效的压缩算法可以很大的压缩header，减少发送包的数量从而降低延迟。
### 服务器推送
服务器除了对最初请求的响应外，服务器还可以额外的向客户端推送资源，而无需客户端明确的请求。

## HTTP1.1的合并请求是否适用于HTTP2.0
首先，答案是“没有必要”。之所以没有必要，是因为这跟HTTP2.0的头部压缩有很大的关系。

在头部压缩技术中，客户端和服务器均会维护两份相同的静态字典和动态字典。

在静态字典中，包含了常见的头部名称以及头部名称与值的组合。静态字典在首次请求时就可以使用。那么现在头部的字段就可以被简写成静态字典中相应字段对应的index。

而动态字典跟连接的上下文相关，每个HTTP/2连接维护的动态字典是不尽相同的。动态字典可以在连接中不听的进行更新。

也就是说，原本完整的HTTP报文头部的键值对或字段，由于字典的存在，现在可以转换成索引index，在相应的端再进行查找还原，也就起到了压缩的作用。

所以，同一个连接上产生的请求和响应越多，动态字典累积得越全，头部压缩的效果也就越好，所以针对HTTP/2网站，最佳实践是不要合并资源。

另外，HTTP2.0多路复用使得请求可以并行传输，而HTTP1.1合并请求的一个原因也是为了防止过多的HTTP请求带来的阻塞问题。而现在HTTP2.0已经能够并行传输了，所以合并请求也就没有必要了。

## 总结
HTTP1.0

无状态、无连接
HTTP1.1

持久连接
请求管道化
增加缓存处理（新的字段如cache-control）
增加Host字段、支持断点传输等
HTTP2.0

二进制分帧
多路复用（或连接共享）
头部压缩
服务器推送

https://segmentfault.com/a/1190000013028798?utm_source=tag-newest