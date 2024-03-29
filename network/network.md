# 网络性能监控
# IPV6
# HttpDNS
# 网络优化
## 速度
网络好的时候怎么样利用好带宽提升
## 弱网
弱网时保持最大连通性
## 安全
防劫持、篡改、窃听
# OKHttp核心原理
# http协议的keep-alive和tcp的keepalive有啥区别  
http的是为了避免每次http请求都走三次握手四次挥手的过程，告诉服务器这次请求完毕了不要关闭连接，在超时之前继续复用。
tcp的也叫保活，是为了定期发送报文保持这个tcp链路是通畅的，避免由于nat等中间的网络设备在连接长时间没活动时给掐断了。
但是如果没有tcp的保活机制，http的keep-alive成功率会下降，因为这个链路可能断了。
参考：https://cloud.tencent.com/developer/news/696654  
https://xiaozhuanlan.com/topic/3758142906
# Http的队头阻塞
https://www.cnblogs.com/shangyueyue/p/11041998.html
https://dazzey.cn/2022/05/13/HTTP1-1-、2-、3版本中的队头阻塞/

https://zhuanlan.zhihu.com/p/488681676

图中第一种请求方式，就是单次发送request请求，收到response后再进行下一次请求，显示是很低效的。

于是http1.1提出了管线化(pipelining)技术，就是如图中第二中请求方式，一次性发送多个request请求。

然而pipelining在接收response返回时，也必须依顺序接收，如果前一个请求遇到了阻塞，后面的请求即使已经处理完毕了，仍然需要等待阻塞的请求处理完毕。这种情况就如图中第三种，第一个请求阻塞后，后面的请求都需要等待，这也就是队头阻塞(Head of line blocking)。

为了解决上述阻塞问题，http2中提出了多路复用(Multiplexing)技术，Multiplexing是通信和计算机网络领域的专业名词。http2中将多个请求复用同一个tcp链接中，将一个TCP连接分为若干个流（Stream），每个流中可以传输若干消息（Message），每个消息由若干最小的二进制帧（Frame）组成。也就是将每个request-response拆分为了细小的二进制帧Frame，这样即使一个请求被阻塞了，也不会影响其他请求，如图中第四种情况所示。

2015 年正式发布的 HTTP/2 默认不再使用 ASCII 编码传输，而是改为二进制数据，来提升传输效率。

# 网络请求可用性兜底方案
# DNS优化
DNS劫持  
TTL过长  
跨运营商慢  
LocalDNS获取失败  

# Http2相关
## http2对比http1
1. 使用了多路复用，解决了http1.1的队首阻塞（tcp阻塞并解决不了），通过多个Stream复用一条TCP连接，根据帧首部的流id重组请求
2. 使用了二进制而不是文本
3. 头部压缩（HPACK 算法）：http1.x的header由于cookie和user agent很容易膨胀，而且每次都要重复发送。当这种请求数很多的时候，会导致网络的吞吐率不高。并且，比较大的HTTP头部会迅速占满慢启动过程中的拥塞窗口，导致延迟加大。所以HTTP头的压缩显得很有必要
4. 服务端推送（Server Push）：HTTP的请求/应答方式的会话都是客户端发起的，缺乏服务器通知客户端的机制，在需要通知的场景，如聊天室，游戏，客户端应用需要不断地轮询服务器。
## 为什么http2使用二进制，对比http1的文本有啥优势
HTTP/2使用二进制编码（Binary Encoding）的协议格式，相比于HTTP/1.1的文本格式，具有以下几个优点：

更高的解析效率：二进制编码的格式可以更高效地解析和处理，因为它们的大小更小，解析速度更快。HTTP/2协议头使用二进制编码，可以更快地解析和处理请求和响应头。
更好的压缩效率：HTTP/2使用了HPACK压缩算法对HTTP头部进行压缩，可以显著地减少传输数据的大小，从而降低网络延迟和带宽消耗。相比之下，HTTP/1.1使用文本格式传输头部，压缩效率较低，容易导致传输过程中的延迟。
更好的多路复用性：HTTP/2的二进制编码可以更好地支持多路复用（Multiplexing），允许在单个TCP连接上同时处理多个请求和响应。这样可以有效地避免网络拥塞和延迟，提高传输效率和响应速度。相比之下，HTTP/1.1使用文本格式传输请求和响应，容易导致请求和响应的阻塞和延迟。
## 怎么判断keep alive是不是复用了
tls层会有个session id，可以判断是否复用了，通过charles的 TLS-> Session Resumed即可判断是否是复用了
## keep alive值设置为多少，怎么选择这个值的
## http2的缺点
1. 弱网时TCP层的队头阻塞：通过动态策略降级为1.1(最近10次里失败率对比2和1看谁高，另外如果2高的话还会60秒发一次keepalive，主要看看有没有好转，连续3次成功的话则好转)；弱网探测时走http1.1；限制一个http2连接的最大并发流的数目（注意：okhttp对http2的最大默认值是5，对http1.1的最大连接数也是5，但都可以自定义修改）

# TLS握手过程
TODO