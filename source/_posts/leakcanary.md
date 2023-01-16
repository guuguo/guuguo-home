---
title: 【Andorid进阶】LeakCanary源码分析
---

> "内存优化会不会？知道怎么定位内存问题吗？"面试官和蔼地坐在小会议室的一侧，亲切地问有些拘谨地小张。
> 
> "就是...那个，用LeakCanary 检测一下泄漏，然后找到对应泄漏的地方，把错误的代码改一下，没回收的引用回收掉，优化下长短生命周期线程的依赖关系吧"
> 
> "那你了解LeakCanary 分析内存泄漏的原理吗？" 
> 
> "不好意思，平时没有注意去看过"
**小张心想:面试怎么老问这个，我只是个普通的菜鸟啊。**
# 前言
app性能优化总是开发中必不可少的一环，而其中内存优化又是重点之一。内存泄漏带来的内存溢出崩溃，内存抖动带来的卡顿不流畅。都在切切实实地影响着用户的体验。我们常常会使用LeakCanary来定位内存泄漏问题。也是时候来探索一下人家是怎么实现的了。
# 名词理解
`hprof` : hprof 文件是 Java 的 内存快照文件（Heap Profile 的缩写）,格式后缀为 .hprof，在leakCanary 中用于内存保存分析

`WeakReference` : 弱引用，当一个对象仅仅被weak reference（弱引用）指向, 而没有任何其他strong reference（强引用）指向的时候, 如果这时GC运行, 那么这个对象就会被回收，不论当前的内存空间是否足够，这个对象都会被回收。在leakCanary 中用于监测该回收的无用对象是否被释放。

`curtains`:Square 的另一个开源框架，Curtains 提供了用于处理 Android 窗口的集中式 API。
在leakCanary中用于监测`window` `rootView` 在detached 后的内存泄漏。
# 目录
本文主要从以下几点入手分析
1. 如何在项目中使用 `LeakCanary`工具
2. 官方原理说明
3. 默认如何监听Activity ,view ,fragment 和 viewmodel
4. Watcher.watch(object) 如何监听内存泄漏
5. 如何保存内存泄漏内存文件
6. 如何分析内存泄漏文件
7. 展示内存泄漏堆栈到ui中


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f881036c7cb149e7953bebb8eec58e8a~tplv-k3u1fbpfcp-zoom-1.image)

# 一，怎么用?
查看[官网文档](https://square.github.io/leakcanary/getting_started/)
可以看出使用方法非常简单,基础用法只需要添加相关依赖就行
```groovy
//（1）
debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.7'
```
> `debugImplementation `只在debug模式的编译和最终的debug apk打包时有效
**注(1):标注的代码中用了一行就实现了初始化，怎么做到的呢？**
通过查看源码可以看到，leakcanary 通过 `ContentProvider` 进行初始化，在`AppWatcherInstaller` 类的`oncreate`方法中调用了真正的初始化代码`AppWatcher.manualInstall(application)`。在`AndroidManifest.xml`中注册该`provider`,注册的`ContentProvider`会在 `application` 启动的时候自动回调 `oncreate`方法。
```kotlin
internal sealed class AppWatcherInstaller : ContentProvider() {
  /**[MainProcess] automatically sets up the LeakCanary code that runs in the main app process. */
  // (1)
  internal class MainProcess : AppWatcherInstaller()
  internal class LeakCanaryProcess : AppWatcherInstaller()
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    ///(2)
    AppWatcher.manualInstall(application)
    return true
  }
  //...
  }
```
说明一下源码中的数字标注

代码(1)中定义了两个内部类继承自 `AppWatcherInstaller`。当用户额外依赖 `leakcanary-android-process` 模块的时候，自动在` process=":leakcanary"` 也注册该provider。
> 代码参见 `leakcanary-android-process` 模块中的AndroidManifest.xml
代码(2),这是真正的初始化代码注册入口


# 二，官方阐述
## 官方说明
> 本小节来自于[官方网站](https://square.github.io/leakcanary/fundamentals-how-leakcanary-works/)的工作原理说明精简
安装 LeakCanary 后，它会通过 4 个步骤自动检测并报告内存泄漏：
1. #### 检测被持有的对象
LeakCanary 挂钩到 Android 生命周期以自动检测活动和片段何时被销毁并应进行垃圾收集。这些被销毁的对象被传递给一个`ObjectWatcher`，它持有对它们的[弱引用](https://en.wikipedia.org/wiki/Weak_reference)。
可以主动观察一个不再需要的对象比如一个 dettached view 或者 已经销毁的 presenter
```
AppWatcher.objectWatcher.watch(myDetachedView, "View was detached")
```
如果`ObjectWatcher`在**等待 5 秒**并运行垃圾收集后没有清除持有的弱引用，则被监视的对象被认为是**保留的**，并且可能会泄漏。LeakCanary 将此记录到 Logcat：
```
D LeakCanary: Watching instance of com.example.leakcanary.MainActivity
  (Activity received Activity#onDestroy() callback) 

... 5 seconds later ...

D LeakCanary: Scheduling check for retained objects because found new object
  retained
```
1. #### Dumping the heap 转储堆信息到文件中
当保留对象的数量达到阈值时，LeakCanary 将 Java 内存快照 dumping **转储**到 Android 文件系统上的`.hprof`文件（**堆内存快照**）中。转储堆会在短时间内冻结应用程序,并展示下图的吐司：
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d28a9a9db994cf1a4df19746188d9e9~tplv-k3u1fbpfcp-zoom-1.image)

1. #### 分析堆内存
LeakCanary使用[Shark](https://square.github.io/leakcanary/shark/)解析`.hprof`文件并在该内存快照文件中定位被保留的泄漏对象。
对于每个保留对象，LeakCanary 找到该对象的引用路径，该引用阻止了垃圾收集器对它的回收。也就是**泄漏跟踪**。
LeakCanary**为每个泄漏跟踪**创建一个**签名** （对持有的引用属性进行相加做sha1Hash），并将具有相同签名的泄漏（即由相同错误引起的泄漏）组合在一起。如何创建签名和通过签名分组有待后文分析。

1. #### 分类内存泄漏
LeakCanary 将它在您的应用中发现的泄漏分为两类：**Application Leaks (应用程序泄漏)**和**Library Leaks(库泄漏)**。一个**Library Leaks**是由已知的第三方库导致的，你没有控制权。这种泄漏正在影响您的应用程序，但不幸的是，修复它可能不在您的控制范围内，因此 LeakCanary 将其分离出来。
这两个类别**分开**在**Logcat结果中**打印：
```
====================================
HEAP ANALYSIS RESULT
====================================
0 APPLICATION LEAKS
====================================
1 LIBRARY LEAK
...
┬───
│ GC Root: Local variable in native code
│
...
```
LeakCanary在其泄漏列表展示中会将其用`Library Leak` 标签标记：
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21246810194243219b72a1bf8eec057d~tplv-k3u1fbpfcp-zoom-1.image)
LeakCanary 附带一个已知泄漏的数据库，它通过引用名称的模式匹配来识别。例如：
```shell
Leak pattern: instance field android.app.Activity$1#this$0
Description: Android Q added a new IRequestFinishCallback$Stub class [...]
┬───
│ GC Root: Global variable in native code
│
├─ android.app.Activity$1 instance
│    Leaking: UNKNOWN
│    Anonymous subclass of android.app.IRequestFinishCallback$Stub
│    ↓ Activity$1.this$0
│                 ~~~~~~
╰→ com.example.MainActivity instance
```
**Library Leaks 通常我们都无力对齐进行修复**
您可以在[AndroidReferenceMatchers](https://github.com/square/leakcanary/blob/main/shark-android/src/main/java/shark/AndroidReferenceMatchers.kt#L49)类中查看已知泄漏的完整列表。如果您发现无法识别的 Android SDK 泄漏，请[报告](https://square.github.io/leakcanary/faq/#can-a-leak-be-caused-by-the-android-sdk)。您还可以[自定义已知库泄漏的列表](https://square.github.io/leakcanary/recipes/#matching-known-library-leaks)。

# 三，监测activity，fragment，rootView和viewmodel
前面提到初始化的代码如下,所以我们 查看manualInstall 的内部细节。
```kotlin
///初始化代码
AppWatcher.manualInstall(application)

///AppWatcher 的 manualInstall 代码
@JvmOverloads
fun manualInstall(
  application: Application,
  retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
  watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
) {
   //*******检查是否为主线程********/
  checkMainThread()
  if (isInstalled) {
    throw IllegalStateException(
      "AppWatcher already installed, see exception cause for prior install call", installCause
    )
  }
  check(retainedDelayMillis >= 0) {
    "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
  }
  installCause = RuntimeException("manualInstall() first called here")
  this.retainedDelayMillis = retainedDelayMillis
  if (application.isDebuggableBuild) {
    LogcatSharkLog.install()
  }
  // Requires AppWatcher.objectWatcher to be set
  ///(2)
  LeakCanaryDelegate.loadLeakCanary(application)
  ///(1)
  watchersToInstall.forEach {
    it.install()
  }
}
```
AppWatcher 作为Android 平台使用 ObjectWatcher 封装的api中心。自动安装配置默认的监听。
以上代码关键的地方用数字标出了
## （1）Install 默认的监听观察
标注(1)处的代码执行了 InstallableWatcher 的 install 操作，在调用的时候并没有传递 `watchersToInstall` 参数，所以使用的是 appDefaultWatchers(application)。该处代码在下面，提供了 四个默认监听的Watcher
```kotlin
fun appDefaultWatchers(
  application: Application,
  ///(1.1)
  reachabilityWatcher: ReachabilityWatcher = objectWatcher
): List<InstallableWatcher> {
  return listOf(
    ///(1.2)
    ActivityWatcher(application, reachabilityWatcher),
    ///(1.3)
    FragmentAndViewModelWatcher(application, reachabilityWatcher),
    ///(1.4)
    RootViewWatcher(reachabilityWatcher),
    ///(1.5)
    ServiceWatcher(reachabilityWatcher)
  )
}
```
用数字标出的四个我们逐个分析
## （1.1） reachabilityWatcher 参数
标注(1.1)处的代码是一个 `ReachabilityWatcher`  参数，`reachabilityWatcher` 在后续的四个实例创建时候都有用到，代码中可以看到`reachabilityWatcher`实例是`AppWatcher` 的成员变量：`objectWatcher`，对应的实例化代码如下。
```kotlin
/**
 * The [ObjectWatcher] used by AppWatcher to detect retained objects.
 * Only set when [isInstalled] is true.
 */
val objectWatcher = ObjectWatcher(
  clock = { SystemClock.uptimeMillis() },
  checkRetainedExecutor = {
    check(isInstalled) {
      "AppWatcher not installed"
    }
    mainHandler.postDelayed(it, retainedDelayMillis)
  },
  isEnabled = { true }
)
```
可以看到objectWatcher 是一个 ObjectWatcher对象，该对象负责检测持有对象的泄漏情况，会在第三小节进行分析。
回到 ActivityWatcher 实例的创建，继续往下看标注的代码

## （1.2）ActivityWatcher 实例 完成Activity 实例的监听
回到之前，标注(1.2)处的代码创建了`ActivityWatcher`实例，并在install 的时候安装，查看ActivityWatcher 类的源码，看监听Activity泄漏是怎么实现的
```kotlin
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
     //(1.2.1) 通过动态代理，构造出生命周期回调的实现类 
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        //(1.2.3)
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
    //(1.2.3)
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
```
### (1.2.1) lifecycleCallbacks 实例
标注(1.2.1)处的代码创建了ActivityLifecycleCallbacks实例，该实例实现了Application.ActivityLifecycleCallbacks。通过 `by ``*noOpDelegate*``()` ，利用动态代理实现了其他回调方法，感兴趣的可以查看 *noOpDelegate 的源码*
### (1.2.2) activity监听器的 install 方法
标注(1.2.2)处的代码是初始化的主要代码，该方法很简单，就是在application的 中注册 lifecycleCallbacks，在activity 被destroy 的时候会走到其中实现的方法
### (1.2.3) 监听activity 的 onActivityDestroyed 回调
标注(1.2.3)处的代码是初始化的主要代码，在 activity被销毁的时候，回调该方法，在其中检查该实例是否有泄漏，调用AppWatcher.objectWatcher. expectWeaklyReachable 方法，在其中完成activity的泄漏监测。 这时候又回到了 1.1 提到的 ObjectWatcher源码，相关分析看第四节 。
### (1.2-end)Activity监测相关总结
这样ActivityInstaller 就看完了，了解了Activity 的初始化代码以及加入监听的细节。总结一下分为如下几步：
1. 调用ActivityInstaller.install 初始化方法
2. 在Application 注册ActivityLifecycleCallbacks
3. 在所有activity onDestroy的时候调用ObjectWatcher的 expectWeaklyReachable方法，检查过五秒后activity对象是否有被内存回收。标记内存泄漏。下一节分析。
4. 检测到内存泄漏的后续操作。后文分析。


##  (1.3) FragmentAndViewModelWatcher  监测 Fragment 和Viewodel实例
（1.3）处是创建了 FragmentAndViewModelWatcher 实例。监测fragment和viewmodel的内存泄漏。
> 该类实现了 `SupportFragment`和 `androidxFragment`以及`androidO` 的兼容，作为sdk开发来说，这种 兼容方式可以学习一下。
```kotlin
private val lifecycleCallbacks =
  object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
    override fun onActivityCreated(
      activity: Activity,
      savedInstanceState: Bundle?
    ) {
      for (watcher in fragmentDestroyWatchers) {
        watcher(activity)
      }
    }
  }

override fun install() {
  application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
}
```
和`ActivityWatcher` 同样的，install是注册了生命周期监听。不过是在对每个 activity create 的时候，交给 `fragmentDestroyWatchers` 元素们监听。所以 `fragmentDestroyWatchers`才是真正的`fragment`和`viewmodel` 监听者。
接下来看 fragmentDestroyWatchers 的元素们创建:

```kotlin
private val fragmentDestroyWatchers: List<(Activity) -> Unit> = run {
  val fragmentDestroyWatchers = mutableListOf<(Activity) -> Unit>()

   //(1.3.1) android框架自带的fragment泄漏监测支持从 AndroidO(26)开始。
  if (SDK_INT >= O) {
    fragmentDestroyWatchers.add(
      AndroidOFragmentDestroyWatcher(reachabilityWatcher)
    )
  }
   //(1.3.2)
  getWatcherIfAvailable(
    ANDROIDX_FRAGMENT_CLASS_NAME,
    ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
    reachabilityWatcher
  )?.let {
    fragmentDestroyWatchers.add(it)
  }
   //(1.3.3)
  getWatcherIfAvailable(
    ANDROID_SUPPORT_FRAGMENT_CLASS_NAME,
    ANDROID_SUPPORT_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
    reachabilityWatcher
  )?.let {
    fragmentDestroyWatchers.add(it)
  }
  fragmentDestroyWatchers
}
```
可以看到内部创建了AndroidOFragmentDestroyWatcher 来针对Fragment 进行监听。原理是利用在 `FragmentManager` 中注册 `FragmentManager.FragmentLifecycleCallbacks` 来监听`fragment` 和 `fragment.view` 以及`viewmodel` 的实例泄漏。
从[官方文档](https://developer.android.com/reference/android/app/FragmentManager.FragmentLifecycleCallbacks?hl=zh-cn)可知，android内部的 fragment 在Api 26中才添加。所以LeakCanary针对于android框架自带的`fragment`泄漏监测支持也是从 AndroidO(26)开始,见代码(1.3.1)。
标注的 1.3.1，1.3.2，1.3.3 实例化的三个Wathcer 分别是 `AndroidOFragmentDestroyWatcher`,`AndroidXFragmentDestroyWatcher`,`AndroidSupportFragmentDestroyWatcher`。内部实现代码大同小异，通过反射实例化不同的Watcher实现了androidX 和support 以及安卓版本间的兼容。
### (1.3.1) AndroidOFragmentDestroyWatcher 实例
(1.3.1)处的代码添加了一个androidO的观察者实例。详情见代码，因为实现大同小异，分析参考1.3.2.
### (1.3.2) AndroidXFragmentDestroyWatcher 实例
(1.3.2)处的代码 调用 `getWatcherIfAvailable` 通过反射创建了`AndroidXFragmentDestroyWatcher`实例，如果不存在Androidx库则返回null。
现在跳到 `AndroidXFragmentDestroyWatcher` 的源码分析
```kotlin
internal class AndroidXFragmentDestroyWatcher(
  private val reachabilityWatcher: ReachabilityWatcher
) : (Activity) -> Unit {

  private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

    override fun onFragmentCreated(
      fm: FragmentManager,
      fragment: Fragment,
      savedInstanceState: Bundle?
    ) {
    //(1.3.2.1)初始化 ViewModelClearedWatcher 
      ViewModelClearedWatcher.install(fragment, reachabilityWatcher)
    }

    override fun onFragmentViewDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
     //监测 fragment.view 的泄漏情况
      val view = fragment.view
      if (view != null) {
        reachabilityWatcher.expectWeaklyReachable(
          view, "${fragment::class.java.name} received Fragment#onDestroyView() callback " +
          "(references to its views should be cleared to prevent leaks)"
        )
      }
    }

    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      //监测 fragment 的泄漏情况
      reachabilityWatcher.expectWeaklyReachable(
        fragment, "${fragment::class.java.name} received Fragment#onDestroy() callback"
      )
    }
  }

 ///初始化，注册fragmentLifecycleCallbacks
  override fun invoke(activity: Activity) {
    if (activity is FragmentActivity) {
      val supportFragmentManager = activity.supportFragmentManager
      supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
      //注册activity的 viewModel 监听回调
      ViewModelClearedWatcher.install(activity, reachabilityWatcher)
    }
  }
}
```
通过源码可以看到，初始化该watcher是通过以下几步。
1. `FragmentManager.registerFragmentLifecycleCallbacks `注册监听回调
2. `ViewModelClearedWatcher.install` 初始化了对于`activity.viewModel`的监听
3. 在回调`onFragmentCreated` 中回调中使用`ViewModelClearedWatcher.install`注册了对于`fragment.viewModel`的监听。
4. 在 `onFragmentViewDestroyed` 监听 `fragment.view` 的泄漏
5. 在 `onFragmentDestroyed` 监听 `fragment`的泄漏。
监听方法和ActivityWatcher大同小异，不同是多了个 `ViewModelClearedWatcher.install` 。现在分析这一块的源码，也就是标注中的 (1.3.2.1)。
```kotlin
//该watcher 继承了ViewModel，生命周期被 ViewModelStoreOwner 管理。
internal class ViewModelClearedWatcher(
  storeOwner: ViewModelStoreOwner,
  private val reachabilityWatcher: ReachabilityWatcher
) : ViewModel() {

  private val viewModelMap: Map<String, ViewModel>?

  init {
    //(1.3.2.3)通过反射获取所有的 store 存储的所有viewModelMap
    viewModelMap = try {
      val mMapField = ViewModelStore::class.java.getDeclaredField("mMap")
      mMapField.isAccessible = true
      @Suppress("UNCHECKED_CAST")
      mMapField[storeOwner.viewModelStore] as Map<String, ViewModel>
    } catch (ignored: Exception) {
      null
    }
  }

  override fun onCleared() {
    ///(1.3.2.4) viewmodle 被清理释放的时候回调，检查所有viewmodle 是否会有泄漏
    viewModelMap?.values?.forEach { viewModel ->
      reachabilityWatcher.expectWeaklyReachable(
        viewModel, "${viewModel::class.java.name} received ViewModel#onCleared() callback"
      )
    }
  }

  companion object {
    fun install(
      storeOwner: ViewModelStoreOwner,
      reachabilityWatcher: ReachabilityWatcher
    ) {
      val provider = ViewModelProvider(storeOwner, object : Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel?> create(modelClass: Class<T>): T =
          ViewModelClearedWatcher(storeOwner, reachabilityWatcher) as T
      })
      ///(1.3.2.2) 获取ViewModelClearedWatcher实例
      provider.get(ViewModelClearedWatcher::class.java)
    }
  }
}
```
通过代码，可以看到viewModel的泄漏监测是通过创建一个新的viewModel实例来实现。在该实例的onCleared处监听storeOwner的其余 viewModel 是否有泄漏。标注出的代码逐一分析：
#### (1.3.2.2 ) 处代码：
获取ViewModelClearedWatcher 实例，在自定义的 Factory中传入storeOwner 和 reachabilityWatcher。
#### (1.3.2.3 ) 处代码：
通过反射获取`storeOwner` 的`viewModelMap`
#### (1.3.2.4 ) 处代码：
在ViewModel完成使命OnClear的时候，开始监测storeOwner旗下所有ViewModel的内存泄漏情况。
### (1.3-end)Fragment 和 viewmodel 监测泄漏总结：
监测方式都是通过`ObjectWatcher`的 `expectWeaklyReachable` 方法进行。fragment 利用`FragmentLifecyclerCallback`回调注册实现，ViewModel 则是在对应StoreOwner下创建了监测viewModel来实现生命周期的响应。
其中我们也能学习到通过反射来创建对应的平台兼容实现对象方式。以及借助创建viewModel来监听其余ViewModel生命周期的想法。
##  (1.4) RootViewWatcher 的源码分析
默认的四个Watcher中，来到了接下来的 RootViewWatcher。window rootview 监听依赖了squre自家的Curtains框架。
```groovy
implementation "com.squareup.curtains:curtains:1.0.1"
```
类的关键源码如下:
```kotlin
 private val listener = OnRootViewAddedListener { rootView ->
 //如果是 Dialog TOOLTIP, TOAST, UNKNOWN 等类型的windows 
 //trackDetached 为true
    if (trackDetached) {
      rootView.addOnAttachStateChangeListener(object : OnAttachStateChangeListener {

        val watchDetachedView = Runnable {
          reachabilityWatcher.expectWeaklyReachable(
            rootView, "${rootView::class.java.name} received View#onDetachedFromWindow() callback"
          )
        }

        override fun onViewAttachedToWindow(v: View) {
          mainHandler.removeCallbacks(watchDetachedView)
        }

        override fun onViewDetachedFromWindow(v: View) {
          mainHandler.post(watchDetachedView)
        }
      })
    }
  }

  override fun install() {
    Curtains.onRootViewsChangedListeners += listener
  }

  override fun uninstall() {
    Curtains.onRootViewsChangedListeners -= listener
  }
}
```
看到关键代码，就是 在Curtains中添加`onRootViewsChangedListeners` 监听器。当windowsType类型为 `**Dialog**` `***TOOLTIP***`**,** `***TOAST***`**,**或者未知的时候 ，在 `onViewDetachedFromWindow` 的时候监听泄漏情况。
Curtains中的监听器会在windows rootView 变化的时候被全局调用。Curtains是squareup 的另一个开源库，Curtains 提供了用于处理 Android 窗口的集中式 API。具体移步他的[官方仓库](https://github.com/square/curtains)。
## (1.5) ServiceWatcher 监听Service内存泄漏
接下来就是AppWatcher中的最后一个Watcher。 ServiceWatcher。代码比较长，截取关键点分析。
### （1.5.1）先看成员变量 `activityThreadServices` ：
```kotlin
private val servicesToBeDestroyed = WeakHashMap<IBinder, WeakReference<Service>>()
private val activityThreadClass by lazy { Class.forName("android.app.ActivityThread") }
private val activityThreadInstance by lazy {
  activityThreadClass.getDeclaredMethod("currentActivityThread").invoke(null)!!
}

private val activityThreadServices by lazy {
  val mServicesField =
    activityThreadClass.getDeclaredField("mServices").apply { isAccessible = true }

  @Suppress("UNCHECKED_CAST")
  mServicesField[activityThreadInstance] as Map<IBinder, Service>
}
```
`activityThreadServices` 是个装了所有<IBinder, Service> 对的Map。代码中可以看到很粗暴地，直接通过反射从`ActivityThread`实例中拿到了mServices 变量 。赋值给`activityThreadServices`。
源码中有多个swap操作，在install的时候执行，主要目的是将原来的一些service相关生命周期回调加上一些钩子，用来监测内存泄漏，并且会在unInstall的时候给换回来。
### （1.5.2）swapActivityThreadHandlerCallback ：
拿到ActivityThread 的Handler，将其回调的 handleMessage，换成加了料的Handler.Callback，加料代码如下
```kotlin
Handler.Callback { msg ->
  if (msg.what == STOP_SERVICE) {
    val key = msg.obj as IBinder
    activityThreadServices[key]?.let {
      onServicePreDestroy(key, it)
    }
  }
  mCallback?.handleMessage(msg) ?: false
}
```
代码中可以看到，主要是对于 STOP_SERVICE 的操作做了一个钩子，在之前执行 onServicePreDestroy。主要作用是为该service 创建一个弱引用，并且加到`servicesToBeDestroyed[token]` 中 。
### （1.5.3）然后再看 `swapActivityManager` 方法。
该方法完成了将`ActivityManager`替换成`IActivityManager`的一个动态代理类。代码如下：
```kotlin
Proxy.newProxyInstance(
  activityManagerInterface.classLoader, arrayOf(activityManagerInterface)
) { _, method, args ->
//private const val METHOD_SERVICE_DONE_EXECUTING = "serviceDoneExecuting"
  if (METHOD_SERVICE_DONE_EXECUTING == method.name) {
    val token = args!![0] as IBinder
    if (servicesToBeDestroyed.containsKey(token)) {
       ///(1.5.3)
      onServiceDestroyed(token)
    }
  }
  try {
    if (args == null) {
      method.invoke(activityManagerInstance)
    } else {
      method.invoke(activityManagerInstance, *args)
    }
  } catch (invocationException: InvocationTargetException) {
    throw invocationException.targetException
  }
}
```
代码所示，替换后的ActivityManager 在调用`serviceDoneExecuting` 方法的时候添加了个钩子，如果该service在之前加入的`servicesToBeDestroyed` map中，则调用`onServiceDestroyed` 监测该service内存泄漏。
### （1.5.4）代码的`onServiceDestroyed`具体代码如下
```kotlin
private fun onServiceDestroyed(token: IBinder) {
  servicesToBeDestroyed.remove(token)?.also { serviceWeakReference ->
    serviceWeakReference.get()?.let { service ->
      reachabilityWatcher.expectWeaklyReachable(
        service, "${service::class.java.name} received Service#onDestroy() callback"
      )
    }
  }
}
```
这里面的代码很熟悉，和之前监测activity等是一样的。
回到swapActivityManager方法，看代理ActivityManager的具体类型。
可以看到代理的对象如下面代码所示，根据版本不同可能是`ActivityManager` 实例或者是`ActivityManagerNative`实例。
代理的接口是 `Class.forName("android.app.IActivityManager")`。
```kotlin
val (className, fieldName) = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
  "android.app.ActivityManager" to "IActivityManagerSingleton"
} else {
  "android.app.ActivityManagerNative" to "gDefault"
}
```
### （1.5-end）Service 泄漏监测总结
总结一下，service的泄漏分析通过加钩子的方式，对一些系统执行做了监听。主要分为以下几步：
1. 获取ActivityThread中mService变量，得到service实例的引用
2. 通过swapActivityThreadHandlerCallback 在ActivityThread 的 Handler.sendMessage 中添加钩子，在执行到`msg.what == STOP_SERVICE` 的时候
# 四，ObjectWatcher 保留对象检查分析
我们转到 ObjectWatcher 的 expectWeaklyReachable 方法看看
```kotlin
@Synchronized override fun expectWeaklyReachable(
  watchedObject: Any,
  description: String
) {
   //是否启用 , AppWatcher 持有的ObjectWatcher 默认是启用的
  if (!isEnabled()) {
    return
  }
  ///移除之前已经被回收的监听对象
  removeWeaklyReachableObjects()
  val key = UUID.randomUUID()
    .toString()
  val watchUptimeMillis = clock.uptimeMillis()
   //(1) 创建弱引用
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  SharkLog.d {
    "Watching " +
      (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
      (if (description.isNotEmpty()) " ($description)" else "") +
      " with key $key"
  }

  watchedObjects[key] = reference
  checkRetainedExecutor.execute {
    //(2)
    moveToRetained(key)
  }
}
```
继续分析源码中标注的地方。
### (1) 创建弱引用
标注(1.2.4)处的代码是初始化的主要代码，创建要观察对象的弱引用，传入queue 作为gc 后的对象信息存储队列，`WeakReference` 中，当持有对象呗gc的时候，会将其包装对象压入队列中。可以在后续对该队列进行观察。
### (2) moveToRetained(key)，检查对应key对象的保留
作为Executor的runner 执行，在AppWatcher中，默认延迟五秒后执行该方法
查看源码分析

```kotlin
@Synchronized private fun moveToRetained(key: String) {
///移除已经被回收的观察对象
  removeWeaklyReachableObjects()
  val retainedRef = watchedObjects[key]
  if (retainedRef != null) {
  //记录泄漏时间
    retainedRef.retainedUptimeMillis = clock.uptimeMillis()
    //回调泄漏监听
    onObjectRetainedListeners.forEach { it.onObjectRetained() }
  }
}
```
从上述代码可知，ObjectWatcher 监测内存泄漏总共有以下几步
1. 清除已经被内存回收的监听对象
2. 创建弱引用，传入 ReferenceQueue 作为gc 信息保存队列
3. 在延迟指定的时间后，再次检查针对的对象是否被回收（通过检查ReferenceQueue队列内有无该WeakReference实例）
4. 检测到对象没有被回收后,回调 `onObjectRetainedListeners` 们的  `onObjectRetained`
# 五，dumpHeap，怎么个DumpHeap流程
## （1.1）objectWatcher 添加 OnObjectRetainedListeners 监听
回到最初AppWatcher的 manualInstall 方法。
可以看到其中执行了loadLeakCanary 方法。
代码如下:
```kotlin
///(2)
  LeakCanaryDelegate.loadLeakCanary(application)
 //反射获取InternalLeakCanary实例
  val loadLeakCanary by lazy {
  try {
    val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
    leakCanaryListener.getDeclaredField("INSTANCE")
      .get(null) as (Application) -> Unit
  } catch (ignored: Throwable) {
    NoLeakCanary
  }
}
```
该方法通过反射获取了 InternalLeakCanary 的静态实例。
并且调用了他的 `invoke(application: Application)`方法，所以我们接下来看InternalLeakCanary的该方法：
```kotlin
override fun invoke(application: Application) {
  _application = application

  checkRunningInDebuggableBuild()
  //（1.2）添加 addOnObjectRetainedListener
  AppWatcher.objectWatcher.addOnObjectRetainedListener(this)

  val heapDumper = AndroidHeapDumper(application, createLeakDirectoryProvider(application))
    //Gc触发器
  val gcTrigger = GcTrigger.Default

  val configProvider = { LeakCanary.config }

  val handlerThread = HandlerThread(LEAK_CANARY_THREAD_NAME)
  handlerThread.start()
  val backgroundHandler = Handler(handlerThread.looper)
///（1.3）
  heapDumpTrigger = HeapDumpTrigger(
    application, backgroundHandler, AppWatcher.objectWatcher, gcTrigger, heapDumper,
    configProvider
  )
  ///（1.4） 添加application前后台变化监听
  application.registerVisibilityListener { applicationVisible ->
    this.applicationVisible = applicationVisible
    heapDumpTrigger.onApplicationVisibilityChanged(applicationVisible)
  }
  //（1.5）
  registerResumedActivityListener(application)
  //（1.6）
  addDynamicShortcut(application)

   // 6 判断是否应该DumpHeap 
  // We post so that the log happens after Application.onCreate()
  mainHandler.post {
    // https://github.com/square/leakcanary/issues/1981
    // We post to a background handler because HeapDumpControl.iCanHasHeap() checks a shared pref
    // which blocks until loaded and that creates a StrictMode violation.
    backgroundHandler.post {
      SharkLog.d {
        when (val iCanHasHeap = HeapDumpControl.iCanHasHeap()) {
          is Yup -> application.getString(R.string.leak_canary_heap_dump_enabled_text)
          is Nope -> application.getString(
            R.string.leak_canary_heap_dump_disabled_text, iCanHasHeap.reason()
          )
        }
      }
    }
  }
}
```
我们看到初始化的时候做了这么6步
- （1.2） 将自己加入到ObjectWatcher 的对象异常持有监听器中
- （1.3）创建内存快照转储触发器 HeapDumpTrigger
- （1.4）监听application 前后台变动，并且记录来到后台时间，便于LeakCanary 针对刚刚切入后台的一些destroy操作做泄漏监测
- （1.5）注册activity生命周期回调，获取当前resumed的activity实例
- （1.6）添加动态的桌面快捷入口
- （1.7）在异步线程中，判断是否处于可dumpHeap的状态，如果处于触发一次内存泄漏检查
其中最重要的是 1.2，我们重点分析作为`ObjectRetainedListener` 他在回调中做了哪些工作。
## （1.2）添加对象异常持有监听
可以看到代码(1.2)，在objectWatcher将自己加入到泄漏监测回调中。
当ObjectWatcher监测到对象依然被异常持有的时候，会回调 onObjectRetained 方法。
从源码中可知，其中调用了 `heapDumpTrigger`的  `scheduleRetainedObjectCheck`方法，
代码如下。

```kotlin
fun scheduleRetainedObjectCheck() {
  if (this::heapDumpTrigger.isInitialized) {
    heapDumpTrigger.scheduleRetainedObjectCheck()
  }
}
```
`HeapDumpTrigger` 顾名思义，就是内存快照转储的触发器。在回调中最终调用了`HeapDumpTrigger` 的 `checkRetainedObjects`方法来检查内存泄漏。
## （1.3）检查内存泄漏checkRetainedObjects
```kotlin
private fun checkRetainedObjects() {
  val iCanHasHeap = HeapDumpControl.iCanHasHeap()

  val config = configProvider()
 //省略一些代码，主要是判断 iCanHasHeap。
 //如果当前处于不dump内存快照的状态，就先不处理。如果有新的异常持有对象被发现则发送通知提示
 //%d retained objects, tap to dump heap
  /** ...*/

  var retainedReferenceCount = objectWatcher.retainedObjectCount

   //主动触发gc
  if (retainedReferenceCount > 0) {
    gcTrigger.runGc()
    //重新获取异常持有对象
    retainedReferenceCount = objectWatcher.retainedObjectCount
  }
  //如果泄漏数量小于阈值，且app在前台，或者刚转入后台，就展示泄漏通知，并先返回
  if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

//如果泄漏数量到达dumpHeap要求，继续往下
   ///转储内存快照在  WAIT_BETWEEN_HEAP_DUMPS_MILLIS （默认60秒）只会触发一次，如果之前刚触发过，就先不生成内存快照，直接发送通知了事。
//省略转储快照时机判断,不满足的话会提示 Last heap dump was less than a minute ago
/**...*/

  dismissRetainedCountNotification()
  val visibility = if (applicationVisible) "visible" else "not visible"
  ///转储内存快照
  dumpHeap(
    retainedReferenceCount = retainedReferenceCount,
    retry = true,
    reason = "$retainedReferenceCount retained objects, app is $visibility"
  )
}
```

这一块也可以看出检测是否需要dumpHeap分为4步。
1. 如果没有检测到异常持有的对象，返回
2. 如果有异常对象，主动触发gc
3. 如果还有异常对象，就是内存泄漏了。
4. 判断泄漏数量是否到达需要dump的地步
5. 判断一分钟内是否叫进行过dump了
6. dumpHeap
前面都是判断代码，关键重点在于dumpHeap方法
## （1.4）dumpHeap 转储内存快照
```kotlin
private fun dumpHeap(
  retainedReferenceCount: Int,
  retry: Boolean,
  reason: String
) {
  saveResourceIdNamesToMemory()
  val heapDumpUptimeMillis = SystemClock.uptimeMillis()
  KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
  when (val heapDumpResult = heapDumper.dumpHeap()) {
    is NoHeapDump -> {
     //省略 dump失败，等待重试代码和发送失败通知代码
    }
    is HeapDump -> {
      lastDisplayedRetainedObjectCount = 0
      lastHeapDumpUptimeMillis = SystemClock.uptimeMillis()
      ///清除 objectWatcher 中，在heapDumpUptimeMillis之前持有的对象，也就是已经dump的对象
      objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
       // 发送文件到HeapAnalyzerService解析
      HeapAnalyzerService.runAnalysis(
        context = application,
        heapDumpFile = heapDumpResult.file,
        heapDumpDurationMillis = heapDumpResult.durationMillis,
        heapDumpReason = reason
      )
    }
  }
}
```
HeapDumpTrigger#dumpHeap中调用到了 AndroidHeapDumper#dumpHeap方法。
并且在dump后马上调用 `HeapAnalyzerService.runAnalysis` 进行内存分析工作，该方法在下一节分析。先看AndroidHeapDumper#dumHeap源码
```kotlin
override fun dumpHeap(): DumpHeapResult {
//创建新的hprof 文件
  val heapDumpFile = leakDirectoryProvider.newHeapDumpFile() ?: return NoHeapDump

  val waitingForToast = FutureResult<Toast?>()
  ///展示dump吐司
  showToast(waitingForToast)

  ///如果展示吐司时间超过五秒，就不dump了
  if (!waitingForToast.wait(5, SECONDS)) {
    SharkLog.d { "Did not dump heap, too much time waiting for Toast." }
    return NoHeapDump
  }

  //省略dumpHeap通知栏提示消息代码
  val toast = waitingForToast.get()

  return try {
    val durationMillis = measureDurationMillis {
    //调用DumpHprofData
      Debug.dumpHprofData(heapDumpFile.absolutePath)
    }
    if (heapDumpFile.length() == 0L) {
      SharkLog.d { "Dumped heap file is 0 byte length" }
      NoHeapDump
    } else {
      HeapDump(file = heapDumpFile, durationMillis = durationMillis)
    }
  } catch (e: Exception) {
    SharkLog.d(e) { "Could not dump heap" }
    // Abort heap dump
    NoHeapDump
  } finally {
    cancelToast(toast)
    notificationManager.cancel(R.id.leak_canary_notification_dumping_heap)
  }
}
```
在该方法内，最终调用 `Debug.dumpHprofData `方法 完成`hprof` 快照的生成。
# 六，分析内存 HeapAnalyzerService
上面代码分析中可以看到，在dumpHeap后紧跟着就是启动内存分析服务的方法。
现在我们跳转到`HeapAnalyzerService`的源码处。
```kotlin
override fun onHandleIntentInForeground(intent: Intent?) {
     //省略参数获取代码
  val config = LeakCanary.config
  val heapAnalysis = if (heapDumpFile.exists()) {
    analyzeHeap(heapDumpFile, config)
  } else {
    missingFileFailure(heapDumpFile)
  }
   //省略完善分析结果属性的代码
  onAnalysisProgress(REPORTING_HEAP_ANALYSIS)
  config.onHeapAnalyzedListener.onHeapAnalyzed(fullHeapAnalysis)
}
```
可以看到重点在于 analyzeHeap，其中调用了 HeapAnalyzer#analyze
HeapAnalyzer 类位于shark模块中。
## （1）HeapAnalyzer#analyze
内存分析方法代码如下：
```kotlin
fun analyze(
  heapDumpFile: File,
  leakingObjectFinder: LeakingObjectFinder,
  referenceMatchers: List<ReferenceMatcher> = emptyList(),
  computeRetainedHeapSize: Boolean = false,
  objectInspectors: List<ObjectInspector> = emptyList(),
  metadataExtractor: MetadataExtractor = MetadataExtractor.NO_OP,
  proguardMapping: ProguardMapping? = null
): HeapAnalysis {
  
 //省略内存快照文件不存在的处理代码

  return try {
    listener.onAnalysisProgress(PARSING_HEAP_DUMP)
   ///io读取 内存快照
    val sourceProvider = ConstantMemoryMetricsDualSourceProvider(FileSourceProvider(heapDumpFile))
    sourceProvider.openHeapGraph(proguardMapping).use { graph ->
      val helpers =
        FindLeakInput(graph, referenceMatchers, computeRetainedHeapSize, objectInspectors)
     //关键代码：在此处找到泄漏的结果以及其对应调用栈
      val result = helpers.analyzeGraph(
        metadataExtractor, leakingObjectFinder, heapDumpFile, analysisStartNanoTime
      )
      val lruCacheStats = (graph as HprofHeapGraph).lruCacheStats()
      ///io读取状态
      val randomAccessStats =
        "RandomAccess[" +
          "bytes=${sourceProvider.randomAccessByteReads}," +
          "reads=${sourceProvider.randomAccessReadCount}," +
          "travel=${sourceProvider.randomAccessByteTravel}," +
          "range=${sourceProvider.byteTravelRange}," +
          "size=${heapDumpFile.length()}" +
          "]"
      val stats = "$lruCacheStats $randomAccessStats"
      result.copy(metadata = result.metadata + ("Stats" to stats))
    }
  } catch (exception: Throwable) {
  //省略异常处理
  }
}
```
通过分析代码可知：分析内存快照分为以下5步：
1. 读取hprof内存快照文件
2. 找到LeakCanary 标记的泄漏对象们的数量和弱引用包装 ids，class name 为`com.squareup.leakcanary.KeyedWeakReference` 
> 代码在 KeyedWeakReferenceFinder#findLeakingObjectIds
1. 找到泄漏对象的gcRoot开始的路径
> 代码在PathFinder#findPathsFromGcRoots
1. 返回分析结果，走结果回调
2. 回调内 展示内存分析成功或者失败的通知栏消息，并将泄漏列表存储到数据库中
> 详情代码看 DefaultOnHeapAnalyzedListener#onHeapAnalyzed 以及 LeaksDbHelper
1. 点开通知栏跳转到`LeaksActivity` 展示内存泄漏信息。

# 七，总结
终于从头到尾，总算是梳理了一波LeakCanary 源码

过程中学习到了这么多—>
- 主动调用Gc的方式 `GcTrigger.Default.runGc()`
```kotlin
Runtime.getRuntime().gc()
```
- seald class 密封类来表达状态，比如以下几个（关键好处在于使用when可以直接覆盖所有情况，而不必使用else）。
```kotlin
sealed class ICanHazHeap {
  object Yup : ICanHazHeap()
  abstract class Nope(val reason: () -> String) : ICanHazHeap()
  class SilentNope(reason: () -> String) : Nope(reason)
  class NotifyingNope(reason: () -> String) : Nope(reason)
}
sealed class Result {
  data class Done(
    val analysis: HeapAnalysis,
    val stripHeapDumpDurationMillis: Long? = null
    ) : Result()
  data class Canceled(val cancelReason: String) : Result()
}
```
- 了解了系统创建内存快照的api 
```
 Debug.dumpHprofData(heapDumpFile.absolutePath)
```
- 知道了通过 ReferenceQueue 检测内存对象是否被gc，之前WeakReference都很少用。
- 学习了leakCanary的分模块思想。作为sdk,很多功能模块引入自动开启。比如 leakcanary-android-process 自动开启对应进程等。
- 学习了通过反射hook代码，替换实例达成添加钩子的操作。比如在Service泄漏监听代码中，替换`Handler`和`activityManager`的操作。

> 多多看源码还是有好处的。难怪我之前工作都找不到。看的太少了。