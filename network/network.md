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

