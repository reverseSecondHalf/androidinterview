# Handler相关
## IdleHandler
https://juejin.cn/post/6844904068129751047
## MessageQueue什么时候阻塞
nativePollOnce的nextPollTimeoutMillis值表示超时等待时长，如果为-1则无限等待知道唤醒，为0则无需等待直接返回。 

队列为空，或者队首的消息执行时间还没到.
具体去仔细看看源码：android.os.MessageQueue#next
# 跨进程怎么超过1MB的大文件
1. 走文件
2. bundle.putBinder(https://blog.csdn.net/ylyg050518/article/details/97671874)
3. 低版本反射MemoryFile，高版本直接SharedMemory，原理同2
