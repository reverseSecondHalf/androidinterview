# [activity启动流程](https://www.jianshu.com/p/c2144e21deca)
# 子线程是否可以更新UI
activity的onResume之前可以
# 点击icon到activity显示发生了哪些
参考：https://juejin.cn/post/6844903959581163528  
https://juejin.cn/post/7179901192229617724
https://juejin.cn/post/6844903959589552142
## Launcher进程请求AMS
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8c05d64c40a~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
至少android12开始类名稍微有所不同：
![image](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202212211646368.png)

## AMS发送创建应用进程请求
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8c04147e855~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
ActivityTaskSupervisor的startSpecificActivity函数里会判断进程是否存在，是的话则调用当前类的realStartActivityLocked(这块兜兜转转最终会走到ActivityThread的handleLaunchActivity)，否则会调用ActivityTaskManagerService的startProcessAsync来创建进程。  
startProcessAsync会走到ActivityManagerService的startProcess，最终会走到Process的start函数，这里会向Zygote进程发送创建进程的请求。
## Zygote进程接受请求并孵化应用进程
zygote通过一个死循环一直监听socket连接，有的话通过fork创建一个新进程。（为什么不用binder通信而是用socket：  
1. zygote和servicemanager的初始化时机问题，zygote需要使用binder时servicemanager不一定能保证初始化好了；
2. Linux中，fork进程其实并不是完美的fork，linux设计之初只考虑到了主线程的fork，也就是说如果主进程中存在子线程，那么fork进程中，其子线程的锁状态，挂起状态等等都是不可恢复的，只有主进程的才可以恢复。而binder作为典型的CS模式，其在Server是通过线程来实现的，Server等待请求状态时，必然是处于一种挂起的状态。所以如果使用binder机制，zygote进程fork子进程后，子进程的binder的Server永远处于一种挂起不可恢复的状态，这样的设计无疑是非常差的。所以，zygote如果使用binder，会导致子进程中binder线程的挂起和锁状态不可恢复，这是原因之二。 参考：https://blog.csdn.net/rzleilei/article/details/125770598?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-125770598-blog-106444099.pc_relevant_default&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-125770598-blog-106444099.pc_relevant_default&utm_relevant_index=1)
## 应用进程启动ActivityThread
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8c06d05ab45~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
通过反射调用到ActivityThread的main函数。
## 应用进程绑定到AMS
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8e5b89da8cb~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
ActivityThread的main函数会调用ActivityManagerService的attachApplication方法，这里最终会调用到android.app.ActivityThread.ApplicationThread的bindApplication，
这里会进行providers的安装和应用Application的onCreate调用。
## AMS发送启动Activity的请求
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8e5b8b0ec4d~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)
现在当前进程已经绑定到了AMS，绑定后呢？回忆一下，上面对AMS的attachApplicationLocked方法分析时，重点提到了两个方法，其中ApplicationThread的bindApplication方法我们分析应用进程是如何绑定到AMS的，没错！另外一个方法mStackSupervisor.attachApplicationLocked(app)就是用来启动Activity的，现在让我们来看看。  

在ActivityThread的sendMessage中会把启动Activity的消息发送给mH,而mH为H类型，其实就是ActivityThread的Handler。到这里AMS就将启动Activity的请求发送给了ActivityThread的Handler。
## ActivityThread的Handler处理启动Activity的请求
![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/9/16daf8e5b8cbe5ce~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.image)

Handler接受到AMS发来的EXECUTE_TRANSACTION消息后，将调用TransactionExecutor.execute方法来切换Activity状态，先是会调用LaunchActivityItem里的execute方法，这里又会调用到ActivityThread的handleLaunchActivity以及接下来的performLaunchActivity方法，这个方法里会调用activity的attach，然后走onCreate方法。

从上面分析中我们知道onCreate已经被调用，且生命周期的状态是ON_CREATE，故executeCallbacks已经执行完毕，所以开始执行executeLifecycleState方法。如下

frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java

反正接下来就会走onStart和onResume。细节不表了。
# 动画原理
参考：https://www.jianshu.com/p/5d0899dca46e
## 逐帧动画
根据提供的每帧动画，给渲染出来
## 补间动画
简单理解就是在每一次VSYN到来时 在View的draw方法里面 根据当前时间计算动画进度 计算出一个需要变换的Transformation矩阵 然后最终设置到canvas上去 调用canvas concat做矩阵变换.动画过程中不会真正实现view的移动。

## 属性动画
属性动画的工作原理很简单，其实就是在一定的时间间隔内，通过不断地对值进行改变，并不断将该值赋给对象的属性，从而实现该对象在属性上的动画效果，属性动画真正实现了View的移动

# setContentView流程
1. 一般设置布局是通过在activity的onCreate里通过setContentView方法，从这里说起，这个方法最终会调用到Activity的setContentView方法，这里是通过getWindow().setContentView的方法来设置的。
2. 这个getWindow即mWindow，是在Activity的attach方法里被赋值的:
mWindow = new PhoneWindow(this, window, activityConfigCallback);
attach方法是在activity的启动过程中，通过ActivityThread的performLaunchActivity触发的。
3. PhoneWindow的setContentView主要包含两步：installDecor和mLayoutInflater.inflate，installDecor主要是new了一个DecorView，然后generatelayout，这个generateLayout函数最终会走DecorView的onResourcesLoaded，会将一个默认的布局文件解析成View并且添加到DecorView上。
4. LayoutInflater的inflate函数主要是：如果是merge标签单独处理，否则是通过createViewFromTag把布局文件生成view，并且根据inflate的两个属性（rootView、是否要attach到root）来看看是不是把这个生成的view设置下layoutParams；再通过rInflateChildren方法对子控件进行遍历；最后看看有root的话并且需要attach就addView到root。
5. 另外第4步骤中，createViewFromTag分为tryCreateView和onCreateView，前者tryCreateView是通过Factory或者Factory2接口hook住view的创建过程，比如我们在Activity的onCreate之前调用LayoutInflater.from(this).setFactory2，然后在onCreateView的实现里自己去做一些定制化操作，比如将TextView都换成我们一个自定义的TextView，后者onCreateView是通过反射方法来创建View的。

# Android的Context机制
1. 有三种：Application、Activity、Service，不同的Context权限有所不同，比如只有Activity才可以启动一个普通的Dialog
2. 有了context，才可以进行android的一些api的访问，资源的使用。
   参考：https://blog.csdn.net/weixin_43766753/article/details/109017196
# Window机制
window机制就是为了管理屏幕上的view的显示以及触摸事件的传递问题。
参见：https://blog.csdn.net/weixin_43766753/article/details/108350589

# Activity、Dialog、PopupWindow、Toast的区别
1. 相同点是最终都是通过WindowManager来进行addView显示的。
2. Dialog和PopupWindow都依赖于activity的存在而存在，都是和activity公用AppWindowToken的。Toast的token是由NotificationManagerService创建的，每个Toast对应一个自己的token，Toast的token在WMS对应的是WindowToken。Toast对话框和Dialog，PopupWindow管理机制不同。Dialog，PopupWindow都是在当前应用中管理，不涉及到系统管理。
3. 类型不同：Activity和Dialog的Window类型TYPE_APPLICATION,PopupWindow的是TYPE_APPLICATION_PANEL，Toast的是TYPE_TOAST，属于系统窗口的一种。
4. 是否能直接获取输入事件：
   Activity和Dialog都对应一个PhoneWindow对象。PopupWindow和Toast没有对应的PhoneWindow，Activity/Dialog由于有PhoneWindow因此就有了DecorView，DecorView可以通过Window.Callback将输入事件派发给Activity/Dialog。
而PopupWindow/Toast都是用一个ViewGroup作为窗口的View，因此PopupWindow/Toast本身不具备获取输入事件的能力，输入事件是直接派发到了View tree。由于PopupWindow/Toast也是通过WindowManagerImpl添加的窗口，因此他们都有对应的ViewRootImpl, 因此也就具备了与WMS通信和事件派发的能力。
   
   参考：https://blog.csdn.net/feiduclear_up/article/details/49080587
   https://juejin.cn/post/6844904096114147341

# input事件
http://gityuan.com/2016/12/31/input-ipc/
https://www.jianshu.com/p/b7cef3b3e703