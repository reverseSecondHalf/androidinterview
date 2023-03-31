# hashmap原理
TODO
# java线程池ThreadPoolExecutor的参数有哪些，分别是啥用途
问这个应该是做了线程池的优化。 首先得了解下ThreadPoolExecutor的基本原理。

![image](https://pic1.zhimg.com/80/v2-ce5945685cf0a78806bae5362778a1d4_1440w.jpg)

当提交任务后，线程池会首先检查当前线程数，如果此时线程数小于核心线程数，比如最开始线程数量为 0，则新建线程并执行任务，随着任务的不断增加，线程数会逐渐增加并达到核心线程数，此时如果仍有任务被不断提交，就会被放入workqueue任务队列中,等待核心线程执行完当前任务后重新从 workQueue 中提取正在等待被执行的任务。


此时，假设我们的任务特别的多，已经达到了 workQueue 的容量上限，这时线程池就会启动后备力量，也就是 maximumPoolSize 最大线程数，线程池会在 corePoolSize 核心线程数的基础上继续创建线程来执行任务，假设任务被不断提交，线程池会持续创建线程直到线程数达到 maximumPoolSize 最大线程数，如果依然有任务被提交，这就超过了线程池的最大处理能力，这个时候线程池就会拒绝这些任务 


corePoolSize：线程池中常驻的线程个数，如果allowCoreThreadTimeOut置位true的话则不会常驻
max  
maximumPoolSize：池子中最大的线程个数，如果核心线程数达到了并且workQueue满了， 这时就会新开线程，但和核心线程加起来不超过maximumPoolSize。
keepAliveTime ： 超过corePoolSize时，空闲线程能存活的最大时间  
workQueue ： 等待队列, 当核心线程都在运行时，任务会放到这个队列里。    
RejectedExecutionHandler ： 处理拒绝的任务策略

# threadlocal原理
TODO
# synchronize是不是可重入锁
是。可重入锁指的是同一个线程不用释放自己已拿到的某个锁，还可以重复的获取这个锁n次，只是在释放的时候，也需要相应的释放n次。synchronize和reentrantlock都是可重入锁，synchronize不用手动释放。
# Atomic变量的实现原理
基于CPU的CAS来实现的。  
原子类的问题：ABA问题，解决：可以考虑使用AtomicStampedReference和AtomicMarkableReference类