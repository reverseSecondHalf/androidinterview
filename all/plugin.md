# Shadow插件化原理
https://zhuanlan.zhihu.com/p/74594715
https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650247043&idx=1&sn=bfc5b48b8d7b2f6c764c4e3da1bcd44a&chksm=88637eecbf14f7faa1292daeb13dc858e085f6b4620fd6ac825876d210999186056da9e664ed&scene=27

1. classloader那块，反射修改了mParent，让类的查找遵从系统->插件的

# 资源路径的处理
参见：
https://zhuanlan.zhihu.com/p/450350883
1. 合并式：

addAssetPath时加入所有插件和主工程的路径；
优点：插件和宿主能直接相互访问资源
缺点：会引入资源冲突(由于主工程和各个插件都是独立编译的，生成的资源id会存在相同的情况)
实现代码如下，其中我将插件的包名与宿主设置成相同的，具体原因见下面shadow部分的说明：
binding.btnPrintResources.setOnClickListener {
    val plugin001Path = File(filesDir.absolutePath, "plugin001.apk").absolutePath
    val plugin002Path: String = File(filesDir.absolutePath, "plugin002.apk").absolutePath
    val mResources = loadResources(this,resources.assets, listOf(pluginPath, pluginPath2))
    val strAppId = mResources?.getIdentifier("str_app", "string", "com.jinyang.plugindemo")
    log("str_app:"+ strAppId?.let { it1 -> mResources.getString(it1) })
    val strPlugin001Id = mResources?.getIdentifier("str_plugin001", "string", "com.jinyang.plugindemo")
    log("str_plugin001:"+ strPlugin001Id?.let { it1 -> mResources.getString(it1) })
    val strPlugin002Id = mResources?.getIdentifier("str_plugin002", "string", "com.jinyang.plugindemo")
    log("str_plugin002:"+ strPlugin002Id?.let { it1 -> mResources.getString(it1) })
}

fun loadResources(context: Context,assetManager:AssetManager, pluginPaths: List<String>): Resources? {
        try {
            val addAssetPathMethod = assetManager::class.java.getDeclaredMethod("addAssetPath", String::class.java)
            addAssetPathMethod.isAccessible = true
            for (path in pluginPaths) {
                addAssetPathMethod.invoke(assetManager, path)
            }
            return Resources(
                assetManager,
                context.resources.displayMetrics,
                context.resources.configuration
            )
        } catch (e: Exception) {
            e.printStackTrace()
        }
        return null
}
2. 独立式：

各个插件只添加自己apk路径
优点：资源隔离，不存在资源冲突
缺点：资源共享比较麻烦（如果想要实现资源的共享，必须拿到对应的Resource对象）
实现方式如下，在各个插件的baseActivity中重写getResources，getAssets方法
open class PluginBaseActivity : Activity() {
    private var pluginClassLoader: ClassLoader? = null
    private var pluginPath: String?=null
    private var pluginAssetManager: AssetManager? = null
    private var pluginResources: Resources? = null
    private var pluginTheme: Resources.Theme? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val nativeLibDir = File(filesDir, "pluginlib").absolutePath
        val dexOutPath = File(filesDir, "dexout").absolutePath
        pluginPath = File(filesDir.absolutePath, "plugin002.apk").absolutePath
        pluginClassLoader = DexClassLoader(pluginPath, dexOutPath, nativeLibDir, this::class.java.classLoader)
        handleResources()
    }

    override fun getResources(): Resources? {
        return pluginResources ?: super.getResources()
    }

    override fun getAssets(): AssetManager {
        return pluginAssetManager ?: super.getAssets()
    }

    override fun getClassLoader(): ClassLoader {
        return pluginClassLoader ?: super.getClassLoader()
    }

    private fun handleResources() {
        try {
            pluginAssetManager = AssetManager::class.java.newInstance()
            val addAssetPathMethod = pluginAssetManager?.javaClass?.getMethod("addAssetPath", String::class.java)
            addAssetPathMethod?.invoke(pluginAssetManager, pluginPath)
        } catch (e: Exception) {
        }
        pluginResources = Resources(pluginAssetManager, super.getResources().displayMetrics, super.getResources().configuration)
        pluginTheme = pluginResources?.newTheme()
        pluginTheme?.setTo(super.getTheme())
    }
}
解决资源id冲突问题的方法：

修改aapt源码，定制aapt工具，编译期间修改PP段;（DynamicAPK使用此方案），原理参考：Android中如何修改编译的资源ID值
修改aapt的产物resources.arsc文件，即，编译后期重新整理插件Apk的资源，编排ID;(VirtualApk使用此方案)，原理参考：插件化-解决插件资源ID与宿主资源ID冲突的问题
通过配置aaptOptions，build.gradle中的android节点加入如下代码，不过此方法只有在compileSdkVersion为28及以上才生效
android {
    aaptOptions {
        additionalParameters  "--package-id", "0x66","--allow-reserved-package-id"
    }
    ...
}
