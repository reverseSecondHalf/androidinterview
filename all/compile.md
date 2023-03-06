# android R文件那些事
R文件是个比较不严格的说法，实际有R.txt, R.java, R.class, R.jar，还有R$*这种class文件（如R$id, R$layout 等等)。
参考：https://mp.weixin.qq.com/s/fxrM6VWlyb8EujwU_mZIXw
## R.class vs R.java
agp 3.6开始不生成R.java，而是直接生成R.class，之前是通过生成R.java再用javac生成R.class。[Android Gradle plugin has made significant performance improvement for annotation processing/KAPT for large projects. This is caused by AGP now generating R class bytecode directly, instead of .java files.](https://android-developers.googleblog.com/2020/02/android-studio-36.html)
## R.txt
aar里必不可少的，资源列表
## R-class文件传递
AGP3.3以前，假如A模块依赖B，B模块依赖C，那么A模块中的R$id这种R-class文件会包含他所依赖的B和C中的所有。AGP 3.3引入了实验性质的namespacedRClass，并在AGP4.1.0中重命名为nonTransitiveRClass，在gradle.properties设置为true即可阻止这种传递依赖，使其只包含自身模块的。
```
android.nonTransitiveRClass=true
```
从 Android Studio Bumblebee 开始，新项目的非传递 R 类默认处于开启状态。对于使用早期版本的 Studio 创建的项目，您可以依次前往 Refactor > Migrate to Non-transitive R Classes，将项目更新为使用非传递 R 类。

## AGP4.1.0
说是已经内联，但是实际demo测试看并没有，why？还是说只是把没有引用的资源给去除了，并没有做内联工作。

App size significantly reduced for apps using code shrinking
[Starting with this release, fields from R classes are no longer kept by default, which may result in significant APK size savings for apps that enable code shrinking. This should not result in a behavior change unless you are accessing R classes by reflection, in which case it is necessary to add keep rules for those R classes.](https://developer.android.com/studio/past-releases/past-agp-releases/agp-4-1-0-release-notes#4.1-app-size-reduction)

## R文件内联
使用[bytex](https://github.com/bytedance/ByteX)或者[滴滴booster](https://github.com/didi/booster)
参见：https://booster.johnsonlee.io/zh/guide/shrinking/res-index-inlining.html#资源索引的问题