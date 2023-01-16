---
title: Android进阶二： 这次我把ARouter源码搞清楚啦！
---
![标题图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/313dd81582f94caa8933519c0fbb345a~tplv-k3u1fbpfcp-zoom-1.image)
> 随着面试和工作中多次遇到ARouter的使用问题，我决定把ARouter的源码从头到尾理一遍。
> 让我瞧瞧你到底有几斤几两，为啥大家在项目组件化中都用你做路由框架。
# 前言
在开发一个项目的时候，我们总是希望架构出的代码能够**自由复用**，**自由组装**，实现单一职责，并且抽离维护着各种各样可重复使用的组件。

而在组件化过程中，路由是个绕不过去的坎。

当模块可以自由拼装拆除的时候，类的强引用方式变得不可取。因为有些类很可能在编译期间就找不到了。所以就需要有种方式能通直接过序列化的字符串来拉起对应的功能或者页面。也就是通常的路由功能。ARouter就是一个接受度非常高的开源路由方案。

我写这篇文章，目的是对ARouter的源码原理进行一个全面的分析梳理。

**看完文章过后，将能学习到Arouter 的使用原理，注解处理器的开发方式，gradle插件对jar和class文件转dex过程进行中间处理。**

# 名词介绍
- `apt:`**APT**(Annotation Processing Tool)即**注解处理器**，是一种处理注解的工具，确切的说它是javac的一个工具，它用来在**编译时**扫描和处理注解。注解处理器以**Java代码**(或者编译过的字节码)作为输入，生成**java文件**作为输出。`ARouter`中通过处理注解生成相关路由信息类
- `asm:`ASM 库是一款基于 Java 字节码层面的代码分析和修改工具。ASM 可以直接生产二进制的 class 文件，也可以在类被加载入 JVM 之前动态修改类行为。在ARouter中用于`arouter_register`插件插入初始化代码。[官方链接](https://asm.ow2.io/index.html)

# 目录
1. 项目模块结构
2. ARouter路由使用分析
3. ARouter初始化分析
4. ARouter注解处理代码生成：arouter-compiler
5. ARouter自动注册插件：arouter-register
6. ARouter idea 插件：arouter helper
7. 自动代码生成
8. gradle插件

# 一. 项目模块结构
[官方仓库](https://github.com/alibaba/ARouter)
我们克隆github的ARouter源码，打开项目就是如下的项目结构图。
| 模块                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| app                   | 示例app模块                                                  |
| module-java           | java示例组件模块                                             |
| module-java-export    | java实例模块的服务数据模块，定义了一个示例的IProvider 服务接口和一些实体类 |
| module-kotlin         | kotlin示例组件模块                                           |
| arouter-annotation    | 注解类以及相关数据实体类                                     |
| arouter-api           | 主要api模块，提供ARouter类等路由方法                         |
| arouter-compiler      | 处理注解，生成相关代码                                       |
| arouter-gradle-plugin | gradle插件，jar包中添加自动注册代码，减少扫描dex文件开销     |
| arouter-idea-plugin   | idea插件，添加ARouter.build(path) 代码行标记，并点击跳转到代码定义 |
重点类简介
- `ARouter` :api入口类
- `LogisticsCenter` :路由逻辑中心维护所有路由图谱
- `Warehouse` :保存所有路由映射，通过他找到所以字符串对应的路由信息。这些信息是在ARouter 初始化的时候填充进来的。
- `RouteType` :路由类型，现在有: `*ACTIVITY，SERVICE，PROVIDER，CONTENT_PROVIDER，BOARDCAST，METHOD，FRAGMENT，UNKNOWN*`
- `RouteMeta`：路由信息类，包含路由类型，路由目标类class,路由组group名等。
# 二. ARouter 路由使用分析

ARouter的接入和使用参考官方说明就可以了。
### 接下来从常用Activity 跳转入手来了解路由导航处理
从最常用的api入手，我们就能知道ARouter 最主要的运转原理，了解他是怎么支撑实现跳转这个我们最常用的功能的。
跳转Activity代码如下:
```java
ARouter.getInstance().build("/test/activity").navigation();
```
**这一句代码就完成了activity跳转**

### **要点步骤如下：**

1. 通过`PathReplaceService` 预处理路径，并从`path:"/test/activity"` 中抽取出` group: "test"`
2. 将`path` 和`group` 作为参数创建 `Postcard` 实例
3. 调用 postcard#navigation ，最终导航到_ARouter#navigation
4. 通过 group 和 path 从`Warehouse.routes*`获取具体路径信息RouteMeta，完善postcard。

### 详细说明

####  第一步：抽取 group

不管是跳转`Activity`,获取`Fragment`实例，还是获取`Provider` 实例。 都是通过`ARouter.getInstance().build("")`方式来进行，这也是 ARouter最核心的路由api。这个build出来的产物，就是`Postcard`类。

我们先看build代码的执行路径：

```java
protected Postcard build(String path) {
    if (TextUtils.isEmpty(path)) {
        throw new HandlerException(Consts.TAG + "Parameter is invalid!");
    } else {
        ///  用户自定义路径处理类。默认为空。 ARouter.getInstance().navigation 直接获取Provider后文分析
        PathReplaceService pService = ARouter.getInstance().navigation(PathReplaceService.class);
        if (null != pService) {
            path = pService.forString(path);
        }
        ///获取path中包含的 group 作为参数二
        return build(path, extractGroup(path), true);
    }
}
```
上面的代码中，先是通过`ARouter`获取了`PathReplaceService` 实例，对路径进行了预处理操作(默认没有自定义实现类)。再通过`extractGroup`方法从 路径中获取了 `group`信息。
```java
private String extractGroup(String path) {
    if (TextUtils.isEmpty(path) || !path.startsWith("/")) {
        throw new HandlerException(Consts.TAG + "Extract the default group failed, the path must be start with '/' and contain more than 2 '/'!");
    }

    try {
        String defaultGroup = path.substring(1, path.indexOf("/", 1));
        if (TextUtils.isEmpty(defaultGroup)) {
            throw new HandlerException(Consts.TAG + "Extract the default group failed! There's nothing between 2 '/'!");
        } else {
            return defaultGroup;
        }
    } catch (Exception e) {
        logger.warning(Consts.TAG, "Failed to extract default group! " + e.getMessage());
        return null;
    }
}
```
从上面`extractGroup` 源码可知，抽取group的时候对路由路径 `"/test/activity"` 做了校验：
1. 一定要"/" 开头
2. 至少要有两个"/"
3. 第一个反斜杠后面的就是group
所以path路径一定要是类似 的格式，或者多来几个"/"。
#### 第二步：创建Postcard实例
很简单，直接new出来了
```java
return new Postcard(path, group);
```
#### 第三步：调用`_ARouter#navigation`
这块代码是路由的安卓核心跳转代码
很长一大串:
```java
protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    ///1.自定义预处理代码
    PretreatmentService pretreatmentService = ARouter.getInstance().navigation(PretreatmentService.class);
    if (null != pretreatmentService && !pretreatmentService.onPretreatment(context, postcard)) {
        // 预处理拦截了 返回
        return null;
    }

    // 设置context
    postcard.setContext(null == context ? mContext : context);

    try {
        ///2.通过路由信息，找到对应的路由信息 RouteMeta ，根据路由类型 RouteType 
        ///完善posrcard   
        LogisticsCenter.completion(postcard);
    } catch (NoRouteFoundException ex) {
        ///... 省略异常日志和弹窗展示。以及相关回调方法
        ///值得一提的是走了 DegradeService 的自定义丢失回调
    }

    if (null != callback) {
        callback.onFound(postcard);
    }

    ///3.如果不是绿色通道，需要走拦截器：InterceptorServiceImpl
    if (!postcard.isGreenChannel()) {   // It must be run in async thread, maybe interceptor cost too mush time made ANR.
        interceptorService.doInterceptions(postcard, new InterceptorCallback() {
            @Override
            public void onContinue(Postcard postcard) {
                ///4.继续导航方法
                _navigation(postcard, requestCode, callback);
            }
            @Override
            public void onInterrupt(Throwable exception) {
               ///省略拦截后的一些代码
               }
        });
    } else {
        ///4.继续导航方法
        return _navigation(postcard, requestCode, callback);
    }

    return null;
}
```
简单总结一下主要代码步骤：
1. 如有自定义预处理导航逻辑，执行并检查拦截

2. 通过path路径找到对应的routemeta路由信息，用该信息完善`postcard`对象（`LogisticsCenter.completion`方法中完成，细节后文分析）

3. 如果不是绿色通道，需要走拦截器：`InterceptorServiceImpl` 。该拦截器服务类中完成拦截器一一执行。(2的源码细节可知，`PROVIDER`和`FRAGMENT`类型是绿色通道)

4. 继续导航方法，调用_navigation。

看代码：
```java
private Object _navigation(final Postcard postcard, final int requestCode, final NavigationCallback callback) {
    final Context currentContext = postcard.getContext();

    switch (postcard.getType()) {
        case ACTIVITY:
            // Build intent
            final Intent intent = new Intent(currentContext, postcard.getDestination());
            //...省略完善intent代码
            // Navigation in main looper.
            runInMainThread(new Runnable() {
                @Override
                public void run() {
                    startActivity(requestCode, currentContext, intent, postcard, callback);
                }
            });

            break;
        case PROVIDER:
            return postcard.getProvider();
        case BOARDCAST:
        case CONTENT_PROVIDER:
        case FRAGMENT:
            Class<?> fragmentMeta = postcard.getDestination();
            try {
                Object instance = fragmentMeta.getConstructor().newInstance();
                if (instance instanceof Fragment) {
                    ((Fragment) instance).setArguments(postcard.getExtras());
                } else if (instance instanceof android.support.v4.app.Fragment) {
                    ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                }

                return instance;
            } catch (Exception ex) {
                logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
            }
        case METHOD:
        case SERVICE:
        default:
            return null;
    }

    return null;
}
```
很明显，代码中注意对各种类型的路由做了处理。
- *ACTIVITY：*新建`Intent` ，通过`postcard`信息，完善`intent`走`context.startActivity` 或者 `context.startActivityForResult`。
- *PROVIDER：*`postcard.getProvider()` 获取`provider`实例（实例化代码在`LogisticsCenter.completion`）
- *FRAGMENT，BOARDCAST，CONTENT_PROVIDER：*`routeMeta.getConstructor().newInstance()` 通过路由信息实例化出实例，如果是Fragment的话，则另外再设置`extras`信息。
- *METHOD，SERVICE：*返回空，啥也不做。说明该类型路由调用`navigation`没啥意义。

看到这里，对于Activity 的路由跳转就很直观了，就是调用了`startActivity` 或者 `startActivityForResult` 方法，其他`provider` `fragment`等实例的获取也十分得清晰明了了，接下来讲讲上面提到的补全`postcard`关键代码。
#### 关键代码：LogisticsCenter.completion 分析
完善`postcard`信息代码是通过`LogisticsCenter.completion` 方法完成的。现在来梳理一下这一块代码：
```java
/**
 * 通过RouteMate 完善 postcard
 * @param postcard Incomplete postcard, should complete by this method.
 */
public synchronized static void completion(Postcard postcard) {
    //省略空判断
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    if (null == routeMeta) {
        // 如果路由的组group没有找到，直接抛异常
        if (!Warehouse.groupsIndex.containsKey(postcard.getGroup())) {
            throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
        } else {
            //...省略一些日志代码
            // 1.动态添加组元素（从groupsIndex 中找到对应 IRouteGroup的生成类，再对组元素进行加载）
            addRouteGroupDynamic(postcard.getGroup(), null);
            completion(postcard);   // Reload
        }
    } else {
        postcard.setDestination(routeMeta.getDestination());
        postcard.setType(routeMeta.getType());
        postcard.setPriority(routeMeta.getPriority());
        postcard.setExtra(routeMeta.getExtra());

        Uri rawUri = postcard.getUri();
        ///2.如果有uri 信息，解析uri相关参数。解析出AutoWired的参数的值
        if (null != rawUri) {   // Try to set params into bundle.
            Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
            Map<String, Integer> paramsType = routeMeta.getParamsType();

            if (MapUtils.isNotEmpty(paramsType)) {
                // Set value by its type, just for params which annotation by @Param
                for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                    setValue(postcard,
                            params.getValue(),
                            params.getKey(),
                            resultMap.get(params.getKey()));
                }
                // Save params name which need auto inject.
                postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
            }
            // Save raw uri
            postcard.withString(ARouter.RAW_URI, rawUri.toString());
        }
        ///3.获取provider实例，如果初始获取，初始化该provider， 最后赋值给postcard 
        switch (routeMeta.getType()) {
            case PROVIDER:  // if the route is provider, should find its instance
                // Its provider, so it must implement IProvider
                Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                IProvider instance = Warehouse.providers.get(providerMeta);
                if (null == instance) { // There's no instance of this provider
                    IProvider provider;
                    try {
                        provider = providerMeta.getConstructor().newInstance();
                        provider.init(mContext);
                        Warehouse.providers.put(providerMeta, provider);
                        instance = provider;
                    } catch (Exception e) {
                        logger.error(TAG, "Init provider failed!", e);
                        throw new HandlerException("Init provider failed!");
                    }
                }
                postcard.setProvider(instance);
                postcard.greenChannel();    // Provider should skip all of interceptors
                break;
            case FRAGMENT:
                postcard.greenChannel();    // Fragment needn't interceptors
            default:
                break;
        }
    }
}
```
梳理一下这一块的代码,这一部分代码完善了postcard信息,总共分成了三个要点
1. **获取路由信息：**如果路由信息找不到，通过组信息，重新动态添加组group内所有路由 ，调用`addRouteGroupDynamic` 。
2. **获取uri内的参数：**如果postcard创建的时候有传递uri。解析uri里面所有需要AutoInject的参数。放置到postcard中。
3. **获取Provider实例，配置是否不走拦截器的绿色通道：**不存在的Provider通过路由信息的 getDestination 反射创建实例并初始化，存在的直接获取。

分析到这里。各种`RouteType`的跳转，实例获取都已经明了了。
现在剩下的问题是，`WareHouse`里面的路由信息数据是哪里来的？前面提到了动态添加组内路由的方法`addRouteGroupDynamic`。

我们来看看：

```java
public synchronized static void addRouteGroupDynamic(String groupName, IRouteGroup group) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    if (Warehouse.groupsIndex.containsKey(groupName)){
        // If this group is included, but it has not been loaded
        // load this group first, because dynamic route has high priority.
        Warehouse.groupsIndex.get(groupName).getConstructor().newInstance().loadInto(Warehouse.routes);
        Warehouse.groupsIndex.remove(groupName);
    }

    // cover old group. 
    if (null != group) {
        group.loadInto(Warehouse.routes);
    }
}
```
可以看到`Warehouse.routes` 里面的所有路由信息，都是从 `IRouteGroup.loadInto` 加载出来的。而`IRouteGroup` 都存在`Warehouse.groupsIndex` *中。*
这时候新的问题出现了，`Warehouse.groupsIndex`的数据是哪里来的呢?
下一节 `ARouter 初始化分析`就有答案。

#### tips:上文中提到的对外可自定义配置:
简单罗列下源码中提到的可自定义配置的IProvider。便于使用的时候自定义。
- PathReplaceService    ///路由自定义处理替换
- DegradeService       //没有找到路由的通用回调
- PretreatmentService    ///navigation 预处理拦截
### 通过Class 获取IProvider实例
前面提到的`PathReplaceService` 等用户自定义类，都是通过`ARouter.getInstance().navigation(clazz)` 方式获取的。
这一块代码又是怎么从路由信息中获取到实例的呢？看看具体的navigation代码：

```java
protected <T> T navigation(Class<? extends T> service) {
    try {
        //1.通过类名从Provider路由信息索引中，获取路由信息，组建postcart
        Postcard postcard = LogisticsCenter.buildProvider(service.getName());

        // Compatible 1.0.5 compiler sdk.
        // Earlier versions did not use the fully qualified name to get the service
        if (null == postcard) {
            // No service, or this service in old version.
             //1.通过类名从Provider路由信息索引中，获取路由信息，组建postcart
                postcard = LogisticsCenter.buildProvider(service.getSimpleName());
        }

        if (null == postcard) {
            return null;
        }

        // Set application to postcard.
        postcard.setContext(mContext);
        //2.完善postcard ,该方法里面创建provider
        LogisticsCenter.completion(postcard);
        return (T) postcard.getProvider();
    } catch (NoRouteFoundException ex) {
        logger.warning(Consts.TAG, ex.getMessage());
        return null;
    }
}
```
很明显，主要代码就是 `LogisticsCenter.buildProvider(service.getName())` ,获取到了`postcard`。后面完善`postcard` 和 获取`provider`实例的代码都已经在上文讲过。
所以我们就看`buildProvider` 方法:

```java
public static Postcard buildProvider(String serviceName) {
    RouteMeta meta = Warehouse.providersIndex.get(serviceName);

    if (null == meta) {
        return null;
    } else {
        return new Postcard(meta.getPath(), meta.getGroup());
    }
}
```
和路由组信息获取类似，`Provider`的路由信息从 `Warehouse.providersIndex ` 维护的映射表中获取。
所以`providersIndex`是专门用来给没有@Route 路由信息的Provider创建实例用的。这就是维护`providersIndex`的用途。
接下来的问题就转为了 `providersIndex` 里面的数据是哪里来的。

### 小结
路由跳转以及获取Provider等实例的原理可以简单总结下：
1. 先是获取`postcard`，可能是直接通过路由路径和`uri`构建, 如`"/test/activity1"`，也可能是通过`Provider` 类名从索引获取，如`PathReplaceService.class.getName()`
2. 然后通过`RouteMate`完善 `postcard`。获取诸如类名信息,路由类型，provider实例等信息。
3. 最后导航，根据路由类型作出跳转或者返回对应实例。

关键点在于`WareHouse` 维护的路由图谱。
# 三. ARouter初始化分析
我们看下对用户提供的`ARouter#init`方法：
```java
public static void init(Application application) {
    if (!hasInit) {
        logger = _ARouter.logger;
        _ARouter.logger.info(Consts.TAG, "ARouter init start.");
        ///调用初始化代码
        hasInit = _ARouter.init(application);
        ///初始化完成后，加载拦截器服务，并初始化所有拦截器
        if (hasInit) {
            _ARouter.afterInit();
        }
        _ARouter.logger.info(Consts.TAG, "ARouter init over.");
    }
}
```
代码关键就两步，
1. 初始化ARouter
2. 获取拦截器服务实例初始化所有拦截器
### 初始化ARouter
init代码最终调用到了`LogisticsCenter#init`
```java
public synchronized static void init(Context context, ThreadPoolExecutor tpe) throws HandlerException {
    mContext = context;
    executor = tpe;

    try {
        long startInit = System.currentTimeMillis();
        //load by plugin first
        //1.加载路由映射表 （通过 ARouter 插件 注册）
        loadRouterMap();
        //2.是否通过插件注册初始化
        if (registerByPlugin) {
            logger.info(TAG, "Load router map by ARouter-auto-register plugin.");
        } else {
            Set<String> routerMap;

            // It will rebuild router map every times when debuggable.
            //可调试 和新版本的时候 重建路由表
            if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
                logger.info(TAG, "Run with debug mode or new install, rebuild router map.");
                // These class was generated by ARouter-compiler.
               //3.在线程池中,扫描所有dex文件,通过包名获取路由映射表类名
               //包名 ROUTE_ROOT_PAKCAGE :com.alibaba.android.ARouter.routes
                routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
                ///保存路由映射表类名们到缓存中
                if (!routerMap.isEmpty()) {
                    context.getSharedPreferences(ARouter_SP_CACHE_KEY, Context.MODE_PRIVATE).edit().putStringSet(ARouter_SP_KEY_MAP, routerMap).apply();
                }

                PackageUtils.updateVersion(context);    // Save new version name when router map update finishes.
            } else {
                ///从 SharedPreferences 缓存中获取所有的路由类文件
                logger.info(TAG, "Load router map from cache.");
                routerMap = new HashSet<>(context.getSharedPreferences(ARouter_SP_CACHE_KEY, Context.MODE_PRIVATE).getStringSet(ARouter_SP_KEY_MAP, new HashSet<String>()));
            }

            logger.info(TAG, "Find router map finished, map size = " + routerMap.size() + ", cost " + (System.currentTimeMillis() - startInit) + " ms.");
            startInit = System.currentTimeMillis();
            ///4.加载 IRouteRoot，IInterceptorGroup，IProviderGroup 类，填充Warehouse 的路由信息组索引
            for (String className : routerMap) {
                if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                    // This one of root elements, load root.
                    ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                    // Load interceptorMeta
                    ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                    // Load providerIndex
                    ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                }
            }
        }
        //..省略路由类初始化结果的日志代码
    } catch (Exception e) {
        throw new HandlerException(TAG + "ARouter init logistics center exception! [" + e.getMessage() + "]");
    }
}
```
通过代码我们了解到了这么几个过程:
1.  方式一 : `ARouter-auto-register` 插件加载路由表（如果有该插件），该方式详细分析见第五节。
2.  方式二 :
   1.  在需要的时候扫描所有 dex文件,找到所有包名为`com.alibaba.android.ARouter.routes`的类,类名放到`routerMap `集里面。
   2. 实例化上面找到的所有类，并通过这些集类加载对应的集映射索引到WareHouse中。
   很显然，`com.alibaba.android.ARouter.routes` 包名下面的类都是自动生成的路由表类。
   通过搜索我们能找到样例代码中生成的该包名对象们:
   ![生成的包名](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cab7ffb613e44068a4b0373600717bc~tplv-k3u1fbpfcp-zoom-1.image)
   `module_java `生成的`IRouteRoot `代码如下所示
```java
public class ARouter$$Root$$modulejava implements IRouteRoot {
  @Override
  public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
    routes.put("m2", ARouter$$Group$$m2.class);
    routes.put("module", ARouter$$Group$$module.class);
    routes.put("test", ARouter$$Group$$test.class);
    routes.put("yourservicegroupname", ARouter$$Group$$yourservicegroupname.class);
  }
}
```
这样我们就完全清楚了 `WareHouse`里面维护的所有路由信息是哪里来的了。追本溯源。接下来我们只需要知道 `ARouter$$Root$$modulejava `等类，是啥时候怎么生成的。我们在下面一小节进行分析。初始化ARouter的过程,其实就是填充`Warehouse `的`providersIndex`，`groupsIndex`，`interceptorsIndex`
### 初始化后续:初始化所有拦截器
初始化完成，看看初始化完成后的操作`afterInit`
```java
static void afterInit() {
    // Trigger interceptor init, use byName.
    interceptorService = (InterceptorService) ARouter.getInstance().build("/ARouter/service/interceptor").navigation();
}
```
这一块代码就是`navigation ` 获取到了`InterceptorService`。上面也讲过,在执行`navigation`的时候，会调用`IProvider`的`init`方法。所以我们需要找到`InterceptorService` 的实现类，并看看他的`init`做了什么。项目中其实现类是`InterceptorServiceImpl`,找到`init`代码如下：
```java
@Override
public void init(final Context context) {
    LogisticsCenter.executor.execute(new Runnable() {
        @Override
        public void run() {
            if (MapUtils.isNotEmpty(Warehouse.interceptorsIndex)) {
                for (Map.Entry<Integer, Class<? extends IInterceptor>> entry : Warehouse.interceptorsIndex.entrySet()) {
                    Class<? extends IInterceptor> interceptorClass = entry.getValue();
                    try {
                        IInterceptor iInterceptor = interceptorClass.getConstructor().newInstance();
                        iInterceptor.init(context);
                        Warehouse.interceptors.add(iInterceptor);
                    } catch (Exception ex) {
                        throw new HandlerException(TAG + "ARouter init interceptor error! name = [" + interceptorClass.getName() + "], reason = [" + ex.getMessage() + "]");
                    }
                }

                interceptorHasInit = true;
                logger.info(TAG, "ARouter interceptors init over.");
                synchronized (interceptorInitLock) {
                    interceptorInitLock.notifyAll();
                }
            }
        }
    });
}
```
代码很明白的告诉我们，该初始化代码从拦截器路由信息索引里面加载并实例化了所有拦截器。然后通知等待的拦截器开始拦截。
### 小结
看完初始化代码之后,明白了WareHouse的数据来源,现在问题变成了`com.alibaba.android.ARouter.routes` 包名的代码何时生成。我们且看下回分解。
# 四. ARouter 注解处理器：arouter-compiler
ARouter 生成路由信息代码利用了注解处理器的特性。
`arouter-compiler` 就是注解处理代码模块，先看看该模块的依赖库
```groovy
//定义的注解类，以及相关数据实体类
implementation 'com.alibaba:arouter-annotation:1.0.6'

annotationProcessor 'com.google.auto.service:auto-service:1.0-rc7'
compileOnly 'com.google.auto.service:auto-service-annotations:1.0-rc7'

implementation 'com.squareup:javapoet:1.8.0'

implementation 'org.apache.commons:commons-lang3:3.5'
implementation 'org.apache.commons:commons-collections4:4.1'

implementation 'com.alibaba:fastjson:1.2.69'
```
依赖库中注解处理相关依赖库说明：
- `Auto-service`: [官方文档](https://github.com/google/auto/tree/master/service)  针对被`@AutoService`注解的类，生成对应元数据，在javac 编译的时候，会自动加载，并放在注释处理环境中。
- `javapoet` :square推出的开源java代码生成框架，我们可以很方便的使用它根据注解、数据库模式、协议格式等来对应生成代码。
- `arouter-annotation` :arouter 的注解类们和路由信息实体类们
- 其他，工具类库
### `RouteProcessor`注解处理器处理流程说明
我们先看看路由处理器 `RouteProcessor`
```java
@AutoService(Processor.class)
@SupportedAnnotationTypes({ANNOTATION_TYPE_ROUTE, ANNOTATION_TYPE_AUTOWIRED})
public class RouteProcessor extends BaseProcessor {
    @Override
    //在该方法中可以获取到processingEnvironment对象，
    //借由该对象可以获取到生成代码的文件对象, debug输出对象，以及一些相关工具类
    public synchronized void init(ProcessingEnvironment processingEnv) {
         //...
        super.init(processingEnv);
    }
    @Override
    //返回所支持的java版本，一般返回当前所支持的最新java版本即可
    public SourceVersion getSupportedSourceVersion() {
          //...
        return super.getSupportedSourceVersion();
    }

    @Override
    //必须实现 扫描所有被注解的元素，并作处理，最后生成文件。该方法的返回值为boolean类型，若返回true,
    //则代表本次处理的注解已经都被处理，不希望下一个注解处理器继续处理，
    //否则下一个注解处理器会继续处理。
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        //... 
        return false;
    }

}
```
可以看到处理注解主要是在process方法。
`RouteProcessor` 继承 `BaseProcessor` 间接继承了`AbstractProcessor`，在`BaseProcessor#init `方法中，获取到`processingEnv` 中的各种实用工具，以供处理注解使用。
值得一提的是，`init` 中获取了`moduleName` 和 `generateDoc` 参数代码如下:
```java
if (MapUtils.isNotEmpty(options)) {
    ///AROUTER_MODULE_NAME
    moduleName = options.get(KEY_MODULE_NAME);
    ///AROUTER_GENERATE_DOC
    generateDoc = VALUE_ENABLE.equals(options.get(KEY_GENERATE_DOC_NAME));
}
```
这一块就是我们常常需要在gradle中配置的arguments 的由来：
```groovy
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName(), AROUTER_GENERATE_DOC: "enable"]
            }
        }
    }
}
//或者kotlin
kapt {
    arguments {
        arg("AROUTER_MODULE_NAME", project.getName())
    }
}
```
接下来看`RouteProcessor#process`方法的具体实现:
```java
Set<? extends Element> routeElements = roundEnv.getElementsAnnotatedWith(Route.class);
this.parseRoutes(routeElements);
```
代码中拿到了所有标注`@Route`注解的相关类元素。
然后在`parseRoutes`方法中进行处理:
省略了一大把代码后，代码还是很长，可以直接到下面看结论：
```java
private void parseRoutes(Set<? extends Element> routeElements) throws IOException {
    if (CollectionUtils.isNotEmpty(routeElements)) {
        // prepare the type an so on.
        logger.info(">>> Found routes, size is " + routeElements.size() + " <<<");

        rootMap.clear();
        ///省略类型获取代码
        
        /*...省略构建'loadInto'方法描述,通过定义变量名，
            定义类型最后得出 MethodSpec.Builder
            void loadInto(Map<String, Class<? extends IRouteGroup>> atlas);
        */
        MethodSpec.Builder loadIntoMethodOfRootBuilder;

        //  Follow a sequence, find out metas of group first, generate java file, then statistics them as root.
        for (Element element : routeElements) {
             //..省略相关代码，根据element类型，创建出对应的RouteMate实例，得到路由信息，
             //并且通过injectParamCollector 方法将Activity和Fragmentr内部的所有@AutoWired
             //注解 的信息放到MetaData的 paramsType 和injectConfig 中
v           
            //对 routeMate进行分类,在groupMap中填充对应数据
            categories(routeMeta);
        }

        /*...省略构建'loadInto'方法描述,通过定义变量名，
            定义类型最后得出 MethodSpec.Builder,主要用来构建providers索引。
            void loadInto(Map<String, RouteMeta> providers);
        */
        MethodSpec.Builder loadIntoMethodOfProviderBuilder;

        Map<String, List<RouteDoc>> docSource = new HashMap<>();

        // Start generate java source, structure is divided into upper and lower levels, used for demand initialization.
        for (Map.Entry<String, Set<RouteMeta>> entry : groupMap.entrySet()) {
            String groupName = entry.getKey();

            /**获取方法描述
            void loadInto(Map<String, RouteMeta> atlas)*/
            MethodSpec.Builder loadIntoMethodOfGroupBuilder;
            
            ///文档列表，后续转为json文件 存在arouter-map-of-{module_name}.json 中
            List<RouteDoc> routeDocList = new ArrayList<>();
            // 为loadInto方法添加方法体，
            // 第一步：遍历组下面的所有RouteMate,如果是IProvider子类，添加到 providers 的loadInto方法中
            // 第二步：添加参数列表
            Set<RouteMeta> groupData = entry.getValue();
            for (RouteMeta routeMeta : groupData) {
                RouteDoc routeDoc = extractDocInfo(routeMeta);

                ClassName className = ClassName.get((TypeElement) routeMeta.getRawType());
                 //省略第一步provider的添加代码
                
                // Make map body for paramsType
                StringBuilder mapBodyBuilder = new StringBuilder();
                Map<String, Integer> paramsType = routeMeta.getParamsType();
                Map<String, Autowired> injectConfigs = routeMeta.getInjectConfig();
                if (MapUtils.isNotEmpty(paramsType)) {
                   //..省略构建参数方法体字符串，以及添加参数们到doc 数据中
                }
                String mapBody = mapBodyBuilder.toString();

                //添加IRouteGroup#loadInto的方法体 
                loadIntoMethodOfGroupBuilder.addStatement(
                        "atlas.put($S, $T.build($T." + routeMeta.getType() + ", $T.class, $S, $S, " + (StringUtils.isEmpty(mapBody) ? null : ("new java.util.HashMap<String, Integer>(){{" + mapBodyBuilder.toString() + "}}")) + ", " + routeMeta.getPriority() + ", " + routeMeta.getExtra() + "))",
                        /*...*/);

                routeDoc.setClassName(className.toString());
                routeDocList.add(routeDoc);
            }

            // Generate groups
            //1.生成对应的IRouteGroup 类代码文件 ARouter$$Group$$[GroupName]
            String groupFileName = NAME_OF_GROUP + groupName;
            JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                    TypeSpec.classBuilder(groupFileName)
                            .addJavadoc(WARNING_TIPS)
                            .addSuperinterface(ClassName.get(type_IRouteGroup))
                            .addModifiers(PUBLIC)
                            .addMethod(loadIntoMethodOfGroupBuilder.build())
                            .build()
            ).build().writeTo(mFiler);

            logger.info(">>> Generated group: " + groupName + "<<<");
            rootMap.put(groupName, groupFileName);
            docSource.put(groupName, routeDocList);
        }

        if (MapUtils.isNotEmpty(rootMap)) {
            // Generate root meta by group name, it must be generated before root, then I can find out the class of group.
            for (Map.Entry<String, String> entry : rootMap.entrySet()) {
                loadIntoMethodOfRootBuilder.addStatement("routes.put($S, $T.class)", entry.getKey(), ClassName.get(PACKAGE_OF_GENERATE_FILE, entry.getValue()));
            }
        }

        // 2.Output route doc 写入json到doc文档中
        if (generateDoc) {
            docWriter.append(JSON.toJSONString(docSource, SerializerFeature.PrettyFormat));
            docWriter.flush();
            docWriter.close();
        }

        // Write provider into disk
        //3.生成对应的IProviderGroup 类代码文件 ARouter$$Providers$$[moduleName]
        String providerMapFileName = NAME_OF_PROVIDER + SEPARATOR + moduleName;
        JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                TypeSpec.classBuilder(providerMapFileName)
                        .addJavadoc(WARNING_TIPS)
                        .addSuperinterface(ClassName.get(type_IProviderGroup))
                        .addModifiers(PUBLIC)
                        .addMethod(loadIntoMethodOfProviderBuilder.build())
                        .build()
        ).build().writeTo(mFiler);

        logger.info(">>> Generated provider map, name is " + providerMapFileName + " <<<");

        // Write root meta into disk.
        //4. 生成对应的IRouteRoot 类代码文件 ARouter$$Root$$[moduleName]
        String rootFileName = NAME_OF_ROOT + SEPARATOR + moduleName;
        JavaFile.builder(PACKAGE_OF_GENERATE_FILE,
                TypeSpec.classBuilder(rootFileName)
                        .addJavadoc(WARNING_TIPS)
                        .addSuperinterface(ClassName.get(elementUtils.getTypeElement(ITROUTE_ROOT)))
                        .addModifiers(PUBLIC)
                        .addMethod(loadIntoMethodOfRootBuilder.build())
                        .build()
        ).build().writeTo(mFiler);

        logger.info(">>> Generated root, name is " + rootFileName + " <<<");
    }
}
```
代码很长，关键结果就是三点，
1. 在`com.alibaba.android.arouter.routes `包名下生成`ARouter$$Group$$[GroupName] `类，包含路由组的所有路由信息。
2. 在该包名下生成`ARouter$$Root$$[moduleName]`类，包含所有组的信息。
3. 在该包名下生成`ARouter$$Providers$$[moduleName] `类，包含所有`Providers`索引
4. 在docs下，生成文件名为 `"arouter-map-of-" + moduleName + ".json"` 的文档。
### 其他注解处理器说明
剩下还有两个注解处理器 `InterceptorProcessor` 和 `AutowiredProcessor`。
生成代码逻辑大同小异，只是逻辑复杂度的区别，
- `AutowiredProcessor` :处理`@Autowired`注解的参数，以参数所在对应的类分类，生成`[classSimpleName]$$ARouter$$Autowired `代码文件，以在Activity或者Fragment跳转的时候自动从`intent`中获取数据，并对activity 和 fragment 对象赋值。 
- `InterceptorProcessor`: 处理`@Interceptor`注解。生成对应的`ARouter$$Interceptors$$[modulename]`代码文件，提供拦截器功能。

值得一提的是，对于自定义类型的`@AutoWired`，`ARouter`提供了 `SerializationService`进行自定义，用户只需要实现该解析类就行。
### 小结
这个模块完成了之前ARouter初始化所需要的所有代码的生成。
ARouter 源码和源码的分析到这里，已经成功走到了闭环，主要功能都已经清楚了。
之前没有写过`AnotationProcessor`相关的代码生成库。这次算是学习到了整个注解处理代码生成框架的使用方式。也了解了ARouter 代码生成的原理和方式。
业余时间自己也尝试写一个简单的代码生成功能试试看吧。下面一小节，再看看ARouter初始化注册的可选方案，`arouter-register`的源码。
# 五. ARouter 自动注册插件：arouter-register
代码在`arouter-gradle-plugin` 文件夹下面，
> 刚开始查看这个模块的源码，部分代码老是飘红，找不到部分类，于是我修改了该模块build.gradle 中的gradle依赖版本号。从2.1.3 改成了 4.1.3。代码果然就正常了。
> gradle插件调试可以更好地理解代码，参考[网上的博客](https://www.jianshu.com/p/99c8e953654e)启动插件调试。
### 注册转换器
ARouter-register 插件通过 `registerTransform` api。添加了一个自定义`Transform`，对dex进行自定义处理。
直接看 该源码中的入口代码 `PluginLaunch#apply`
```java
def isApp = project.plugins.hasPlugin(AppPlugin)
//only application module needs this plugin to generate register code
if (isApp) {
    def android = project.extensions.getByType(AppExtension)
    def transformImpl = new RegisterTransform(project)
    android.registerTransform(transformImpl)
}
```
代码中调用了`AppExtension.registerTransform`方法注册了 `RegisterTransform`。查阅[api文档](http://tools.android.com/tech-docs/new-build-system/transform-api)可知，该方法的功能是：**允许第三方方插件在将编译的类文件转换为 dex 文件之前对其进行操作。**
那就知道了，该方法就是类文件转换中间的一道工序。
### 扫描class文件和jar文件，保存路由类信息
那工序做了什么呢？看看代码`RegisterTransform#transform`：
```java
@Override
void transform(Context context, Collection<TransformInput> inputs
               , Collection<TransformInput> referencedInputs
               , TransformOutputProvider outputProvider
               , boolean isIncremental) throws IOException, TransformException, InterruptedException {

    Logger.i('Start scan register info in jar file.')

    long startTime = System.currentTimeMillis()
    boolean leftSlash = File.separator == '/'

    inputs.each { TransformInput input ->

        //通过AMS 的 ClassVisistor 扫描所有的jar 文件，将所有扫描到的IRouteRoot IInterceptorGroup IInterceptorGroup类
        //都加到ScanSetting 的 classList中
        //详情可以看看 ScanClassVisitor
        //如果jar包是 LogisticsCenter.class，标记该类文件到 fileContainsInitClass
        
        input.jarInputs.each { JarInput jarInput ->
            //排除对于support库，以及m2repository 内第三方库的扫描。scan jar file to find classes 
            if (ScanUtil.shouldProcessPreDexJar(src.absolutePath)) {
                //扫描
                ScanUtil.scanJar(src, dest)
            }
            //..省略重命名扫描过的jar包相关代码
        }
        // scan class files
        //..省略扫描class文件相关代码，方式类似扫描jar包
    }

    Logger.i('Scan finish, current cost time ' + (System.currentTimeMillis() - startTime) + "ms")
    //如果存在 LogisticsCenter.class 类文件
    //插入注册代码到 LogisticsCenter.class 中
    if (fileContainsInitClass) {
        registerList.each { ext ->
            //...省略一些判空和日志代码
            ///插入初始化代码
            RegisterCodeGenerator.insertInitCodeTo(ext)
        }
    }
    Logger.i("Generate code finish, current cost time: " + (System.currentTimeMillis() - startTime) + "ms")
}
```
从代码中可知，这一块代码有四个关键点。
1. 通过ASM扫描了对应的jar 文件和class文件，并将扫描到的对应`routes`包下的类加入到`ScanSetting` 的`classList`属性 中 
2. 如果扫描到包含`LogisticsCenter.class` 类文件，将该文件记录到`fileContainsInitClass` 字段中。
3. 扫描完成的文件重命名。
4. 最后通过`RegisterCodeGenerator.insertInitCodeTo(ext)` 方法插入初始化代码到`LogisticsCenter.class`中。
明白了扫描流程，我们再看看代码是怎么插入的.
### 遍历包含入口class的jar文件，准备插入代码
在`RegisterCodeGenerator.insertInitCodeTo(ext)`代码中，先判断`ScanSetting#classList`是否为空，再判断文件是否是jar文件。如果判断都过了，最后走到 `RegisterCodeGenerator#insertInitCodeIntoJarFile`代码：
```java
private File insertInitCodeIntoJarFile(File jarFile) {
  //将包含 LogisticsCenter.class 的 jar文件，插入初始化代码
  //操作在 ***.jar.opt 临时文件做
    if (jarFile) {
        def optJar = new File(jarFile.getParent(), jarFile.name + ".opt")
        if (optJar.exists())
            optJar.delete()
        ///通过JarFile 和JarEntry 
        def file = new JarFile(jarFile)
        Enumeration enumeration = file.entries()
        JarOutputStream jarOutputStream = new JarOutputStream(new FileOutputStream(optJar))
        //遍历jar中的所有class，查询修改
        while (enumeration.hasMoreElements()) {
            JarEntry jarEntry = (JarEntry) enumeration.nextElement()
            String entryName = jarEntry.getName()
            ZipEntry zipEntry = new ZipEntry(entryName)
            InputStream inputStream = file.getInputStream(jarEntry)
            jarOutputStream.putNextEntry(zipEntry)
            ///如果是LogisticsCenter.class文件，调用referHackWhenInit 插入代码
            ///如果不是，不改变数据直接写入
            if (ScanSetting.GENERATE_TO_CLASS_FILE_NAME == entryName) {
                Logger.i('Insert init code to class >> ' + entryName)
                //！！！！重点代码,插入初始化代码
                def bytes = referHackWhenInit(inputStream)
                jarOutputStream.write(bytes)
            } else {
                jarOutputStream.write(IOUtils.toByteArray(inputStream))
            }
            inputStream.close()
            jarOutputStream.closeEntry()
        }
        jarOutputStream.close()
        file.close()

        if (jarFile.exists()) {
            jarFile.delete()
        }
        optJar.renameTo(jarFile)
    }
    return jarFile
}
```
从代码中可知，按步骤梳理：
1. 创建临时文件，`***.jar.opt`
2. 通过输入输出流，遍历`jar`文件下面的所有`class`，判断是否`LogisticCenter.class`
3. 对`LogisticCenter.class `调用 `referHackWhenInit` 方法插入初始化代码，写入到opt临时文件
4. 对其他`class` 原封不动写入`opt`临时文件
5. 删除原来的`jar`文件，将临时文件改名为原来的`jar`文件名
这一步完成了对于`jar`文件的修改。插入了ARouter的自动注册初始化代码。
### 插入初始化代码
插入操作主要是找到 LogisticCenter
关键的插入代码在于`RegisterCodeGenerator#referHackWhenInit`：
```java
private byte[] referHackWhenInit(InputStream inputStream) {
    ClassReader cr = new ClassReader(inputStream)
    ClassWriter cw = new ClassWriter(cr, 0)
    ClassVisitor cv = new MyClassVisitor(Opcodes.ASM5, cw)
    cr.accept(cv, ClassReader.EXPAND_FRAMES)
    return cw.toByteArray()
}
```
可以看到代码中利用了`ams` 框架的 `ClassVisitor` 来访问入口类。
再看`MyClassVisistor` 的`visitMethod` 实现：

```java
@Override
MethodVisitor visitMethod(int access, String name, String desc,
                          String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions)
    //generate code into this method
   //针对loadRouterMap 方法进行处理
    if (name == ScanSetting.GENERATE_TO_METHOD_NAME) {
        mv = new RouteMethodVisitor(Opcodes.ASM5, mv)
    }
    return mv
}
```
可以看到，当`asm`访问的方法名为`loadRouterMap`时候，就通过`RouteMethodVisitor` 对齐进行操作，具体代码如下：
```java
class RouteMethodVisitor extends MethodVisitor {
    RouteMethodVisitor(int api, MethodVisitor mv) {
        super(api, mv)
    }
    @Override
    void visitInsn(int opcode) {
        //generate code before return
        if ((opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN)) {
            extension.classList.each { name ->
                name = name.replaceAll("/", ".")
                mv.visitLdcInsn(name)//类名
                // generate invoke register method into LogisticsCenter.loadRouterMap()
                mv.visitMethodInsn(Opcodes.INVOKESTATIC
                        , ScanSetting.GENERATE_TO_CLASS_NAME
                        , ScanSetting.REGISTER_METHOD_NAME
                        , "(Ljava/lang/String;)V"
                        , false)
            }
        }
        super.visitInsn(opcode)
    }
    @Override
    void visitMaxs(int maxStack, int maxLocals) {
        super.visitMaxs(maxStack + 4, maxLocals)
    }
}
```
代码中涉及到`asm` `MethodVisistor`的不少`api`，我查询，简单了解下，[博客链接在此](https://blog.csdn.net/qq_33589510/article/details/105273233)
解释一下用的的几个方法
- visitLdcInsn：访问ldc指令，向栈中压入参数
- visitMethodInsn：调用方法的指令，上面代码中，用来调用`LogisticsCenter.register(String className)`方法
- visitMaxs: 用以确定类方法在执行时候的堆栈大小。
### 小结
对这里我们就十分清晰了插入初始化代码的路径。
1. 首先是扫描所有的`jar`和`class`，找到对应的`routes`包名的类文件和 包含 `LogisticsCenter.class `类的`jar`文件。类文件名依据类别存放在ScanSetting中。
2. 找到`LogisticsCenter.class` ，对他进行字节码操作，插入初始化代码。
整个`register`插件的流程就完成了
# 六. ARouter idea 插件：arouter helper
该插件源码在 `arouter-idea-plugin` 文件夹下面
> 刚开始的时候编译老不成功，于是我修改了源码模块中 `id "org.jetbrains.intellij"` 插件的版本号，从0.3.12 改成了 0.7.3，果然就可以成功运行编译了。命令`./gradlew :arouter-idea-plugin:buildPlugin `可以编译插件。
### 插件效果
先看看这个用法效果。
安装很简单，只需要在插件市场搜索ARouter Helper 安装就行。
在安装了该插件之后，相关`ARouter.build()`路由的`java`代码那一行，行号右侧会出现一个定位图标，如下图所示。
![插件效果](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bd11ee46fb34274a81d81e2fa1a8ce4~tplv-k3u1fbpfcp-zoom-1.image)
点击定位图标，就能自动跳转到路由的定义类。
![跳转源码](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/320e8a683a604a9ea8f06785d1683fee~tplv-k3u1fbpfcp-zoom-1.image)

看完效果，我们直接看源码。
插件模块代码就一个类 `com.alibaba.android.arouter.idea.extensions.NavigationLineMarker`。

### 判断是否是ARouter.build
```java
NavigationLineMarker` 继承了`LineMarkerProviderDescriptor`，实现了`GutterIconNavigationHandler<PsiElement>
```
- `LineMarkerProviderDescriptor`：将小图标（16x16 或更小）显示为行标记。也就是该插件识别到navigation方法之后，显示在行号右侧的标准图标。
- `GutterIconNavigationHandler` 图标点击处理器，处理图标点击事件。
看看行图标的获取代码:
```kotlin
override fun getLineMarkerInfo(element: PsiElement): LineMarkerInfo<*>? {
    return if (isNavigationCall(element)) {
        LineMarkerInfo<PsiElement>(element, element.textRange, navigationOnIcon,
                Pass.UPDATE_ALL, null, this,
                GutterIconRenderer.Alignment.LEFT)
    } else {
        null
    }
}
```
1. 先是过`isNavigationCall`判断是否是`ARouter.build() `方法。
2. 然后配置`LineMarkerInfo` ，将`this` 配置为点击处理者
所以我们先看`isNavigationCall`：
```kotlin
private fun isNavigationCall(psiElement: PsiElement): Boolean {
    if (psiElement is PsiCallExpression) {
        ///resolveMethod:解析对被调用方法的引用并返回该方法。如果解析失败，则为 null。
        val method = psiElement.resolveMethod() ?: return false
        val parent = method.parent
         
        if (method.name == "build" && parent is PsiClass) {
            if (isClassOfARouter(parent)) {
                return true
            }
        }
    }
    return false
}
```
该方法判断是否调用的是`ARouter.build`方法,如果是就返回true。展示行标记图标。
### 点击定位图标跳转源码
接下来再看点击图标的相关跳转 `navigate`方法:
```kotlin
override fun navigate(e: MouseEvent?, psiElement: PsiElement?) {
    if (psiElement is PsiMethodCallExpression) {
        ///build方法参数列表
        val psiExpressionList = (psiElement as PsiMethodCallExpressionImpl).argumentList
        if (psiExpressionList.expressions.size == 1) {
            // Support `build(path)` only now.
            ///搜索所有带 @Route 注解的类，匹配注解的path路径有没有包含路径参数，包含的话就跳转
            val targetPath = psiExpressionList.expressions[0].text.replace("\"", "")
            val fullScope = GlobalSearchScope.allScope(psiElement.project)
            val routeAnnotationWrapper = AnnotatedMembersSearch.search(getAnnotationWrapper(psiElement, fullScope)
                    ?: return, fullScope).findAll()
            val target = routeAnnotationWrapper.find {
                it.modifierList?.annotations?.map { it.findAttributeValue("path")?.text?.replace("\"", "") }?.contains(targetPath)
                        ?: false
            }

            if (null != target) {
                // Redirect to target.
                NavigationItem::class.java.cast(target).navigate(true)
                return
            }
        }
    }

    notifyNotFound()
}
```
1. 获取build方法的参数，作为目标路径
2. 搜索所有带 @Route 注解的类，匹配注解的path路径有没有包含目标路径参数
3. 找到的目标文件直接跳转 `NavigationItem::class.java.cast(target).navigate(true)`
# 完结撒花
梳理完了一波Arouter的源码，耗费了不少精力。
却也学习到了一些之前了解不多的东西。
1. arouter的路由原理
2. apt 注解处理器
3. Gradle 插件+ asm 字节码扫描插桩
4. Idea 行标记插件和跳转
接下来要尝试动手搞个自己的gradle 和idea插件去。
当然是前提是先好好划水休息一下~

