# Shadow插件化原理
https://zhuanlan.zhihu.com/p/74594715
https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650247043&idx=1&sn=bfc5b48b8d7b2f6c764c4e3da1bcd44a&chksm=88637eecbf14f7faa1292daeb13dc858e085f6b4620fd6ac825876d210999186056da9e664ed&scene=27

1. classloader那块，反射修改了mParent，让类的查找遵从系统->插件的
