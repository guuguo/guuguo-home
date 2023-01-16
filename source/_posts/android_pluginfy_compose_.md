---
title: 【Android进阶】完美插件化实现，compose 开发动态加载

excerpt: 温度爬升，蚊虫也开始猖狂了起来。燥热的空气里，穿梭着几只置身死于度外的飞虫，全然没有在意我这个执掌着生死的巨人，让人钦佩。 一直在小公司徘徊，在小团队里面摸爬滚打。面试中往往会被面试官对于一些平时用不到的技能细致追问，想要跳高只能开始学习工作中用不到的技能，不然就会陷入死循环怪圈

photo: ../images/compose_plugin.png
---
![img.png](../images/compose_plugin.png)
## 参考文章
-  https://juejin.cn/post/6973888932572315678
-  https://zhaomenghuan.js.org/blog/android-plugin-framework-proxy-hook.html
-  https://juejin.cn/post/7028196921143459870
# 前言
温度爬升，蚊虫也开始猖狂了起来。燥热的空气里，穿梭着几只置身死于度外的飞虫，全然没有在意我这个执掌着生死的巨人，让人钦佩。

一直在小公司徘徊，在小团队里面摸爬滚打。面试中往往会被面试官对于一些平时用不到的技能细致追问，想要跳高只能开始学习工作中用不到的技能，不然就会陷入死循环怪圈

- 你没这块工作经验，单位不要。
- 又因为没单位要，所以就没有工作经验。
  所以只能自己偷偷默默学习，以demo当作经验。

# 名词解释
`双亲委派机制` ：在类加载器中，指的是当一个类加载器收到了类加载的请求的时候，他不会直接去加载指定的类，而是把这个请求委托给自己的父加载器去加载。 只有父加载器无法加载这个类的时候，才会由当前这个加载器来负责类的加载。



# 正文
[首先附上demo 源码链接](https://github.com/guuguo/plugin_demo)
插件化一直都没有尝试过。听起来高大上，重要的是面试中被问到的概率也是居高不下。
在开发过程中，不用安装app就能运行新的apk是多么美妙的事情，插件化对于我们的工程应用也有实际意义。本文就以compose Demo项目完整实现一下插件化。完成插件的生成装载，并成功跳转展示对应的插件页面。



# 一，插件化的方案
在Android中，真正安装一个apk的过程很简单，就只是将apk文件拷贝到对应的目录中，并且解压出对应的so文件就好了，其余就是一些解析，扫描组件，校验，dex优化的操作，用户安装包都是拷贝存放在`data/app`中。
启动app之后，会从zygote fork一个新的进程，并使用ClassLoader加载apk。
而我们要动态加载，也是利用了DexClassLoader来加载我们的插件apk。
但是比起真正的安装apk，我们也有以下问题。
1. 如何利用ClassLoader加载apk？
2. 没有注册对应manifest，在启动activity的时候怎么绕过Activity限制？
3. 没有加载资源文件，要如何获取？

**接下来的内容中，以解决以上问题为主要任务，实现完整的插件app Activity启动，展示页面的过程。**
> 本文是对于compose项目的插件化，所以暂不涉及xml布局资源的加载获取。compose布局由代码完成。


# 二，DexClassLoader加载apk

> 如果需要将apk放置到缓存文件夹之外，需要配置好存储权限，并动态请求，否则利用DexClassLoader加载会报错：No original dex files found for dex location

接下来进入正题。

###  我们先创建一个工程。目录如下所示
 ```Dart
 plugin_pjoject
     - app  //主工程app
     - plugin_app  //一个插件app
     - plugin_base  //基础类库，包含插件宿主Activity，加载插件工具类，和基础插件接口类
 ```
### 接下来在`plugin_base`中编写加载代码
插件apk的生成只需要将`plugin_app` 直接打包出apk就行，不需要签名，不会进行签名校验。
方便起见，我们将插件 `apk` 直接放到`assets`里面。
代码逻辑如下：

1. 在app启动的时候，先拷贝assets中的插件`plugin.apk`到缓存目录中
2. 生成一个对应的`DexClassLoader` ，专门用来后续加载插件中的类。

 ```java
 object PluginLoader {
     fun loadPlugin() {
         val inputStream = Utils.getApp().assets.open("plugin.apk")
         val filesDir = Utils.getApp().externalCacheDir
         val apkFile = File(filesDir?.absolutePath, "plugin.apk")
         apkFile.writeBytes(inputStream.readBytes())
 
         val dexFile = File(filesDir, "dex")
         FileUtils.createOrExistsDir(dexFile)
         "输出dex路径${dexFile}".logI()
         pluginClassLoader = DexClassLoader(apkFile.absolutePath, dexFile.absolutePath, null, this.javaClass.classLoader)
     }
 }
 ///插件的类加载器
 lateinit var pluginClassLoader: DexClassLoader;
 ```
这样我们就得到`pluginClassLoader`。
### DexClassLoader源码分析

#### 1. 关联`android dalvik`源码
要想对源码进行分析，肯定要先能看到源码，我们默认在AndroidStudio中无法查看DexClassLoader源码,在manager中下载了对应source也一样。所以我们需要自己下载对应源码和as关联，具体做法参考下面文章学习：
https://www.jianshu.com/p/9af6d2fadcb1

#### 2. 双亲委派机制
在关联好源码后，我们先看CLassLoader类中最关键的loadClass方法

 ```Java
 protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
         // 检查是否被加载过
         Class<?> c = findLoadedClass(name);
         if (c == null) {
             if (parent != null) {
                 ///使用父加载器先加载类
                 c = parent.loadClass(name, false);
             } else {
                 c = findBootstrapClassOrNull(name);
             }
 
             if (c == null) {
                 // 如果仍然没找到，就执行findClass寻找
                 c = findClass(name);
             }
         }
         return c;
     }
 }
 ```
从该版本加载器代码中看到，类加载的默认机制就是先从祖先类加载开始加载，如果找不到就一层层子加载器加载。其中findClass只是个空方法，交由子类自己实现具体代码。
其中`findBootstrapClassOrNull`方法更是直接私有化，并返回空，看起来并没有实际意义。

> 值得注意的是，里面有个静态方法`createSystemClassLoader`，其方法返回了一个`PathClassLoader`作为系统类的加载器。

**重点：在我们的需求中，我们只需要加载宿主或者系统中不存在的类，所以只需要创建一个加载器，并将原来的加载器作为父加载器就行了。**
这样既能加载插件新类，又不影响原来的旧类。
> 从中也能看到，如果我们想要优先让自己的类加载器加载，就需要实现子类覆写loadClass方法，自定义加载逻辑。
#### 3. DexClassLoader 对于apk的加载实现

我们先来看下`DexClassLoader`代码。
 ```java
 /**
 一个类加载器，它从包含classes.dex条目的.jar和.apk文件加载类。这可用于执行未作为应用程序的一部分安装的代码。
 在 API 级别 26 之前，此类加载器需要一个应用程序私有的可写目录来缓存优化的类。使用Context.getCodeCacheDir()创建这样一个目录：
   File dexOutputDir = context.getCodeCacheDir();
 不要在外部存储上缓存优化的类。外部存储不提供保护您的应用程序免受代码注入攻击所必需的访问控制。
 */
 public class DexClassLoader extends BaseDexClassLoader {
     public DexClassLoader(String dexPath, String optimizedDirectory, 
             String librarySearchPath, ClassLoader parent) {
         super(dexPath, null, librarySearchPath, parent);
     }
 }
 ```
`DexClassLoader`代码很简单，本来它和`PathClassLoader`的区别就是可以自定义`optimizedDirectory`，但从注释中可知，为了安全起见，该参数在api 26也就是android 8.0之后就已经失效了，优化输出地址不再可以自由配置，而有系统统一设置。该加载器的所有的实现都在`BaseDexClassLoader`中，只是进行了`optimizedDirectory`的屏蔽操作。
接下来看下`BaseDexClassLoader`的加载方法

 ```java
 @Override
 protected Class<?> findClass(String name) throws ClassNotFoundException {
     // 首先，检查该类是否存在于我们的共享库中。
     //...省略
     //检查该类加载器操作的 dexPath 中是否存在相关类。
     List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
     Class c = pathList.findClass(name, suppressedExceptions);
     //...省略异常处理
     return c;
 }
 ```
其中关键就是通过`**pathList**``.findClass`通过DexPathList集合来查找。再定位到`DexPathList.findClass`代码
 ```java
 public Class<?> findClass(String name, List<Throwable> suppressed) {
     for (Element element : dexElements) {
         Class<?> clazz = element.findClass(name, definingContext, suppressed);
         if (clazz != null) {
             return clazz;
         }
     }
 
     if (dexElementsSuppressedExceptions != null) {
         suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
     }
     return null;
 }
 ```
findClass最后是移交给dexElements处理了。
`dexElements`初始化代码在`makeDexElements`方法中，其中扫描了我们给的apk路径所有文件(给的路径可以是多个，以pathSeparator分割)，并将dex后缀的文件加载为DexFile，并和file一起组装为Element对象放置到Element数组中，给dexELements赋值。
**这就完成了apk内所有dex文件的加载。**
接下来的DexFile具体加载定义类的代码都是native代码，分析就到此为止。

#### 4. 最后总结下流程
对于我们的插件apk加载的过程如下。

1. 以apk为路径，构建DexClassLoader
2. BaseDexClassLoader加载apk内的所有dex文件，加载为Element
3. 插件内的类在需要的时候，在经历过双亲委托的父节点加载器加载后，被DexClassLoader加载出来
   所以对于加载插件，我们只需要创建一个专用的DexClassLoader就行了，如下：

 ```java
 DexClassLoader(apkFile.absolutePath, dexFile.absolutePath, null, this.javaClass.classLoader)
 ```


# 三，startActivity 流程初步分析

#### 1. Activity注册校验
加载出类之后，正常的代码我们已经可以执行了。但是在安卓中，我们想要显示界面，还要面对安卓对activity的注册校验。没有注册过的activity直接是无法打开的。会报错：
**Unable to find explicit activity class  \**\*; have you declared this activity in your AndroidManifest.xml?**
这是因为在我们startActivity的时候，`Instrumentation`中会进行校验。看源码可知，startActivity的流程如下:

![](https://api.onedrive.com/v1.0/shares/s!Aq4YQLDeCtuUgtB-vp5Pg4GoDdG3Xw/root/content)
对于`Activity`注册的校验，就是在`Instumentation.checkStartActivityResult`中进行的。

#### 2. Instrumentation 代理
我们从源码中得知，控制跳转的代码由`Instumentation`处理，所以我们可以对他做文章。
**我们想要正常打开我们的插件页面，用的是容器思想，建立一个容器activity，承载插件的页面，而要兼容startActivity 直接配置容器Class，只需要代理Instumentation对象，进行容器替换处理就可以了。**
我们先实现自己的`Instrumentation`代理类：

 ```Kotlin
 class PluginInstrumentation(var instrumentation:Instrumentation): Instrumentation() {
 
     @SuppressLint("DiscouragedPrivateApi")
     fun execStartActivity(
         who: Context?, contextThread: IBinder?, token: IBinder?, target: Activity?,
         intent: Intent?, requestCode: Int, options: Bundle?
     ): ActivityResult? {
         try {
             val pluginClazz = pluginClassLoader.loadClass(intent?.component?.className)
             var newIntent=intent;
             if (pluginClazz.superclass == IPluginActivityInterface::class.java) {
                 newIntent=Intent(who,HostActivity::class.java)
                 intent?.extras?.let {
                     newIntent.putExtras(it)
                 }
                 newIntent.putExtra(HostActivity.ARG_PLUGIN_CLASS_NAME,pluginClazz.name)
             }
             val execStartActivity: Method = Instrumentation::class.java.getDeclaredMethod(
                 "execStartActivity",
                 Context::class.java,
                 IBinder::class.java,
                 IBinder::class.java,
                 Activity::class.java,
                 Intent::class.java,
                 Int::class.javaPrimitiveType,
                 Bundle::class.java
             )
 
             return execStartActivity.invoke(instrumentation, who, contextThread, token, target, newIntent, requestCode, options) as ActivityResult
         } catch (e: Exception) {
             e.printStackTrace()
         }
         return null
     }
 }
 ```
在代理类中，我们重写了`execStartActivity`。对于插件Activity的Intent意图，替换成`HostActivity`。其余的插件相关页面逻辑，我们在`HostActivity`中进行调用。
这样就能绕过注册限制，展示插件中定义的页面和布局了。

> 需要注意的是，对于其中的插件类，我们是用专门的pluginClassLoader加载的。
#### 3. 替换instrumentation代理类
我们先看instumentation 的初始化代码。

 ```JavaScript
 final void attach(Context context, ActivityThread aThread,
         Instrumentation instr, IBinder token, int ident,
         Application application, Intent intent, ActivityInfo info,
         CharSequence title, Activity parent, String id,
         NonConfigurationInstances lastNonConfigurationInstances,
         Configuration config, String referrer, IVoiceInteractor voiceInteractor,
         Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
         IBinder shareableActivityToken) {
     attachBaseContext(context);
 ///...
     mInstrumentation = instr;
     }
 ```
可以看到`mInstrumentation`在attach方法中被赋值，我们想要代理改变量，就只需要在`attach`之后，利用反射重新赋值代理类就行了。
我们从源码中观察到，`attach`在`ActivityThread.performnLaunchActivity`中被调用，也在
`mInstrumentation.callActivityOnCreate`的调用之前，而onCreate的执行就在该调用链中。
所以我们只需要在Activity.onCreate中完成`mInstrumentation`的代理类替换就行了。

> 这一块代码我们通过对`Application`添加`activity`生命周期监听实现 ，代码如下，这一块代码的执行通过实现自己的`ContentProvider`实现。利用框架对provider的初始化机制，实现xml注册就无感注册监听，具体代码可以看源码中的`PluginContentProvider`类。
 ```Kotlin
 override fun onCreate(): Boolean {
     Utils.getApp().registerActivityLifecycleCallbacks(object :ActivityLifecycleCallbacks{
         @SuppressLint("DiscouragedPrivateApi")
         override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
             // 拿到原始的 mInstrumentation字段
             val mInstrumentationField: Field = Activity::class.java.getDeclaredField("mInstrumentation")
             mInstrumentationField.isAccessible = true
             // 创建代理对象
             val originalInstrumentation: Instrumentation = mInstrumentationField.get(activity) as Instrumentation
             mInstrumentationField.set(activity, PluginInstrumentation(originalInstrumentation))
         }
         //... 省略其他生命周期代码
     })
     return true
 }
 ```


# 四，利用宿主Activity替换掉实际打开的Activity

在完成`mInstrumentation`替换后，我们还需要完善我们的宿主容器HostActivtiy以及对应插件`Activity`的接口，完成以下几个目的：
1. 像打开正常的activity一样，打开插件activity
2. 占位Activity到插件Activity的转换
3. 无侵入式插件Activity开发
#### 完成插件Activity的抽象代理类(插件Activity只是写法和正常Activity一样，其实不是Activity的子类)，模拟正常Activity的生命周期
简单起见，我们就先只定义了`setContentView`，`onCreate`和`getIntent` 方法，其他生命周期方法在后续需要的时候在完善。重点是`registerHostActivity`方法，我们通过该方法注册宿主Activity，并在其各个生命周期中的各个方法，都交给`mHostActivity`代理完成。
 ```Kotlin
 abstract class IPluginActivityInterface {
     lateinit var mHostActivity: HostActivity
     fun registerHostActivity(hostActivity: HostActivity) {
         mHostActivity = hostActivity;
     }
 
     fun getIntent() = mHostActivity.intent
     open fun onCreate(savedInstanceState: Bundle?){}
     fun setContentView(layoutResID: Int) {
         mHostActivity.setContentView(layoutResID)
     }
 }
 ```
#### 在`HostActivity`中，实例化插件`IPluginActivityInterface`类
我们在`HostActivity` 的`onCreate`方法中取出之前Instumentation中传递的`ARG_PLUGIN_CLASS_NAME`，通过反射实例化出插件Activity
 ```Kotlin
 override fun onCreate(savedInstanceState: Bundle?) {
     super.onCreate(savedInstanceState)
     intent.getStringExtra(ARG_PLUGIN_CLASS_NAME)?.let {
         val clazz= pluginClassLoader.loadClass(it)
         pluginActivity = clazz.newInstance() as? IPluginActivityInterface
         pluginActivity?.registerHostActivity(this)
     }
     pluginActivity?.onCreate(savedInstanceState)
 }
 ```
这样就能正常加载插件Activity了。
# 四，compose ui开发和兼容
插件模块中的ui开发使用的是compose。对于compose的兼容非常简单，我们简单实现个setContent拓展方法，中转调用宿主activity的setContent方法就行了，如下：
 ```Kotlin
 fun IPluginActivityInterface.setContent(
     parent: CompositionContext? = null,
     content: @Composable () -> Unit
 ) {
     mHostActivity.setContent(parent,content)
 }
 ```
这样我们就能很正常的开发插件App了。如图所示
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d63b221467384f4f9be31583974b7f79~tplv-k3u1fbpfcp-zoom-1.image)
界面开发完成后，自然就是跳转了。
跳转到`PluginActivity`的代码，也和普通Activity没有区别，直接startActivity，代码如下：

 ```Nginx
 val intent = Intent().apply {
     component = ComponentName(context, "com.example.plugin_app.PluginsActivity")
 }
 context.startActivity(intent)
 ```
至此，完整的app，插件app开发已经实现。更多的细节在[源码中](https://github.com/guuguo/plugin_demo)
# 最后
吁了一口长气。
成功的完成了这个小文章的撰写。
最后放上示例demo的gif吧
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bdaf6ee06a954b52b8888927a6036a59~tplv-k3u1fbpfcp-zoom-1.image)