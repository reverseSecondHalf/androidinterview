参见：https://juejin.cn/post/7052614577216815134#heading-16
https://juejin.cn/post/7016225898768629773#heading-9
# 1. 插件化
# 2. [AndResGuard](https://github.com/shwenzhang/AndResGuard)
它将资源的名称进行了混淆和缩减。你完全可以用它对resources.arsc进行优化，只是具体优化效果与编码方式、id数量、平均减少命名长度有关。优化后的资源名称会发生改变，一般会缩减为一到两个英文字母.
# 3. R8
开启R8能带来一定程度的减包。
# 4. 找到相同或接近的图片
相同资源做下沉处理，以供各业务使用
可以用[matrix-apk-canary](https://github.com/Tencent/matrix)来扫，还能扫一些别的
# 5. [内部private方法去除](https://mp.weixin.qq.com/s/ZHisCVjO_ZrtvvEWBYUQFQ)
# 6. png压缩
# 7. 删除R文件
（https://github.com/jokermonn/thinr3不确定是否可用）
# 8. 去除行号等proguard的优化
# 9. 动态库打包
使用c++_shared
so压缩
去除debug信息
