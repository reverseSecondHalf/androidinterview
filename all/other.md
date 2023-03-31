# 事件分发机制
优先级：onTouchListener > onTouchEvent > onClickListener 

# handler是怎么做到从其它线程切换到主线程的
1. Looper：每个线程都有一个唯一的Looper（通过Looper里有一个静态的ThreadLocal<Looper> sThreadLocal保证），是个死循环，不断的从其mQueue里通过其next函数获取下一个Message，执行msg.target.dispatchMessage，也就是Handler的dispatchMessage。 需要注意的是，MessageQueue的next函数里会调用nativePollOnce，这里如果队列空的会释放CPU资源进入休眠状态。
2. Handler : new的时候默认使用当前线程的Looper，也可以传你自己的Looper。  
3. MessageQueue : 存储在Looper里，Handler里也拿了引用。是个单向链表，按Message.when自然序排，表头是 MessageQueue.mMessages。  
4. Message : 有3个重要属性target、callback和next。next是指向下一个message很好理解，至于target，最终会被设置为handler：在调用handler的sendMessage时，最终会走到MessageQueue的enqueueMessage，这里会把message的target置位this（注意异步消息的target为null）

从上面的过程可以看到，假如Handler是在A线程创建的，然后在线程B通过handler这个对象，调用其sendMessage函数，这里会拿到mQueue(mLooper.mQueue)，调用其enqueueMessage函数，而mLooper是在Handler创建的时候赋值的，也就是说mLooper是属于A线程所有，这样就从B线程切换到A线程了。所以实现线程切换的核心是ThreadLocal的Looper，handler只是把来自不同线程的消息塞到某个线程的Looper的队列里。

# binder原理,android为啥使用binder