官方介绍文档：https://source.android.com/docs/core/runtime?hl=zh-cn  
https://zhuanlan.zhihu.com/p/375238193
# Android的类加载机制
https://www.jianshu.com/p/ff489696ada2
https://blog.csdn.net/dengjiamingcsu/article/details/79809274
# tinker热修复原理
https://github.com/Tencent/tinker/blob/dev/tinker-android/tinker-android-loader/src/main/java/com/tencent/tinker/loader/SystemClassLoaderAdder.java
这个文件里的V23的install函数去反射拿到BaseDexClassLoader里的pathList（libcore/dalvik/src/main/java/dalvik/system/DexPathList.java），然后再往其dexElement（Element[]）里添加修复后的dex，并且排到队列前面，这样pathList的findClass就会先找到修复后的dex并返回。
# JVM vs Dalvik vs ART
dalvik为什么基于寄存器（没有和jvm一样基于栈是因为寄存器更快，更适合性能差的移动设备）  
为什么没有用java一样的字节码（信息有冗余、android把多个.class编译成一个.dex效率更高）
https://www.jianshu.com/p/857e2541ff01  
https://juejin.cn/post/6844903480520359944
栈与寄存器的对比：https://paul.pub/android-dalvik-vm/
# Dalvik内存模型
注意与JVM内存模型区分，是有不同的。  
JVM：线程共享区（堆、方法区）、私有区（栈帧、程序计数器、本地方法栈）、直接内存。  
Dalvik：线程共享区（堆、方法区）、私有区（虚拟机栈、程序计数器、本地方法栈）。  

每个线程都会开辟一个虚拟机栈，每个方法对应一个栈帧，无论是 JVM 还是 Dalvik&ART 虚拟机在栈结构上都是这样的。栈区就是管理方法的运行。

一个栈帧中无论是局部变量表还是操作数栈，都是基于栈结构。指令执行的最小单位是一个字节 8 位，这样能兼容到所有设备；方法运行时局部变量表和操作数栈会不断的入栈出栈。这种设计能够体现跨平台的优势。

在 Android 为了能够一次处理更多的指令有更快的运行速度，Dalvik&ART 虚拟机参考 CPU 结构，在栈帧内部将局部变量表和操作数栈的概念去除了，换为寄存器结构（也可以理解这里的寄存器就是局部变量表+操作数栈），寄存器能够一次操作 2/4/6 个字节单位的指令，用空间换时间。

需要注意的是，无论是基于寄存器指令集结构还是基于栈指令集结构，它们都是有使用栈结构，一个线程还是一个虚拟机栈，还是用的栈帧，区别是栈帧内部结构的不同，JVM 栈帧内部是局部变量表和操作数栈，而 Dalvik&ART 虚拟机是寄存器。

https://blog.csdn.net/qq_31339141/article/details/124053400

# 垃圾回收
参见：https://paul.pub/android-art-vm/#id-垃圾回收
## Dalvik虚拟机
原理参见：https://www.kancloud.cn/alex_wsc/androids/473616


Dalvik 的问题：碎片化

在Dalvik虚拟机中，堆实际上就是一块匿名共享内存。Dalvik虚拟机并不是直接管理这块匿名共享内存，而是将它封装成一个mspace，交给C库来管理。mspace是libc中的概念，我们可以通过libc提供的函数create_mspace_with_base创建一个mspace，然后再通过mspace_开头的函数管理该mspace。  
使用Mark-Sweep算法。  
Mark阶段从对象的根集开始标记被引用的对象。标记完成后，就进入到Sweep阶段，而Sweep阶段所做的事情就是回收没有被标记的对象占用的内存。

在垃圾收集的Mark阶段，要求除了垃圾收集线程之外，其它的线程都停止，否则的话，就会可能导致不能正确地标记每一个对象。这种现象在垃圾收集算法中称为Stop The World，会导致程序中止执行，造成停顿的现象。为了尽可能地减少停顿，我们必须要允许在Mark阶段有条件地允许程序的其它线程执行。这种垃圾收集算法称为并行垃圾收集算法（Concurrent GC）。
为了实现Concurrent GC，Mark阶段又划分两个子阶段。第一个子阶段只负责标记根集对象(GC ROOTS)。所谓的根集对象，就是指在GC开始的瞬间，被全局变量、栈变量和寄存器等引用的对象。有了这些根集变量之后，我们就可以顺着它们找到其余的被引用变量。例如，一个栈变量引了一个对象，而这个对象又通过成员变量引用了另外一个对象，那该被引用的对象也会同时标记为正在使用。这个标记被根集对象引用的对象的过程就是第二个子阶段。在Concurrent GC，第一个子阶段是不允许垃圾收集线程之外的线程运行的，但是第二个子阶段是允许的。不过，在第二个子阶段执行的过程中，如果一个线程修改了一个对象，那么该对象必须要记录起来，因为它很有可能引用了新的对象，或者引用了之前未引用过的对象。如果不这样做的话，那么就会导致被引用对象还在使用然而却被回收。这种情况出现在只进行部分垃圾收集的情况，这时候Card Table的作用就是用来记录非垃圾收集堆对象对垃圾收集堆对象的引用。Dalvik虚拟机进行部分垃圾收集时，实际上就是只收集在Active堆上分配的对象。因此对Dalvik虚拟机来说，Card Table就是用来记录在Zygote堆上分配的对象在部收垃圾收集执行过程中对在Active堆上分配的对象的引用。


## Android5-7.1
1. ART 有多个不同的 GC 方案，这些方案包括运行不同垃圾回收器。默认方案是 CMS（Concurrent Mark Sweep，并发标记清除）方案，主要使用粘性（sticky）CMS 和部分（partial）CMS。粘性CMS是ART的不移动（non-moving ）分代垃圾回收器,它仅扫描堆中自上次 GC 后修改的部分，并且只能回收自上次GC后分配的对象。   
   注：垃圾回收策略有三种类型：
Sticky 仅仅释放上次GC之后创建的对象  
Partial 仅仅对应用程序的堆进行垃圾回收，但是不处理Zygote的堆  
Full 会对应用程序和Zygote的堆都会进行垃圾回收 
2. 同时增加了后台或者内存分配不足时进行Compacting GC操作，减少了内存碎片化的问题。  
3. 除了新的垃圾回收器之外，ART 还引入了一种基于位图的新内存分配程序，称为 RosAlloc（插槽运行分配器）。此新分配器具有分片锁，当分配规模较小时可添加线程的本地缓冲区，因而性能优于 DlMalloc。

与Dalvik相比，暂停次数2次减少到1次。Dalvik的第一次暂停主要是为了进行根标记。而在ART中，标记过程是并发进行的，它让线程标记自己的根，然后马上就恢复运行。


引入分代管理
将堆分为新生代 (Young Generation) 和老年代 (Old Generation)，对应的GC也分为两种：  
Minor GC: 针对新生代的垃圾回收  
Major GC (Full GC) : 针对整个堆的垃圾回收
引入分代管理是基于这样的假设：生命周期短的对象 (如临时生成的对象) 的创建和销毁要比 (生命周期) 长的对象更加频繁。

程序运行时，触发的大部分是 Minor GC，只会针对新生代做处理，这样比 Dalvik 的全局回收要高效很多，极大降低了创建临时变量的开销。

应用进入后台之前，它会避免执行压缩。  
应用进入后台之后，它会暂停应用线程以执行压缩（Stop-The-World）。  
如果对象分配因内存碎片而失败，则必须执行压缩操作，应用可能会短时间无响应。

Compacting GC包括Semi-Space（SS）、Generational Semi-Space（GSS）和Mark-Compact （MC）三种。  
Semi-Space（SS）GC和Generational Semi-Space（GSS）GC的共同特点是都具有一个From Space和一个To Space。在GC执行期间，在From Space分配的还存活的对象会被依次拷贝到To Space中，这样就可以达到消除内存碎片的目的。  
与SS GC相比，GSS GC还多了一个Promote Space。当一个对象是在上一次GC之前分配的，并且在当前GC中仍然是存活的，那么它就会被拷贝到Promote Space中，而不是To Space中。这相当于是简单地将对象划分为新生代和老生代的，即在上一次GC之前分配的对象属于老生代的，而在上一次GC之后分配的对象属于新生代的。一般来说，老生代对象的存活性要比新生代的久，因此将它们拷贝到Promote Space中去，可以避免每次执行SS GC或者GSS GC时，都需要对它们进行无用的处理。

Mark-Compact GC与Semi-Space GC和Generational Semi-Space GC相比，它们的共同特点都是在Stop-the-word前提下进行的，并且都是通过移动对象来实现垃圾回收和内存压缩的目的。不过前者的对象移动操作是就地进行的，而后两者的对象移动操作是要通过一个额外的Space来进行的。当然，Mark-Compact GC的对象移动操作可以就地进行并不是免费的午餐，它要付出的代价在回收阶段执行一个额外的新地址计算操作，这又是一个计算机世界以时间换空间或者说以空间换时间的例子。

ART运行时堆划分为四个空间，分别是Image Space、Zygote Space、Allocation Space和Large Object Space。其中，Image Space、Zygote Space、Allocation Space是在地址上连续的空间，称为Continuous Space，而Large Object Space是一些离散地址的集合，用来分配一些大对象，称为Discontinuous Space。根据GC算法不同，可能还会有别的space（详细可见：https://paul.pub/android-art-vm/#id-垃圾回收）

参见：https://www.kancloud.cn/alex_wsc/androids/473625
https://juejin.cn/post/6961611061229256712
## Android 8 开始
默认采用并发复制Concurrent Copying （CC）方案。

支持使用名为“RegionTLAB”的触碰指针分配器。此分配器可以向每个应用线程分配一个线程本地分配缓冲区 (TLAB)，这样，应用线程只需触碰“栈顶”指针，而无需任何同步操作，即可从其 TLAB 中将对象分配出去。  
依靠读取屏障拦截来自堆的引用读取，并发复制对象来执行堆碎片整理，从而不用暂停用户线程。  
GC 只有一次很短的暂停，对于堆大小而言，该次暂停在时间上是一个常量。

从Android 8.0 开始，ART 引入了 Bump Allocator 的机制来分配内存，处理方式如下：

堆内存被分割为多个 region
每个线程对应一个 region, 并且维护一个指向下一个空闲空间的指针 Bump Pointer
分配内存时，直接 Bump Pointer 指针指向的地址分配即可，由于每个线程有对应的region，所以分配的过程是并发的，非常高效。
当前 region 剩余空间不够时，触发 compaction 操作，具体步骤见下一节。
基于上面的优化，相比 Android 4.4 的 Dalvik，ART 在 Android 8.0 的内存分配性能提升到18x，表现已经好于大部分的带锁对象池实现了。

在 Concurrent Copying (CC) GC 中，GC时会遍历每个region，根据当前的对象状态来决定是否进行 copy 操作，具体过程如下：
（参考：https://zhuanlan.zhihu.com/p/375238193）

找到 GC Root 并标记 GC Root 引用的对象  
标记出未被 GC Root 引用的对象(垃圾对象)  
对region进行分析，决定每一个region是否进行copy（经过分析，垃圾对象较多的区域会被搬移， 而垃圾对象较少的区域不会被搬移，原因这个区域大部分对象后面还会用到，copy操作是有成本的，全量copy不划算）  
copy 对象到新的 region
## Android 10 开始

默认采用并发复制（CC）方案，但是增加了分代处理。

支持快速回收存留期较短的对象，提高 GC 吞吐量，并降低全堆 GC 的执行。

（参见：https://zhuanlan.zhihu.com/p/375238193）
从 Android 10 开始，Concurrent Copying (CC) GC 中加入了分代管理机制(也不知道当时为啥要移除)，有了分代管理，对应的 Minor GC 与 Major GC 也就回来，这一块我们重点讲一下对应的工作流程：

在 Generational CC 中，堆内存并没有显式地划分为不同的代，而是在运行时 把不同的 region 标记为新生代或者老年代；

CC Minor GC - 根据 GC Root，标记新生代 region 的对象  
Minor GC 不会导致追踪(trace)老年代region中的对象，但如果新生代region中的对象被老年代region引用，还是要在copy后更新对应的引用，这里用到了一个Remember Set的机制，将这一操作的开销讲到最小。  
新生代对象 copy 到新的 region 中，原region被回收，Minor GC 完成

 CC Major GC:

追踪所有的region，(根据规则)标记要回收的 region

 将需要回收的 region 中的对象 copy 到新的 region 中，回收原来的 region

## 对比：
https://pic4.zhimg.com/80/v2-8fb3acd7e8460ca5c85dcc9bdca113b3_1440w.webp