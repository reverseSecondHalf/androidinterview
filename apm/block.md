# 1. 卡顿监控的几个方式
卡顿的线上监控主要涉及两个问题：
1. 怎么判断触发卡顿
2. 是什么导致了卡顿
第一个问题：可以使用Looper消息队列的日志、插入空消息到队列、 Choreographer的doFrame方式判断两帧间隔。
第二个问题：在Looper消息队列日志期间抓取堆栈或者上报方法耗时。

参考：
https://juejin.cn/post/6844903949560971277
## 1.1. 利用Looper的消息队列（如[BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor)）
Looper轮循的时候，每次从消息队列取出一条消息，如果logging不为空，就会调用 logging.println，我们可以通过设置Printer，计算Looper两次获取消息的时间差，如果时间太长就说明Handler处理时间过长，直接把堆栈信息打印出来，就可以定位到耗时代码。
### 1.1.1. 缺点
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
## 1.2. 插入空消息到消息队列
通过一个监控线程，每隔1秒向主线程消息队列的头部插入一条空消息。假设1秒后这个消息并没有被主线程消费掉，说明阻塞消息运行的时间在0～1秒之间。换句话说，如果我们需要监控3秒卡顿，那在第4次轮询中，头部消息依然没有被消费的话，就可以确定主线程出现了一次3秒以上的卡顿。
发送间隔建议不要设置太小，因为监控线程和主线程处理空消息都会带来一些性能损耗，但基本影响不大。

## 1.3. 方法插桩（如[TraceCanary](https://github.com/Tencent/matrix/wiki/Matrix-Android-TraceCanary))
在每个要监控的方法的入口和出口分别加上methodStart和methodEnd两个方法，类似插桩埋点。
### 1.3.1. 缺点
1. 无法监控系统方法
2. apk体积会增大（每个方法都多了代码）
### 1.3.2. 优化
1. 过滤简单的方法
2. 只需要监控主线程执行的方法

注：TraceCanary的方法上报耗时触发时机是通过Choreographer的doFrame函数判断两次调用之间的时间差，大于多少则认为卡顿，从而触发上报
TraceCanary的缺点：
1. 需要常驻8MB常驻内存记录方法耗时信息，对低端机还是有一定影响
2. 由于每次投递8MB太大，所以对堆栈过深的做了裁剪，影响卡顿原因的定位

## 1.4. 对TraceCanary的改进
1. setMessageLogging其他人也调用的问题，通过编译期发现，并替换为我们自己的实现，然后保存，每次pritln时先调用保存的，再调用我们自己的逻辑。
2. 

## 1.5. [Profilo](https://github.com/facebookincubator/profilo)
facebook出品，可以监测release包，用的黑科技较多，不适合线上

# 2. 卡顿线下分析
参考：https://juejin.cn/post/6844904062610046990#heading-32

## 2.1. CPU Profiler
优势：

图形的形式展示执行时间、调用栈等。
信息全面，包含所有线程。
劣势：

运行时开销严重，整体都会变慢，可能会带偏我们的优化方向。

使用方式：
Debug.startMethodTracing("");
// 需要检测的代码片段
...
Debug.stopMethodTracing();
复制代码
最终生成的生成文件在sd卡：Android/data/packagename/files。

## 2.2. Systrace
systrace 利用了 Linux 的ftrace调试工具（ftrace是用于了解Linux内核内部运行情况的调试工具），相当于在系统各个关键位置都添加了一些性能探针，也就是在代码里加了一些性能监控的埋点。Android 在 ftrace 的基础上封装了atrace，并增加了更多特有的探针，比如Graphics、Activity Manager、Dalvik VM、System Server 等等。
下面我们来简单回顾一下Systrace。
作用：
监控和跟踪API调用、线程运行情况，生成HTML报告。
优势：

1、轻量级，开销小。
2、它能够直观地反映CPU的利用率。
3、右侧的Alerts能够根据我们应用的问题给出具体的建议，比如说，它会告诉我们App界面的绘制比较慢或者GC比较频繁。

## 2.3. [perfetto](https://perfetto.dev/docs/)
google官方工具，可以认为是systrace的升级版。可参见：https://juejin.cn/post/6974377095359266847

## 2.4. [Simpleperf](https://android.googlesource.com/platform/system/extras/+/master/simpleperf/doc/README.md)

Android Studio includes a graphical front end to Simpleperf, documented in Inspect CPU activity with CPU Profiler. Most users will prefer to use that instead of using Simpleperf directly.

Simpleperf is a native CPU profiling tool for Android. It can be used to profile both Android applications and native processes running on Android. It can profile both Java and C++ code on Android. The simpleperf executable can run on Android >=L, and Python scripts can be used on Android >= N.

Android 5.0 新增了Simpleperf性能分析工具，它利用 CPU 的性能监控单元（PMU）提供的硬件 perf 事件。使用 Simpleperf 可以看到所有的 Native 代码的耗时，有时候一些 Android 系统库的调用对分析问题有比较大的帮助，例如加载 dex、verify class 的耗时等。Simpleperf 同时封装了 systrace 的监控功能，通过 Android 几个版本的优化，现在 Simpleperf 比较友好地支持 Java 代码的性能分析。具体来说分几个阶段：第一个阶段：在 Android M 和以前，Simpleperf 不支持 Java 代码分析。第二个阶段：在 Android O 和以前，需要手动指定编译 OAT 文件。第三个阶段：在 Android P 和以后，无需做任何事情，Simpleperf 就可以支持 Java 代码分析。从这个过程可以看到 Google 还是比较看重这个功能，在 Android Studio 3.2 也在 Profiler 中直接支持 Simpleperf。

## 2.5. Nanoscope
Uber的，有性能损耗的instrument 工具，需要自己刷ROM。

# 3. 冻帧率
Choreographer的doFrame根据时间以及实际计算的frameCount判断是否是冻帧，一般可以配置为700毫秒。
触发时机比如可以设置为activity的onResume开始，onPause结束

# 4. FPS
原理同冻帧率。

# 5. 设备分级
根据设备硬件属性将设备分级，不同级别策略不同，比如低端机有些动画就不做了、gif就不播了。