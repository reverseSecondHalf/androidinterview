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
3. 类型不同：Activity和Dialog的Window类型TYPE_APPLICATION,PopupWindow的是TYPE_APPLICATION_PANEL，Toast的是TYPE_TOAST，属于系统窗口的一种。不同类型窗口的zorder值不一样，大的覆盖显示小的，系统窗> 子window>应用（参见：https://blog.csdn.net/weixin_43766753/article/details/108350589）
4. 是否能直接获取输入事件：
   Activity和Dialog都对应一个PhoneWindow对象。PopupWindow和Toast没有对应的PhoneWindow，Activity/Dialog由于有PhoneWindow因此就有了DecorView，DecorView可以通过Window.Callback将输入事件派发给Activity/Dialog。
而PopupWindow/Toast都是用一个ViewGroup作为窗口的View，因此PopupWindow/Toast本身不具备获取输入事件的能力，输入事件是直接派发到了View tree。由于PopupWindow/Toast也是通过WindowManagerImpl添加的窗口，因此他们都有对应的ViewRootImpl, 因此也就具备了与WMS通信和事件派发的能力。
   
   参考：https://blog.csdn.net/feiduclear_up/article/details/49080587
   https://juejin.cn/post/6844904096114147341

# input事件
http://gityuan.com/2016/12/31/input-ipc/
https://www.jianshu.com/p/b7cef3b3e703