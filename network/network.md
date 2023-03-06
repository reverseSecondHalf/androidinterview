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
# 网络请求可用性兜底方案
# DNS优化
DNS劫持  
TTL过长  
跨运营商慢  
LocalDNS获取失败  

