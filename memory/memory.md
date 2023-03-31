# 内存泄漏的排查
## 使用LeakCanary在开发期间就降低内存泄漏
## APM工具在线上对内存泄漏进行监控
# OOM排查
## APM工具在线上对内存泄漏进行监控
# 匿名共享内存加锁方式（需要支持跨进程）
参考：https://github.com/Tencent/MMKV/wiki/android_ipc
1. 类似mmkv这种使用文件锁，但可能需要做额外封装
2. 使用互斥锁pthread_mutex_lock,然而 Android 版的 pthread_mutex 并不保证robust，亦即对 pthread_mutex 加了锁的进程被 kill，系统不会进行清理工作，这个锁会一直存在下去，那么其他等锁的进程就会永远饿死。