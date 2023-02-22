# 1. 业界知名工具
## 1.1. [LeakCanary](https://github.com/square/leakcanary/)
仅用于线下监控使用，dump会冻结当前进程导致卡死一段时间。
2.6版本以前只检测activity和frament的泄漏，之后加入了ViewModel、RootView、Service的检测，用到了对WindowManagerGlobal的反射以及对系统ActivityThread的hook。不管哪种检测，都是在这些对象的销毁时机（比如Activity的onDestroy）里调用ObjectWatcher的expectWeaklyReachable函数，该函数会检查现有队列（ReferenceQueu）里有没有为空的，说明已经被回收没有泄漏，然后将这些activity、fragment等的弱引用生成KeyedWeakReference对象加入到队列里，然后会再检查一遍有没有被回收，没有的话则调用onObjectRetained()，会走到HeapDumpTrigger的scheduleRetainedObjectCheck函数，然后是checkRetainedObjects函数，这里会强制执行一次GC（Runtime.getRuntime().gc()）并做短暂的sleep，然后看看有没有还没回收的对象，有的话根据一定条件触发dump。
参见源码解读文章：  
https://blog.51cto.com/u_15375308/4660492
https://www.jianshu.com/p/33d786d981b5 
## 1.2. [KOOM](https://github.com/KwaiAppTeam/KOOM)
# 2. 内存泄漏的监控
## 2.1. Java内存监控
用于监控应用的 Java 内存泄漏问题，它的核心原理:
周期性查询Java堆内存、线程数、文件描述符数等资源占用情况，当连续多次超过设定阈值或突发性连续快速突破高阈值时，触发镜像采集镜像采集采用虚拟机supend->fork虚拟机进程->虚拟机resume->dump内存镜像的策略，将传统Dump冻结进程20s的时间缩减至20ms以内基于shark执行镜像解析，并针对shark做了一系列调整用于提升性能，在手机设备测即可执行离线内存泄露判定与引用链查找，生成分析报告。
### 2.1.1. activity泄漏
分析dump的数据，如果activity的mFinished或mDestroyed值为true则是可能泄漏了
### 2.1.2. fragment泄漏
Fragment的销毁判定规则为，mFragmentManager为空且mCalled为true，mCalled为true表示此fragment已经经历了一些生命周期
### 2.1.3. 大内存对象的监测，比如：大Bitmap、大的原始类型数组、大的Object数组等
### 2.1.4. OOM监控
#### 2.1.4.1. heap OOM：连续几次发现堆内存都到达了指定的阈值
#### 2.1.4.2. thread OOM：打开的线程数过多：读取/proc/self/task
#### 2.1.4.3. fd OOM：打开的文件数过多：读取/proc/self/fd
#### 2.1.4.4. FastHugeMemoryOOMTracker：内存暴涨过快，比如打开了一张大图  
#### 2.1.4.5. native heap OOM：判断Debug.getNativeHeapAllocatedSize()占比总的内存值
## 2.2. Native内存监控
### 2.2.1. [libmemunreachable](https://android.googlesource.com/platform/system/memory/libmemunreachable/+/master/README.md)
KOOM使用的是这个
参见KOOM的原理：https://github.com/KwaiAppTeam/KOOM/blob/master/koom-native-leak/README.zh-CN.md  
hook malloc/free 等内存分配器方法，用于记录 Native 内存分配元数据「大小、堆栈、地址等」
周期性的使用 mark-and-sweep 分析整个进程 Native Heap，获取不可达的内存块信息「地址、大小」
利用不可达的内存块的地址、大小等从我们记录的元数据中获取其分配堆栈，产出泄漏数据「不可达内存块地址、大小、分配堆栈等」
### 2.2.2. [GWP-ASan](https://developer.android.com/ndk/guides/gwp-asan)？
参见：https://juejin.cn/post/7027025137320853534
https://juejin.cn/post/7025432746382065678
## 2.3. I/O监控
可以考虑只监控主线程的I/O操作，natvie hook来实现。
可以参考matrix的：https://github.com/Tencent/matrix/blob/master/matrix/matrix-android/matrix-io-canary/src/main/cpp/io_canary_jni.cc
以下摘自：张绍文的高手课：https://time.geekbang.org/column/article/75914

Profilo 使用到是 PLT Hook 方案，它的性能比GOT Hook要稍好一些，不过 GOT Hook 的兼容性会更好一些。关于几种 Native Hook 的实现方式与差异，我在后面会花篇幅专门介绍，今天就不展开了。最终是从 libc.so 中的这几个函数中选定 Hook 的目标函数。
```

int open(const char *pathname, int flags, mode_t mode);
ssize_t read(int fd, void *buf, size_t size);
ssize_t write(int fd, const void *buf, size_t size); write_cuk
int close(int fd);
```
因为使用的是 GOT Hook，我们需要选择一些有调用上面几个方法的 library。微信 Matrix 中选择的是libjavacore.so、libopenjdkjvm.so、libopenjdkjvm.so，可以覆盖到所有的 Java 层的 I/O 调用，具体可以参考[io_canary_jni.cc](https://github.com/Tencent/matrix/blob/master/matrix/matrix-android/matrix-io-canary/src/main/cpp/io_canary_jni.cc)。
不过我更推荐 Profilo 中[atrace.cpp](https://github.com/facebookincubator/profilo/blob/main/cpp/atrace/Atrace.cpp#L172)的做法，它直接遍历所有已经加载的 library，一并替换。
```
void hookLoadedLibs() { auto& functionHooks = getFunctionHooks(); auto& seenLibs = getSeenLibs(); facebook::profilo::hooks::hookLoadedLibs(functionHooks, seenLibs);}
```
不同版本的 Android 系统实现有所不同，在 Android 7.0 之后，我们还需要替换下面这三个方法。
```
open64  
__read_chk  
__write_chk
```

## 2.4. 线程泄漏
### 2.4.1. 编译期将new Thread方式的替换成线程池方式，将线程池调用方式的统一变为自己的线程池调用方式，避免因线程池参数设置不对导致的核心线程空闲的时候没有释放，会使整体的线程数量处于较高位置。具体的：
1. 限制空闲线程存活时间，keepAliveTime 设置小一点，例如 1-3s
2. 允许核心线程在空闲时自动销毁（allowCoreThreadTimeOut）
详细可见滴滴booster的说明：https://booster.johnsonlee.io/zh/guide/performance/multithreading-optimization.html#通过命令行配置

### 2.4.2. 线程监控
hook pthread_create/pthread_exit 等线程方法，用于记录线程的生命周期和创建堆栈，名称等信息
当发现一个joinable的线程在没有detach或者join的情况下，执行了pthread_exit，则记录下泄露线程信息
当线程泄露时间到达配置设置的延迟期限的时候，上报线程泄露信息
参见KOOM的：https://github.com/KwaiAppTeam/KOOM/blob/master/koom-thread-leak/README.zh-CN.md