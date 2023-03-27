# 1. 点击icon到activity显示发生了哪些
参考：https://juejin.cn/post/6844903959581163528  
https://juejin.cn/post/7179901192229617724
https://juejin.cn/post/6844903959589552142
## 1.1. Launcher进程请求AMS
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8c05d64c40a~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
至少android12开始类名稍微有所不同：
![image](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202212211646368.png)

## 1.2. AMS发送创建应用进程请求
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8c04147e855~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
ActivityTaskSupervisor的startSpecificActivity函数里会判断进程是否存在，是的话则调用当前类的realStartActivityLocked(这块兜兜转转最终会走到ActivityThread的handleLaunchActivity)，否则会调用ActivityTaskManagerService的startProcessAsync来创建进程。  
startProcessAsync会走到ActivityManagerService的startProcess，最终会走到Process的start函数，这里会向Zygote进程发送创建进程的请求。
## 1.3. Zygote进程接受请求并孵化应用进程
zygote通过一个死循环一直监听socket连接，有的话通过fork创建一个新进程。（为什么不用binder通信而是用socket：  
1. zygote和servicemanager的初始化时机问题，zygote需要使用binder时servicemanager不一定能保证初始化好了；
2. Linux中，fork进程其实并不是完美的fork，linux设计之初只考虑到了主线程的fork，也就是说如果主进程中存在子线程，那么fork进程中，其子线程的锁状态，挂起状态等等都是不可恢复的，只有主进程的才可以恢复。而binder作为典型的CS模式，其在Server是通过线程来实现的，Server等待请求状态时，必然是处于一种挂起的状态。所以如果使用binder机制，zygote进程fork子进程后，子进程的binder的Server永远处于一种挂起不可恢复的状态，这样的设计无疑是非常差的。所以，zygote如果使用binder，会导致子进程中binder线程的挂起和锁状态不可恢复，这是原因之二。 参考：https://blog.csdn.net/rzleilei/article/details/125770598?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-125770598-blog-106444099.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-125770598-blog-106444099.pc_relevant_default&utm_relevant_index=1)
## 1.4. 应用进程启动ActivityThread
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8c06d05ab45~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
通过反射调用到ActivityThread的main函数。
## 1.5. 应用进程绑定到AMS
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8e5b89da8cb~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
ActivityThread的main函数会调用ActivityManagerService的attachApplication方法，这里最终会调用到android.app.ActivityThread.ApplicationThread的bindApplication，
这里会进行providers的安装和应用Application的onCreate调用。
## 1.6. AMS发送启动Activity的请求
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8e5b8b0ec4d~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
现在当前进程已经绑定到了AMS，绑定后呢？回忆一下，上面对AMS的attachApplicationLocked方法分析时，重点提到了两个方法，其中ApplicationThread的bindApplication方法我们分析应用进程是如何绑定到AMS的，没错！另外一个方法mStackSupervisor.attachApplicationLocked(app)就是用来启动Activity的，现在让我们来看看。  

在ActivityThread的sendMessage中会把启动Activity的消息发送给mH,而mH为H类型，其实就是ActivityThread的Handler。到这里AMS就将启动Activity的请求发送给了ActivityThread的Handler。
## 1.7. ActivityThread的Handler处理启动Activity的请求
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8e5b8cbe5ce~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)

Handler接受到AMS发来的EXECUTE_TRANSACTION消息后，将调用TransactionExecutor.execute方法来切换Activity状态，先是会调用LaunchActivityItem里的execute方法，这里又会调用到ActivityThread的handleLaunchActivity以及接下来的performLaunchActivity方法，这个方法里会调用activity的attach，然后走onCreate方法。

从上面分析中我们知道onCreate已经被调用，且生命周期的状态是ON_CREATE，故executeCallbacks已经执行完毕，所以开始执行executeLifecycleState方法。如下

frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java

反正接下来就会走onStart和onResume。细节不表了。

## 另外一个简述的版本
1、Launcher被调用点击事件，转到Instrumentation类的startActivity方法。
2、Instrumentation通过AIDL方式使用Binder机制告诉ATMS要启动应用的需求。
3、ATMS收到需求后，反馈Launcher，让Launcher进入Paused状态
4、Launcher进入Paused状态，ATMS将创建进程的任务交给AMS，AMS通过socket与Zygote通信，告知Zygote需要新建进程。
5、Zygote fork进程，并调用ActivityThread的main方法，也就是app的入口。
6、ActivityThread的main方法新建了ActivityThread实例，并新建了Looper实例，开始loop循环。
同时ActivityThread也告知AMS，进程创建完毕，开始创建Application，Provider，并调用Applicaiton的attach，onCreate方法。
7、最后就是创建上下文，通过类加载器加载Activity，调用Activity的onCreate方法。

作者：Coolbreeze
链接：https://juejin.cn/post/7017798180347576350
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
# 2. 几个进程的介绍
init进程：init是所有linux程序的起点，是Zygote的父进程。解析init.rc孵化出Zygote进程。

Zygote进程：Zygote是所有Java进程的父进程，所有的App进程都是由Zygote进程fork生成的。

SystemServer进程：System Server是Zygote孵化的第一个进程。SystemServer负责启动和管理整个Java framework，包含AMS，PMS等服务。

Launcher：Zygote进程孵化的第一个App进程是Launcher。

1.init进程是什么？

Android是基于linux系统的，手机开机之后，linux内核进行加载。加载完成之后会启动init进程。

init进程会启动ServiceManager，孵化一些守护进程，并解析init.rc孵化Zygote进程。

2.Zygote进程是什么？

所有的App进程都是由Zygote进程fork生成的，包括SystemServer进程。Zygote初始化后，会注册一个等待接受消息的socket，OS层会采用socket进行IPC通信。

3.为什么是Zygote来孵化进程，而不是新建进程呢？

每个应用程序都是运行在各自的Dalvik虚拟机中，应用程序每次运行都要重新初始化和启动虚拟机，这个过程会耗费很长时间。Zygote会把已经运行的虚拟机的代码和内存信息共享，起到一个预加载资源和类的作用，从而缩短启动时间。