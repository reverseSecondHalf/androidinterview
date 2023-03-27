# 设计模式
参考：https://www.jianshu.com/p/6e5eda3a51af
# MVC MVP MVVM 
https://juejin.cn/post/6844903913418670093  
MVC:   
View:xml + Activity
Controller: Activity
问题：Activity臃肿，兼具view和controller的功能  
MVP:  
View：Activity
为了解决MVC的问题，通过Presenter将view和控制器的功能隔离开。
通过Presenter实现数据和视图之间的交互，完全隔离了View层与Model层，二者互不干涉。
Activity代码变得更加简洁：简化了Activity的职责，仅负责UI相关操作，其余复杂的逻辑代码提取到了Presenter层中进行处理  

Activity和presenter会相互持有，业务复杂度增加后会导致接口过多。  

MVVM：  
基本上与 MVP 模式完全一致，将逻辑处理层 Presenter 改名为 ViewModel。
通过databinding或者主动observe实现了数据和UI的联动。   
viewmodel的生命周期自动管理，可以减少内存泄漏的问题，而且生命周期比activity长，在横竖屏切换时数据还在（activity onDestroy时会判断是不是横竖屏切换导致的，是的话不销毁viewmodel）。  
ViewModelStore内部持有一个HashMap，这是 ViewModel 实例的最终存放点，而 ViewModelStore 的实例是通过ViewModelStoreOwner获取，Activity 就是ViewModelStoreOwner实例，且持有ViewModelStore实例。但当横竖屏切换 Activity 销毁重建时，作为成员变量的 ViewModelStore 依然会被销毁，为了避免它被重建，在配置发生变化时 onRetainNonConfigurationInstance()，ViewModelStore 实例还会被寄存在一个静态类NonConfigurationInstances中，这样在恢复时，就可以从中恢复 ViewModelStore 实例。