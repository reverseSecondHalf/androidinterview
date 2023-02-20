# 卡顿监控的几个方式
卡顿的线上监控主要涉及两个问题：
1. 怎么判断触发卡顿
2. 是什么导致了卡顿

参考：
https://juejin.cn/post/6844903949560971277
## 利用Looper的消息队列（如[BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor)）
Looper轮循的时候，每次从消息队列取出一条消息，如果logging不为空，就会调用 logging.println，我们可以通过设置Printer，计算Looper两次获取消息的时间差，如果时间太长就说明Handler处理时间过长，直接把堆栈信息打印出来，就可以定位到耗时代码。
### 缺点
1. 不过println 方法参数涉及到字符串拼接，考虑性能问题，所以这种方式只推荐在Debug模式下使用
2. 堆栈有可能不是发生时的
3. mLogging.println 之前的耗时无法统计到
```

public void setMessageLogging(Printer printer) {
    mLogging = printer;
}

public static void loop() {
    final Looper me = myLooper();


    ...
    for (;;) {
        Message msg = queue.next(); // might block
        ...
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }
        ... // dispatch Message
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        ...
        msg.recycleUnchecked();
    }
}
```
## 插入空消息到消息队列
通过一个监控线程，每隔1秒向主线程消息队列的头部插入一条空消息。假设1秒后这个消息并没有被主线程消费掉，说明阻塞消息运行的时间在0～1秒之间。换句话说，如果我们需要监控3秒卡顿，那在第4次轮询中，头部消息依然没有被消费的话，就可以确定主线程出现了一次3秒以上的卡顿。
发送间隔建议不要设置太小，因为监控线程和主线程处理空消息都会带来一些性能损耗，但基本影响不大。

## 方法插桩（如[TraceCanary](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary))
在每个要监控的方法的入口和出口分别加上methodStart和methodEnd两个方法，类似插桩埋点。
### 缺点
1. 无法监控系统方法
2. apk体积会增大（每个方法都多了代码）
### 优化
1. 过滤简单的方法
2. 只需要监控主线程执行的方法

注：TraceCanary的方法上报耗时触发时机是通过Choreographer的doFrame函数判断两次调用之间的时间差，大于多少则认为卡顿，从而触发上报
TraceCanary的缺点：
1. 需要常驻8MB常驻内存记录方法耗时信息，对低端机还是有一定影响
2. 由于每次投递8MB太大，所以对堆栈过深的做了裁剪，影响卡顿原因的定位

## 对TraceCanary的改进
1. setMessageLogging其他人也调用的问题，通过编译期发现，并替换为我们自己的实现，然后保存，每次pritln时先调用保存的，再调用我们自己的逻辑。
2. 

## [Profilo](https://github.com/facebookincubator/profilo)

# 卡顿线下分析
参考：https://juejin.cn/post/6844904062610046990#heading-32
