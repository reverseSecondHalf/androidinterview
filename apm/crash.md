# 1. Crash治理策略
## 1.1. 开发阶段
提交代码review  
静态代码检查工具  
提倡使用Kotlin
## 1.2. 测试阶段
Monkey自动化测试
自动化篡改服务端返回值为空看客户端代码的健壮性
## 1.3. 灰度阶段
注意和线上版本对比，分析发现出哪些是新增的crash，对新增的尽量都解决，如果不能解决的需要评估上线后会带来多少的占比
## 1.4. 上线后阶段
刚上线后需要密切关注崩溃率变化。  
上线后一段时间（差不多大部分都升上来后），需要将线上的一些崩溃分配给指定开发，提前解决问题。
# 2. Crash治理工具
1. 线上监控系统
2. 第三方sdk修改工具
3. 代码插桩工具  
4. Java Crash兜底框架：通过配置下发解决线上可以被暂时忽略掉的crash或者无法解决的系统crash但可以进行兜底处理的问题。
# 3. Crash分析思路
## 3.1. 一般的业务crash
这类crash一般按堆栈直接修复即可，对于空指针类型的，需要注意下是否真的是简单的加个空指针判断就行了，是否是逻辑上有问题。
## 3.2. Android系统问题
这类问题一般是规避。比如：
1. android8.0透明activity如果设置了portrait属性直接崩溃，可以直接hook基类activity解决掉。
2. andoridP上开始的webview的setDataDirectorySuffix崩溃，需要在启动时为每个进程设置不同的路径。 
    
无法规避的走兜底策略catch住。
## 3.3. 堆栈无法定位的
三步走：
业务和系统日志 -> 缩小范围 -> 需要更多信息
### 3.3.1. 业务和系统日志
如果根据已有的信息，比如业务日志、系统日志、用户崩溃前的操作行为
可利用的信息如：日志、用户崩溃前的操作行为（activity、fragment级别），定位到问题并解决。
### 3.3.2. 缩小范围
根据现有可利用的信息，看看是否能缩小范围，比如缩小到某个业务甚至某个功能，又或者crash是否有某种分布趋势（如只在某个机型或者android版本上出现），根据这些我们再去查看这些业务最近有啥变更：比如开启了某个实验、新增了哪些代码，又或者是我们拿特定设备或机型尝试复现看看。

### 3.3.3. 需要更多信息
   如果以上还不能解决，看看是否是需要通过发版再增加一些额外信息来解决。比如遇到过一种空指针：
   ```
   03-01 11:50:11.836: E/AndroidRuntime(21136): FATAL EXCEPTION: main  03-01 11:50:11.836: E/AndroidRuntime(21136): java.lang.NullPointerException  03-01 11:50:11.836: E/AndroidRuntime(21136):    at android.view.ViewGroup.dispatchGetDisplayList(ViewGroup.java:3147)  03-01 11:50:11.836: E/AndroidRuntime(21136):    at android.view.View.getDisplayList(View.java:12646)  03-01 11:50:11.836: E/AndroidRuntime(21136):    at android.view.View.getDisplayList(View.java:12754)
   ```
   单从这个无法定位到是哪处代码的removeView移除导致了问题，如果通过排查代码也无法找到的话，只能通过插桩手段在所有调用removeView相关的操作处加上堆栈记录，然后在崩溃上报时把堆栈记录带上去，下次就能大概率定位到问题。

# 4. OOM治理
## 4.1. 堆内存OOM
到达了Runtime.getRuntime.MaxMemory()的上限。
1. 设置largeHeap
2. 内存泄漏治理
3. 内存抖动治理，因为会带来碎片化增加最终导致可分配内存减少
## 4.2. native内存OOM
1. so库的内存治理，主要是通过inline hook和plt hook来做监控，快发生OOM时做堆栈上报来解决
## 4.3. 图片导致的OOM
这个其实不属于一个单独的OOM分类，只是因为图片在3.0-7.1是在java堆上分配的，8.0及以后是在native上分配的，所以单独拎出来介绍。
1. 大图监控及治理
2. 低端机图片优化（比如图片从ARGB8888->RGB565、缓存变小)
3. 通过一些手段将8.0以下将图片分配在native上（比如fresco的匿名共享内存、以及一些[hook的方式](https://mp.weixin.qq.com/s/S-YJ72qW_amYgIkBSEnsGg)）
## 4.4. 虚拟内存OOM（一般是在32位机器上）
[快速缓解 32 位 Android 环境下虚拟内存地址空间不足的“黑科技”](http://androidos.net.cn/doc/2021/7/29/532.html)
### 4.4.1. 堆缩减（android5-7和8-11的方案不同）
具体可参考：阿里开源的[Patrons](https://github.com/alibaba/Patrons)库  
收益很大，几百MB起步。
### 4.4.2. 释放Webview预分配的内存
能带来100多MB
### 4.4.3. 线程栈空间减半
通过native的hook方式或者java层创建线程时传递一半的栈空间大小，来减少默认栈空间大小，减半能带来100多MB
## 4.5. 线程创建OOM
### 4.5.1. 降低线程数量
1. 为 maxPoolSize 设置上限

有些开发者或者第三方库设置的线程池 maxPoolSize 通常是 2 * NCPU + 1 或者 2 * NCPU，当有多个模块都这样使用的时候，就容易造成某一时刻出现大量的线程，尤其是 CachedThreadPoolExecutor，通过控制单个线程池的 maxPoolSize 的上限，可以将某一时刻，所有线程池造成的叠加效应降到尽可能低的水平。

2. 允许核心线程在空闲时自动销毁
executor.allowCoreThreadTimeOut(true)
### 4.5.2. FD创建OOM
通过监控来定位解决。

参考：
https://juejin.cn/post/7044165242565165086#heading-5
https://juejin.cn/post/6948397601515372557#heading-3