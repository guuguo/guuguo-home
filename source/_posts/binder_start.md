---
title: 【Android面试速学】进程间通讯学习，从Binder使用看起
---
![img.png](../images/start_binder.png)
`编写：guuguo` `校对：guuguo`

# 系列介绍
抱佛脚的目的只有一个，就是斩获自己期望中的offer.
# 前言
Binder 是安卓中非常重要的进程间通讯工具，通过`Binder` 安卓在`ServiceManager`中对外提供了一系列的服务。学习Binder，将很好地为我们学习`framework`开个好头。
# 名词解释
`IPC `：inter Process communication 进程间通讯。要说ipc，肯定要提到多进程，只有在多进程场景中，才需要进程间通讯。
多进程的情况分两种：

1. 第一种可能是应用因为自身原因需要采用多进程，可能是某个模块由于特殊原因需要跑在单独进程中，也有可能是为了向安卓系统获取多份内存空间，加大可使用的内存。
2. 第二种可能是需要向其他应用获取数据，两个不同的应用肯定是跑在不同的进程中，就涉及到进程间通讯，我们通过ContentProvider向系统查询数据的时候，其实也是一种进程间通讯。
   `RPC`：Remote Procedure Call 远程过程调用。不在同一个进程中的两台服务器A和B，如果A需要调用B的方法，就需要中间媒介来表达调用和传递参数。简单的理解是一个节点请求另一个节点提供的服务。
# Android 使用多进程
`Android` 开启进程方式很简单，在`AndoridMenifest`中给四大组件`Activity`、`Service`、`Receiver`、`ContentProvider`）指定 android:process 属性就可以了。
> 还有一种非常规的开启进程的方式，就是通过jni在native层fock一个新的进程。
最开始的时候，当我们需要使用多进程，兴致勃勃地在需要的组件上添加`process` 属性，以为就可以正常运行的时候，却会发现有些问题也随之而来。
1. 静态数据获取的时候得到的不是同一份。
2. `sharePreference` 等数据共享可靠性下降。
3. Application会多次创建。
4. 线程同步机制完全失效。
   这些问题产生的最主要原因就是不同的进程是跑在不同的虚拟机中的，彼此拥有独立的内存空间，所以不同进程对于静态值的修改只会影响自身进程。所以只要是通过内存来共享的数据，在四大组件的多进程中，都会失败，这也是多进程带来的主要影响。
   为了解决这些问题，便要使用安卓提供的多进程间通讯的方法。方法有很多，我们可以使用Intent,文件共享，SharePreference ，AIDL，Socket和基于Binder的Messager等方式来实现。安卓使用了linux内核，但是进程间通讯的方式不完全继承自linux。
   Binder 是安卓系统中最有特色的进程间通讯方案。
   这么说是因为在安卓中，通过独特的Binder，我们可以轻松实现进程间通讯。
# 目录
这里主要讲三个方面的内容
1. 序列化相关接口 Serializable
2. 序列化相关的 Parcelable。
3. 进程间通信的 Binder
   当我们想要使用Intent 和Binder 来传输数据的时候，就需要使用Parcelable 或者 Serializable。或者当我们想要将数据持久化或者通过网络传输的时候，也需要使用Serializable 来完成对象的持久化。
# Serializable
Serializable 是java提供的一个序列化接口，使用方式很简单，只需要让需要序列化的类实现该接口就行。
**序列化实现原理**：
1. 执行`ObjectOutputStram#writeObject`进行序列化
2. 内部会创建 `ObjectStreamClass` 实例，维护`serialVersionUID` 值，`writeObject`方法 和 `readObject`方法等
3. 判断并执行自定义的`writeReplace`方法，得到新的待序列化对象
4. 判断是否`Serializable` 实例，执行`writeOrdinaryObject` 方法写入序列化
   除了简单的实现 Serializable接口之外，还有一些可选的字段和方法可以进行自定义 ，如下面的代码所示：
```kotlin
class User(val name: String = "你好阿") : Serializable {
    private fun writeReplace(): Any? = null
    private fun readResolve(): Any? = null
    companion object {
        private val serialVersionUID = 1L
        private fun writeObject(ops: ObjectOutputStream) {}
        private fun readObject(ips: ObjectInputStream) {}
        private fun readObjectNoData() {}
    }
}
```
这些方法都在`ObjectStreamClass` 中进行维护和处理，实现版本维护和序列化自定义。
**比较重要**的是`serialVersionUID`字段，该字段标记了实体类的版本号，当反序列化的时候，通过比对版本号判断是否结构没有发生大的改变，完成反序列过程。
**提供该字段会让反序列化变得更加可靠可控。**一般在即时的数据传输过程中不需要提供该字段，系统也会自动生成该类的hash值并将它赋值给`serialVersionUID`。但是在一些持久化操作的时候，提供该字段就是一个比较重要的工作。因为如果没有提供，一旦有属性改动，类的`hash`结果也会改变，直接将导致之前的数据无法反序列化成功，抛出异常。这对于体验的影响是严重的。
**所以在需要持久化数据的场景下，需要提供**`**serialVersionUID**` **字段。**
Serializable 的序列化和反序列化代码很简单，下面举个简单的例子。
```kotlin
val file = File("aaaa")
file.createNewFile()
///序列化过程
ObjectOutputStream(FileOutputStream(file))
    .use {
        it.writeObject(User("张三"))
    }
///反序列化
val user: User? =
    ObjectInputStream(FileInputStream(file)).use {
        it.readObject() as User?
    }
println("序列化结果")
println(user?.name)
```
上述代码完成了`Serializable` 方式序列化的整个过程。很简单，直接使用`ObjectOutputStream` 和 `ObjectInputStream` 就可以了。
# Parcelable
介绍完Serializable，再来看看Parcelable 。Parcelable也是一个接口，只要实现这个接口，就可以实现序列化并通过intent和binder进行传递。看看一个经典的用法：
```kotlin
class User(val name: String? = "小王") : Parcelable {
    constructor(parcel: Parcel) : this(parcel.readString()) {
    }
    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeString(name)
    }
    override fun describeContents(): Int {
        return 0
    }
    companion object CREATOR : Parcelable.Creator<User> {
        override fun createFromParcel(parcel: Parcel): User {
            return User(parcel)
        }
        override fun newArray(size: Int): Array<User?> {
            return arrayOfNulls(size)
        }
    }
}
```
可以看到有这么四个自定义的方法，说明如下：
1. writeToParcel        实现序列化功能，写入到parcel
2. describeContents     提供内容描述，几乎都是返回0，只有在存在文件描述符的时候才返回1
3. createFromParcel        实现反序列功能，从序列化后的对象中创建原始对象
4. newArray        提供数组容器
   `Parcelale`和`Serializable` 都能实现序列化，怎么选呢？我们可以根据两种方案的区别来选择。`Serializable`使用简单，但是开销比较大，在序列化和反序列化的过程中需要大量的I/O操作。而`Parcelable` 的使用稍微复杂一些，但是性能比较好是Android 推荐的序列化方式。所以在进行内存传递的时候，可以使用`Parcelable`进行序列化，但是在涉及到持久化和网络传输的时候，`Parcelable`也能实现，但是使用会比较复杂，所以在这两种情况下建议使用`Serializable`。上面就是两种序列化方案的区别。
# Binder
Binder是Android专有的一种通信方式，Binder底层有kernel驱动的支持，设备驱动文件是/dev/binder，通过该驱动，android 在native层有一整套C/S架构，在java层也封装了一层相应实现。直观来说，Binder是Android中的一个类，它继承了IBinder接口。
Binder 可以进行跨进程通讯，也可以进行本地进程通讯。我们在写一个无须跨进程的本地服务 LocalService 时，可以直接获取Binder类来进行通讯。
基于binder,Android实现了多个ManagerService。因为android 系统有各种系统组件硬件需要暴露给其他进程，并且要集中管理，所以安卓在实现管理方案之后，再通过binder来暴露对应的接口服务，比如pms，ams，wms。
Android开发中，开发者对于Binder最直接的应用就是使用AIDL，相关使用流程大概有以下几步：
1. 创建aidl文件，声明方法
2. 继承生成的Stub类(Binder的抽象子类)，实现服务端操作的相关接口方法
3. 创建另一个进程中运行的service，在其onBind方法中返回该Binder实例
4. 使用该Service，通过`ServiceConnection#onServiceConnected`回调中得到的参数IBinder 获取定义接口的Binder实例。如`IHelloManager.Stub.asInterface(service)`。
5. 通过Binder实例进行远程方法调用。
## AIDL ( Android 接口定义语言 )
先看[谷歌官方开发者文档](https://developer.android.com/guide/components/aidl?hl=zh-cn)的介绍
我们可以利用AIDL定义客户端与服务均认可的编程接口，以便二者使用进程间通信 (IPC) 进行相互通信。在安卓中，编写进程间通信的代码较为繁琐，Android 会使用AIDL 帮我们处理这类问题。
我们先从一个典型的AIDL实例入手来探究。
### 定义AIDL接口
我们来定义一个 aidl 接口文件
```kotlin
//IHelloManager.aidl
package top.guuguo.wanandroid.tv;
import top.guuguo.wanandroid.tv.User;
interface IHelloManager {
    User getFriend();
    void setFriend(in User friend);
}
//User.aidl
package top.guuguo.wanandroid.tv;
parcelable User;
```
其中用到了User对象，所以上文还定义了该`User.aidl`，该对象实现Parcelable接口。
我们找到`generated/aidl_source_output_dir`观察对应生成的java 类:`IHelloManager.java`。

```kotlin
public interface IHelloManager extends android.os.IInterface {
    /**
     * Default implementation for IHelloManager.
     */
    public static class Default implements top.guuguo.wanandroid.tv.IHelloManager {
        /***/
    }
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements top.guuguo.wanandroid.tv.IHelloManager {
        /***/
        private static class Proxy implements top.guuguo.wanandroid.tv.IHelloManager {
            /***/
        }
    }
    public top.guuguo.wanandroid.tv.User getFriend() throws android.os.RemoteException;
    public void setFriend(top.guuguo.wanandroid.tv.User friend) throws android.os.RemoteException;
}
```
可以看到生成了`IHelloManager` 接口，实现`IInterface`。可以看到默认生成了三个该接口的实现类。`Default`，`Stub` 和`Stub.Proxy`。
`Stub` 是一个`Binder`类,是一个实例是服务端对象。`Stub.Proxy` 是`Proxy`的服务端代理类，其中执行方法的时候，调用了服务端的transact方法进行进程间数据交互转换，这两个实现类就是`IHelloManager`的核心类。
看看Stub类的代码：
```kotlin
public static abstract class Stub extends android.os.Binder implements top.guuguo.wanandroid.tv.IHelloManager {
    private static final java.lang.String DESCRIPTOR = "top.guuguo.wanandroid.tv.IHelloManager";
    public Stub() {
        this.attachInterface(this, DESCRIPTOR);
    }
    public static top.guuguo.wanandroid.tv.IHelloManager asInterface(android.os.IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof top.guuguo.wanandroid.tv.IHelloManager))) {
            return ((top.guuguo.wanandroid.tv.IHelloManager) iin);
        }
        return new top.guuguo.wanandroid.tv.IHelloManager.Stub.Proxy(obj);
    }
    @Override
    public android.os.IBinder asBinder() {}
    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
        java.lang.String descriptor = DESCRIPTOR;
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(descriptor);
                return true;
            }
            case TRANSACTION_getFriend: {
                data.enforceInterface(descriptor);
                top.guuguo.wanandroid.tv.User _result = this.getFriend();
                reply.writeNoException();
                if ((_result != null)) {
                    reply.writeInt(1);
                    _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                } else {
                    reply.writeInt(0);
                }
                return true;
            }
            case TRANSACTION_setFriend: {
                data.enforceInterface(descriptor);
                top.guuguo.wanandroid.tv.User _arg0;
                if ((0 != data.readInt())) {
                    _arg0 = top.guuguo.wanandroid.tv.User.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
                this.setFriend(_arg0);
                reply.writeNoException();
                return true;
            }
            default: {
                return super.onTransact(code, data, reply, flags);
            }
        }
    }
    static final int TRANSACTION_getFriend = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_setFriend = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    public static boolean setDefaultImpl(top.guuguo.wanandroid.tv.IHelloManager impl) {
        if (Stub.Proxy.sDefaultImpl != null) {
            throw new IllegalStateException("setDefaultImpl() called twice");
        }
        if (impl != null) {
            Stub.Proxy.sDefaultImpl = impl;
            return true;
        }
        return false;
    }
    public static top.guuguo.wanandroid.tv.IHelloManager getDefaultImpl() {
        return Stub.Proxy.sDefaultImpl;
    }
}
```
下面介绍一下`Stub`类中成员的含义：
- ### DESCRIPTOR
```
Binder`的唯一标识，一般是当前Binder的类名。本例是`"top.guuguo.wanandroid.tv.IHelloManager"
```
- ### asInterface(android.os.IBinder obj)
将服务端的Binder对象转换成对应AIDL接口对象。通过`queryLocalInterface`区分进程，如果双端是在同一进程中，返回的对象就是Stub对象，如果是在不同的进程中，则返回其Proxy代理对象。
- ### asBinder
返回当前Binder实例
- ### onTransact
该方法对传输的数据进行序列化和反序列化操作。完整的方法是`public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)`。该方法中，通过code定位客户端所请求的方法是什么，接着从data中取出方法所需的参数，然后执行目标方法，如果目标方法有返回值，向`reply`写入返回结果。
该方法如果返回false，那么客户端的请求就会失败。我们可以在该方法做一些调用限制，避开一些不期望的进程调用该方法。
- ### `Proxy#getFriend` 和`Proxy#setFriend`
这两个代理方法先处理传入参数，将其写入到`Parcel`中，然后调用`mRemote.transact`发起RPC(远程过程调用)请求。同时当前线程挂起，直到RPC过程返回后，再继续执行当前线程，并取出`reply`的返回结果。反序列化后返回数据。
## bindService
通过上面AIDL 及其生成代码的分析，我们知道了AIDL 只是个便于我们快速生成Binder通讯模板代码的方式。我们要在相关组件中使用该Binder进行IPC时，需要通过绑定服务获取`Binder`实例。
下面是绑定服务获取`binder`的相关代码:
```kotlin
val connection = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        "onServiceConnected".toast()
        binder = IHelloManager.Stub.asInterface(service)
    }
    override fun onServiceDisconnected(name: ComponentName?) {
        "onServiceDisconnected".toast()
    }
}
override fun onStart() {
    super.onStart()
    val intent = Intent(this, HelloService::class.java)
    intent.action = ":startHello"
    bindService(intent, connection, BIND_AUTO_CREATE)
}
override fun onStop() {
    super.onStop()
    unbindService(connection)
}
```
从上面的代码中，我们就可以通过`bindService` 获取`aidl`定义的`Binder`实例。通过该Binder实例，直接就可以对远程进程进行方法调用了。绑定服务具体的流程是什么呢？
现在看一下整个调用路径
1. 发起绑定服务： `mBase.bindService`
2. 定位到具体的绑定方法：经过查阅`Activity#attach`方法、`ActivityThread#performLaunchActivity`方法和 `createBaseContextForActivity` 方法，得知`mBase`是`ContextImpl`实例。
   `mBase.bindService`调用到了`ContextImpl#bindServiceCommon`方法
1. 获取`ActivityManager` Binder代理对象：在`ActivityManager.``*getService*``()` 方法中，从`ServiceManager.getService(Context.ACTIVITY_SERVICE)`获取IBinder实例`(BinderProxy)`
2. 通过 `ActivityManager` 调用`ActivityManagerService`的绑定服务方法，进行绑定服务。
   查阅源码和网络搜索中发现，获取`Binder`和`binder`通信原理，涉及到AOSP源码中的`ServiceManager`，`Binder`的` native C/S`实现等，暂不学习，放到以后针对性得学习aosp中的binder通信机制。对应地，先学习一下[Skytoby](https://skytoby.github.io/)大佬的[Binder机制分析文章](https://skytoby.github.io/2020/深入理解Binder机制6-总结篇/)，了解个大概。
   借用作者的Binder机制结构图如下：
   ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/370a226ab0e94b09be597b0895cdb15e~tplv-k3u1fbpfcp-zoom-1.image)
   接下来再看，以及如何手写binder的实现。
## 手写Binder实现类
通过上面的分析，我们大致了解了`Binder`的工作机制。所以我们尝试一下不使用aidl，使用binder进行进程通讯。
基础的实现只需要写三个类就行
1. 定义接口类
```kotlin
interface IActivityManager : IInterface {
    fun startActivity(code: Int): String?
}
```
1. 服务端Binder抽象类，在onTransact 完成远程输送的数据反序列化以及服务端执行任务的工作。
```kotlin
abstract class ActivityManager : Binder(), IActivityManager {
    companion object {
        val DESCRIPTOR = "top.guuguo.aidltest.IActivityManager"
        val CODE_START_ACTIVITY = FIRST_CALL_TRANSACTION + 0
        fun asInterface(obj: IBinder?): IActivityManager? {
            if (obj == null) return null
            return (obj.queryLocalInterface(DESCRIPTOR)
                ?: Proxy(obj)) as IActivityManager
        }
    }
    override fun asBinder(): IBinder {
        return this
    }
    override fun onTransact(code: Int, data: Parcel, reply: Parcel?, flags: Int): Boolean {
        when (code) {
            INTERFACE_TRANSACTION -> {
                reply?.writeString(DESCRIPTOR);
                return true;
            }
            CODE_START_ACTIVITY -> {
                data.enforceInterface(DESCRIPTOR)
                reply?.writeNoException()
                reply?.writeString(startActivity(data.readInt()))
                return true
            }
        }
        return super.onTransact(code, data, reply, flags)
    }
}
```
1. 客户端代理类(完成数据的序列化反序列化工作，具体交给代理的对象完成)
```kotlin
    class Proxy(val remote: IBinder) : ActivityManager() {
        override fun startActivity(code: Int): String? {
            val params = Parcel.obtain()
            val reply = Parcel.obtain()
            params.writeInterfaceToken(DESCRIPTOR)
            params.writeInt(code)
            remote.transact(CODE_START_ACTIVITY, params, reply, 0)
            reply.readException()
            val str = reply.readString()
            params.recycle()
            reply.recycle()
            return str
        }
        override fun getInterfaceDescriptor(): String? {
           return DESCRIPTOR
        }
        override fun asBinder(): IBinder {
            return remote
        }
    }
```
通过`InterfaceToken`标记,完成服务端和客户端相关工作。 在具体使用的时候，实现ActivityManager，完成服务端的任务自定义实现如下:
```kotlin
inner class HelloManagerImpl : ActivityManager() {
    override fun startActivity(code: Int): String? {
        return "progress:" + getProcessName(baseContext)
    }
}
```
完成了编写，同样地通过`bindService` 就可以绑定服务获取Binder进行进程间通讯了。
调用例子中的` binder?.startActivity(1)` 便可以获取到对应的字符串结果。

# 完结
Binder的使用方式大概学习了一遍，也手写了模拟`ActivityManagerService`的Binder实现。大概了解了如何通过Binder来进行进程间通讯。
同样的，安卓系统中利用Binder对外提供服务的例子比比皆是，安卓通过`ServiceManager` 提供了比如`PackageManagerService` `WindowsManagerService`等服务。如果了解了这些框架层的实现，对于我们的开发之路将会很有帮助。
了解Binder使用是个起步，接下来朝着进阶出发。

> 这样的话面试又会顺利一分吧。

