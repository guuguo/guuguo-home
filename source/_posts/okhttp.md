# 【Android进阶】超级全-从okhttp的源码出发，了解客户端的网络请求

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZmYxMjIxYjQ3ODM0OTM0ZDkzZTQ4MDZkOWNlOGUwNmJfM2UwY2M1MWIyZjhkZThiZjg4NzVlNDkzNGEyNjljMzNfSUQ6NzEzNjExMzUzMzc1ODI5MTk2OV8xNjYxNTA2NDQxOjE2NjE1OTI4NDFfVjM" alt="boxcnhjp5TbFZz4paoaVGyeX7of" style="width:900px;height383.0px;" />



艳阳高照，温度高企。

然而对于知识与履历不佳的Android开发来说，却仿佛坠入了寒冬。

招聘市场能看到的安卓岗位基本上来来去去都是那几家公司，大公司不敢面，小公司待遇不满足。仿佛失业就摆在面前了。

所以能怎么办呢，

只能继续学习了。

OKHttp作为Android十分流行的网络请求框架，有着精妙的设计和丰富的功能。支持了缓存能力，重试重定向能力，还自己实现了一套网络连接传输能力。完美支持客户端的各种网络需求。

居家必备，不得不看。

# 准备：

- Okhttp依赖

Idea 创建java项目，依赖okhttp，可以直接在main方法中执行okhttp网络请求

```Groovy
implementation("com.squareup.okhttp3:okhttp:4.10.0")
```

- Okhttp源码

直接去 github download 

https://github.com/square/OkHttp

# 一，做一个同步请求，探索okhttp发起请求的过程

我们有一个get接口: **https://mock.apifox.cn/m1/810160-0-default/test**

请求会返回一个json :

```JSON
{"data":"hello"}
```

由此，我们开启http请求

## 发起HTTP请求

我们先来了解下这样的请求是如何在http报文中体现的

HTTP 请求报文由下面的四个部分组成

- 请求行  request line

- 请求头  header

- 空行

- 请求数据 data

在这个请求中，默认的请求头为空，因为是get请求，请求体也是空。

现在使用OKHTTP发起这样的请求：

```Java
public static void main(String... args) throws Exception {
  OkHttpClient client = new OkHttpClient();
  Request request = new Request.Builder()
      .url("https://mock.apifox.cn/m1/810160-0-default/test")
      .build();
  try (Response response = client.newCall(request).execute()) {
    ResponseBody body = response.body();
    System.out.println(body.string());
  }
}
```

`client.newCall(request).execute()`代码从构建`RealCall`到发起请求的调用步骤如下：

![c9975d46-d5fe-4fb4-8f4e-650827709042](/Users/guodeqing/Downloads/c9975d46-d5fe-4fb4-8f4e-650827709042.svg)

请求的发起从`execute`开始。分析下`RealCall`的`execute`方法，

- 第一个红框对各种超时进行了处理

- 第二个红框执行了网络的拦截链，直到响应结果返回

- 第三个红框中的client.dispather，则是记录了当前进行中的请求任务

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZjgyMDI1Zjc2MjRjYTYyNTViYjJlNTU0NGJmNjU2MzdfYjg2ZmZhMDA2MGJmZjlhNDYzYTRiOGVjMmEwNGU2MjRfSUQ6NzEzMDE0MzM3Mjc5OTgxOTc3N18xNjYxNTA2NDQxOjE2NjE1OTI4NDFfVjM" alt="boxcnpts6UL7mvtkiKPASGMLknb" style="width:837px;height278.0px;" />



跟踪请求的发起，我们在`RealCall.getResponseWithInterceptorChain`方法中看到了一系列责任链形成，并通过该链条将用户的请求一步步处理成为给用户的响应结果返回。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ODMxMGY3ZTQ4NDQ2MDA2MTI3Zjc0MTA1YTJmZDY2MjRfNzZjMGU3YjY0NGEwMDM0ZDBmZDIyNTQ3YzQ2ODVhZDRfSUQ6NzEzMDEyNzEzODc1OTkwMTIxMl8xNjYxNTA2NDQxOjE2NjE1OTI4NDFfVjM" alt="boxcneMIe9uDFSh9rmBXoWklvfd" style="width:551px;height244.0px;" />

## 拦截器责任链的串联

直接看链条中的最后一个拦截器`CallServerInterceptor`，**它对服务器进行了网络调用**。

断点`CallServerInterceptor.intercept`方法，看下他的调用堆栈：

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MDIxNGRmYTIwOGUyM2NmNzE2NjlkMGNkNDJjYzFhMDJfMDliYjQ4NTM5OGJhYmQyMGY0MTMyNjUyYTk0YjRkNTJfSUQ6NzEzMDEyODMyMzA1Nzk1ODkxNV8xNjYxNTA2NDQyOjE2NjE1OTI4NDJfVjM" alt="boxcnodiQ9ZcCme2QddAnlvPDLd" style="width:601px;height298.0px;" />

可以看到，每个拦截器通过调用`RealInterceptorChain`链的process方法，进行链条的传递。

顺序和前面拦截器添加的顺序一致，对应拦截器和作用如下：

1. `client.interceptors`  

用户的自定义拦截器，在所有拦截器之前，拦截的请求和响应都还是用户数据，未被封装处理。每次用户请求只会拦截一次，不参与重试的拦截。

2. `RetryAndFollowUpInterceptor`  

此拦截器从故障中恢复并根据需要遵循重定向。如果调用被取消，它可能会抛出IOException 。

3. `BridgeInterceptor` 

作为从应用程序代码到网络代码的桥梁，对用户的数据和网络的数据进行双向转换。首先根据用户请求构建网络请求。然后继续调用网络。最后从网络响应构建用户响应。

4. `CacheInterceptor`

处理来自缓存的请求并将响应写入缓存。

5. `ConnectInterceptor`

打开到目标服务器的连接并继续到下一个拦截器。网络可能用于返回的响应，或使用条件 GET 验证缓存的响应。

6. `if(forWebSocket) client.networkInterceptors`

WebSocket网络拦截器，除了网络请求拦截器外，该拦截器在其他所有拦截器之后。所以他会在网络连接等完成后，真正每次网络请求中进行拦截。也会在每次重试和重定向中拦截。

7. `CallServerInterceptor`

这是链中的最后一个拦截器。它对服务器进行网络调用



一个同步的任务发起，会直接开始依次处理请求。各个拦截器的细节内容繁多，后面一一分析。

# 二，做一个异步请求，探索okhttp的多任务请求机制

将之前的`execute()`请求执行方式换成`enqueue()`方法，网络请求就会以异步的形式被发起：

```Java
client.newCall(request).enqueue(new Callback() {
  @Override
  public void onFailure(@NotNull Call call, @NotNull IOException e) {
  }
  @Override
  public void onResponse(@NotNull Call call, @NotNull Response response) throws IOException {
    ResponseBody body = response.body();
    System.out.println(body.string());
  }
});
```

## Dispatcher 说明

我们再次深入`RealCall.enqueue()`方法的调用链，发现其中直接就创建了个`AsyncCall`，并交给了`Client.dispatcher`执行。

`AsyncCall`是`Runnable`的实现类，以便在需要的时候，将该异步请求交给线程池执行，真正的一步请求执行将会在后文提到。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=YTQwZDE2ZjAxODUwMzM5M2QxZWI5YWMyMjEwOTdjMTRfNzA4NWI4ODlhN2U2M2RiOWZkZjllZGFlMmYzZDJiZDdfSUQ6NzEzMDE1ODQ5Mzk2MTg5NTkzN18xNjYxNTA2NDQyOjE2NjE1OTI4NDJfVjM" alt="boxcnVhRGDK8jPUW0IM6YobkiNg" style="width:740px;height143.0px;" />

我们看看`dispather`变量的创建时机，是在Client.Builder创建的时候，直接实例化出来的`Dispatcher`对象

```Kotlin
//client.dispatcher
@get:JvmName("dispatcher") val dispatcher: Dispatcher = builder.dispatcher
//Builder.dispatcher
internal var dispatcher: Dispatcher = Dispatcher()
```

`Dispather`类的说明如下。

关于何时执行异步请求的策略。

每个调度程序都使用ExecutorService在内部运行调用。如果您提供自己的执行程序，它应该能够同时运行the configured maximum调用数。

很明显他是okhttp中异步的调度器，决定了okhttp异步请求的能力。

## Dispatcher 的请求发起和调度

回到`enqueue`方法

- 第一个红框将异步请求添加到待处理队列

- 第二个红框代码针对相同的host的请求，进行计数

- 第三个红框是调度器的关键方法，内部触发了异步请求的检查，会将符合条件的请求开始异步执行

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=YWRmZWY4ZjdjMzZlYjIwMjI2MzQ5ZGM2YTk5NDFmYTlfN2M3OGU0NjA2OGRkNDY4ZWE3MGExZGUwN2I3ZDE3MDhfSUQ6NzEzMDE2MzY2MjU2MDU5MTkwMF8xNjYxNTA2NDQzOjE2NjE1OTI4NDNfVjM" alt="boxcnhroGrHyyez52TaX3aeiY0e" style="width:676px;height291.0px;" />

- readyAsyncCalls （待处理异步请求队列）

- runningAsyncCalls （处理中的异步请求队列）

`promoteAndExecute()` 方法中会检查待处理请求列表，对当前同事执行的请求数量，以及单一host同事请求的数量进行阈值判断，如果满足条件就会将待处理的请求转为执行状态。并在检查完成后，调用`AsyncCall.executeOn(executorService)` 真正开始执行

默认的maxRequests值是64

maxRequestsPerHost值是5

如果需要自定义，可以对Dispatcher对象进行赋值

```Kotlin
private fun promoteAndExecute(): Boolean {
  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    val i = readyAsyncCalls.iterator()
    while (i.hasNext()) {
      val asyncCall = i.next()
///1.数量阈值判断检查
      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

      i.remove()
      asyncCall.callsPerHost.incrementAndGet()
      executableCalls.add(asyncCall)
      runningAsyncCalls.add(asyncCall)
    }
    isRunning = runningCallsCount() > 0
  }
///2.执行检查通过的请求
  for (i in 0 until executableCalls.size) {
    val asyncCall = executableCalls[i]
    asyncCall.executeOn(executorService)
  }

  return isRunning
}
```

到现在为止，调度树走到了红色小人所在的环节，后续异步请求的真正执行，就在于`AsyncCall.run`方法中，该方法被线程池调用。

![db635df8-69cd-4fc7-8edd-b3f2a365f19d](/Users/guodeqing/Downloads/db635df8-69cd-4fc7-8edd-b3f2a365f19d.svg)

## 异步请求的真正执行

`AsyncCall`的`executeOn`方法除了一些异常捕获和断言外，主要就是将自己的对象放到线程池中执行：

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZDFhMTI3YjE0ZGQ0YmZkMzY5NGY0NWZhZWJjMTc3NDVfMWYxYTAwNmMzNTE0MmQ2MjBiYjA1N2YzNTE0MzY2YTJfSUQ6NzEzMDE3ODkxNDQzOTM4MDk5NF8xNjYxNTA2NDQzOjE2NjE1OTI4NDNfVjM" alt="boxcnXHoyyJlyRgNBsOGqLk6a8b" style="width:475px;height109.0px;" />

所以真正开启异步网络请求的方法就在`AsyncCall.run`。

> 代码十分的似曾相识：这不就是同步请求 `RealCall.enqueue()`吗？

殊途同归。处理对当前线程命名，回调的调用外，一切都是熟悉的配方。

- 第一个红框对各种超时进行了处理

- 第二个红框执行了网络的拦截链，直到响应结果返回

- 第三个红框中的client.dispather，则是记录了当前进行中的请求任务

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ODE1NDVhMTJkZTUxNWZlYmRmNWRlYTEzZTU1NmI0Y2ZfYTQ2ZWI1YmRmZmJlYjg2ZjdkNjUzNjYyZmRlNWQ4NWJfSUQ6NzEzMDE3OTM5NjE5MjQzNjIyNV8xNjYxNTA2NDQ0OjE2NjE1OTI4NDRfVjM" alt="boxcngAJJUnA24g8047BxeRlFXd" style="width:792px;height617.0px;" />

至此，异步请求也走完了流程

焦点又回到了拦截器链。

**里面究竟怎么实现的网络请求？**

# 三，okhttp网络请求的真正劳动力：各大拦截器

## 3-1 重试重定向拦截器：RetryAndFollowUpInterceptor

[点击跳转github源码地址](https%3A%2F%2Fgithub.com%2FRunningTheSnail%2FOkhttp%2Fblob%2Fmaster%2Fokhttp%2Fsrc%2Fmain%2Fjava%2Fokhttp3%2Finternal%2Fhttp%2FRetryAndFollowUpInterceptor.java)。根据源码的逻辑，我们先画出对应的流程图

![e6397441-f2f3-4a1d-9a0b-104dcf39d292](/Users/guodeqing/Downloads/e6397441-f2f3-4a1d-9a0b-104dcf39d292.svg)

可以看到逻辑很简单，其中关键是以下两点：

1. 满足重试条件，就恢复请求重试。

2. 响应里面的信息满足重定向条件，就重定向

这两点控制了重试和重定向的所有逻辑，我们针对源码分析下

### 重试恢复请求条件

恢复请求的逻辑在recover方法中。

传递的参数分别是` e: IOException`：请求错误信息，`call:RealCall`：网络调用信息，`userRequest:Request`：请求数据，`requestSendStarted:Boolean`：是否已经发送过请求体。

总结下来的发送限制如下：

- 需要开启 **retryOnConnectionFailure **开关

- 如果发送过请求体，需要请求体支持多次发送才能重试

- 协议错误不可重试，SSL证书错误不可重试，SSL对等验证失败不可重试。io 发送中断不可重试

```Kotlin
///报告并尝试从与服务器通信失败中恢复。如果e是可恢复的，则返回 true；如果失败是永久性的，则返回 false。只有当主体被缓冲或在请求发送之前发生故障时，才能恢复带有主体的请求。
private fun recover(
  e: IOException,
  call: RealCall,
  userRequest: Request,
  requestSendStarted: Boolean
): Boolean {
  // The application layer has forbidden retries.
  if (!client.retryOnConnectionFailure) return false

  // We can't send the request body again.
  if (requestSendStarted && requestIsOneShot(e, userRequest)) return false

  // This exception is fatal.
  if (!isRecoverable(e, requestSendStarted)) return false

  // No more routes to attempt.
  if (!call.retryAfterFailure()) return false

  // For failure recovery, use the same route selector with a new connection.
  return true
}
```

### 重定向条件

默认情况下，当原始请求因以下原因失败时，OkHttp 将尝试重新传输请求正文：

- 陈旧的连接。该请求是在重用连接上发出的，并且该重用连接已被服务器关闭。

- 客户端超时 (HTTP 408)。

- Authenticator满足的授权质询（HTTP 401 和 407）。

- 可重试的服务器故障（带有Retry-After: 0响应标头的 HTTP 503）。

- 合并连接上的错误定向请求 (HTTP 421)。

## 3-2 桥接拦截器：BridgeInterceptor

该拦截器的主要功能就是将用户请求封装成网络请求，将网络响应封装成用户给用户的响应。主要逻辑如下：

1. 封装用户请求，Header完善

1. Content-Type，ContentLength，Transfer-Encoding从body中完善

2. Host，Connection，Accept-Encoding，Cookie，User-Agent完善

2. 网络请求

3. 封装网络响应body（如果有）

1. 去除 Content-Encoding ，Content-Length 响应头

2. 解压gzip

3. 设置contentType给body

## 3-3 缓存拦截器：CacheInterceptor->DiskLruCache

缓存的获取，只有在`okhttpClient`配置了`cache`的情况下才会生效。

`OKHttp`内置了`DiskLruCache`作为缓存工具类。

详解也可以参考他人的优秀文章

> [庖丁解牛Android源码-OKHttp源码](https://juejin.cn/post/6844903568839802893)

在`CacheInterceptor`中一开头就直接尝试从`DiskLruCache`中获取缓存

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=NDhkNThlNDY1Nzg1MjEzMWRlMGNjZjM2MmUwYjdmNThfYzgxYzExODBhMmM1ZGJiMGU0M2ZmNWE0NWZkYjc0MzNfSUQ6NzEzMDU2MjYyNzI5NDg4NzkzN18xNjYxNTA2NDQ0OjE2NjE1OTI4NDRfVjM" alt="boxcnh2QxNKX3dW3Zem44B7M7Ph" style="width:669px;height171.0px;" />

然后计算缓存策略，逐步执行：

1. 缓存没有

1. if(策略不请求网络)->返回失败响应

2. 缓存有

1. if(策略不请求网络)->返回缓存的响应

3. 请求网络，并返回

1. if(状态码==HRRP_NOT_MODIFIED【304】)->返回缓存响应

2. if(响应有正文，响应可缓存)缓存新的网络请求响应，并返回网络响应

3. if(响应不可缓存) 返回网络响应，移除缓存

上文可缓存和不可缓存的部分逻辑如下，就是对各种返回状态码进行判断。

**可缓存判断：**

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MmUxZmZiZDI5ZGYxOGU1Y2Q4NWYwMmI5NjNjNzIxYzJfYzhlZTc3NjZjZTczZjgzMzdhN2E5NWRhMWQ2OGI2YmZfSUQ6NzEzMDkyODYwMjg2MjQ1Mjc0MF8xNjYxNTA2NDQ0OjE2NjE1OTI4NDRfVjM" alt="boxcnb3pHRDImqf2V5d6qAsamGf" style="width:480px;height736.0px;" />

**不可缓存判断(POST:PATCH:PUT:DELETE:MOVE不可缓存)：**

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=Yzc5YTk2MDMxNTYwZTc2ZmU3Yjk1MzBmMmMyOTc5NTRfMGUzYzJjZmUwMzRlYTZmMzU2YmI0MmMyOTY3NWZkMWZfSUQ6NzEzMDkyODkzOTM3MjgyMjUyOV8xNjYxNTA2NDQ1OjE2NjE1OTI4NDVfVjM" alt="boxcnvGSAXOfuCVEcYjJUR3Jbrf" style="width:579px;height119.0px;" />

### DiskLruCache

`DiskLruCache` 是Andorid 硬盘缓存的优秀方案。OKHttp 将其io相关的操作交由okio实现。

OKHttp中从cache中获取Response的代码分为三步

1. 通过url生成的key获取`DiskLruCache.Snapshot`

2. 以snapshot获取metaData生成Entry

3. 从entry中获取Response

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=NmFiNzM1OGZjYzUzNDI1MTRkMzE5MmNjMjQ4NzU5ZjZfMDhmODM5ZDk5ZDJkZGNkZjQ5MTRlNzJiYzMwNWM3ZDZfSUQ6NzEzMDgxNjgyNTQ1MzU3NjE5M18xNjYxNTA2NDQ1OjE2NjE1OTI4NDVfVjM" alt="boxcn8vmMpQ03r3GlXPcRcjhCzf" style="width:487px;height526.0px;" />

### journal日志

`journal`是`DiskLruCache`缓存的的日志文件，缓存类将会从`journal`日志中读取所有的缓存操作，并生成

`lruEntries`链表。

下图是典型的`journal`日志样例

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=NTkxNzNiYzljM2NkMzlkNzRjYzU5MmNmM2ZlODE3ZWVfZmMxY2RiZTM2MzIxYWQwM2VkOTkzOGY1N2MyYWMxM2RfSUQ6NzEzMDgyMjIxMjY1MTAyNDM4Nl8xNjYxNTA2NDQ1OjE2NjE1OTI4NDVfVjM" alt="boxcn1SZ2xGP3dWIBKS1MXKDr6b" style="width:587px;height304.0px;" />

日志的前五行构成其标题。它们是常量字符串“libcore.io.DiskLruCache”、磁盘缓存的版本、应用程序的版本、值计数和空行。

文件中随后的每一行都是缓存条目状态的记录。每行包含空格分隔的值：一个状态、一个键(key)和可选的特定于状态的值。

- `DIRTY` 行跟踪正在积极创建或更新条目。每个成功的 DIRTY 操作都应该跟随一个 CLEAN 或 REMOVE 操作。没有匹配的 CLEAN 或 REMOVE 的 DIRTY 行表示可能需要删除临时文件。 

- `CLEAN` 行跟踪已成功发布并可被读取的缓存条目。发布行后面是其每个值的长度。 

- `READ` 行跟踪 LRU 的访问。 

- `REMOVE` 行跟踪已删除的条目

每次缓存操作都会更新附加日志文件。日志有时可能会因为删除多余的行而被压缩。压缩期间将使用一个名为“journal.tmp”的临时文件；如果打开缓存时该文件存在，则应删除该文件。

### Snapshot的获取

从`DiskLruCache`获取Snapshot，会先经历初始化`initalize`过程，然后再从`lruEntries`从获取对应的实体及其快照。而 `lruEntries` 是一个`LinkedHashMap`。他是一个有序的哈希链表，正式他的访问排序特性，决定了`DiskLruCache`的 LRU(Least Recently Used)近期最少使用特性。

关键点在于初始化过程，将会通过`journal`日志文件进行初始化。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=YTIwMDdiZjk1OTQxNDFkNzVjMWE1NTIxZWY0Y2M1NDJfMjE3NzViNTBmZDk3NzYwNDgzNTUwYWYwNDJmNWZmOGJfSUQ6NzEzMDgxODIwNDA1MDY2OTU3MF8xNjYxNTA2NDQ1OjE2NjE1OTI4NDVfVjM" alt="boxcne0pajVs1NF3IyhsjrjfXPk" style="width:439px;height439.0px;" />

通过查看`initialize`源码。可知初始化经历了三个过程：

1. readJournal() 读日志

2. processJournal() 处理日志

3. rebuildJournal() 重建压缩日志

对于`lruEntries`的填充就是在`readJournal() `期间，读取每一行Journal状态记录完成的。

具体代码在`readJournalLine()`方法中，简化细节如下：

```Kotlin

@Throws(IOException::class)
private fun readJournalLine(line: String) {
  //...
  //移除REMOVE状态的Entry
  if (secondSpace == -1) {
    key = line.substring(keyBegin)
    if (firstSpace == REMOVE.length && line.startsWith(REMOVE)) {
      lruEntries.remove(key)
      return
    }
  }
  //将正常状态的实体，加入到lruEntries中
  var entry: Entry? = lruEntries[key]
  if (entry == null) {
    entry = Entry(key)
    lruEntries[key] = entry
  }
  //对Dirty状态的可写，对READ状态的不处理，对CLEAN状态的读取状态标为正常
  when (firstSpace) {
    CLEAN -> {
      val parts = line.substring(secondSpace + 1).split(' ')
      entry.readable = true
      entry.currentEditor = null
      entry.setLengths(parts)
    }
    DIRTY -> entry.currentEditor = Editor(entry)
    READ -> {}
  }
}
```

就是逐行根据`journal`的状态填充`lruEntries`。

最后返回的`Snapshot`就是对应`key`的`Entry.snapshot()`;

该`Snapshot`将会打开所有流，以确保用户拿到已缓存完成的快照。

到此，缓存已经可以正常交由`CacheInterceptor`处理了

## 3-4 网络连接拦截器：ConnectInterceptor

网络连接拦截器主代码非常简单：

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZDM2ODcyZmE1YjA5ODY3MWJhZTdlNTlkZWFhYWMzZjJfYTJhMWM0NGQ5MDdhMWM5ZDIzYzY2ZGE4NzJkNjIxMTFfSUQ6NzEzMDkyOTk2MjIxOTgxNDk0MF8xNjYxNTA2NDQ2OjE2NjE1OTI4NDZfVjM" alt="boxcn8TWbe33y3SVxhyR3hBb7Zf" style="width:532px;height200.0px;" />

关键点就在于`Exchange`的初始化。

这一块和下面的网络传输实现非常强关联。所以和`CallServerInterceptor`一起分析。

## 3-5 网络传输拦截器：CallServerInterceptor

![e45bda7b-758d-4144-8f44-9fd470e6cf93](/Users/guodeqing/Downloads/e45bda7b-758d-4144-8f44-9fd470e6cf93.svg)

**`100-continue`**：HTTP/1.1 协议里设计 100 (Continue) HTTP 状态码的的目的是，在客户端发送 Request Message 之前，HTTP/1.1 协议允许客户端先判定服务器是否愿意接受客户端发来的消息主体（基于 Request Headers）。

即， 客户端 在 Post（较大）数据到服务端之前，允许双方“握手”，如果匹配上了，Client 才开始发送（较大）数据。

如果请求中有“Expect: 100-continue”标头，则在传输请求正文之前等待“HTTP1.1 100 Continue”响应。如果我们没有得到那个，返回我们得到的（例如 4xx 响应）而不传输请求正文。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MGM4OWJjMTQwYWVhNzI4ODFiMmNhNTE1MDZkZjlmZmVfMWQzOGIzZjdjYzllMjM2MDVmOGQ4NjJiMzM1NjNkZTVfSUQ6NzEzNDU5MjEyMDIxNjE0MTgyOF8xNjYxNTA2NDQ2OjE2NjE1OTI4NDZfVjM" alt="boxcnsyWXNQWaMsWz6FeTSKHpcg" style="width:798px;height226.0px;" />

### CallServerInterceptor请求流程

`CallServerInterceptor`中的网络请求全程通过`exchange`实现。

1. 对于有`body`的请求，检验`100-continue`的响应

2. 对于校验通过的请求再发送`requestBody`

3. 读取响应Header建立对应Builder

4. 读取响应body，建立Response返回

代码中一系列操作券是通过`exchange`完成，精简一下一个post请求就是这样子：

```Kotlin
///1. 写入请求头
val exchange = realChain.exchange!!
exchange.writeRequestHeaders(request)

///2. 开始请求，并写入请求bdy
exchange.flushRequest()
val bufferedRequestBody = exchange.createRequestBody(request, true).buffer()
requestBody.writeTo(bufferedRequestBody)
bufferedRequestBody.close()
exchange.finishRequest()

//3. 读取响应头
responseBuilder = exchange.readResponseHeaders(expectContinue = false)!!
exchange.responseHeadersStart()

var response = responseBuilder
    .request(request)
    .handshake(exchange.connection.handshake())
    .sentRequestAtMillis(sentRequestMillis)
    .receivedResponseAtMillis(System.currentTimeMillis())
    .build()
    
//3. 读取响应body
response = response.newBuilder()
    .body(exchange.openResponseBody(response))
    .build()
 
exchange.responseHeadersEnd(response)
var code = response.code
```

我们从`Exchange`的注释中，可以得知他的主要功能：

传输单个 HTTP 请求和响应对。将在实际处理I/O 的 `ExchangeCodec`上进行分层连接管理和事件。

意思也就是，他也没干啥事，所有的分层连接管理和事件一股脑交给了`ExchangeCodec`。

### 请求中的状态码校验

继续回到之前的网络处理，其中还夹杂了一些状态码和`Header`的校验，具体如下

`100`：前面有提到的100的校验。如果是100 需要重新获取对应的实际响应

`websocket & 101`：直接返回空的body

`Connection:close`：返回对应response，标记`noNewExchanges` 防止在此连接上创建进一步的交换

`204 | 205`：抛异常`**"HTTP 状态码 拥有 非空的 Content-Length"**`

### Exchange 和 ExchangeCodec

在`ConnectInterceptor`中，仅仅是调用了`Exchange`的初始化。其中实例化了`Exchange`。

将`RealCall` `eventLisener`  `exchangeFinder` 和 `codec`作为参数传输了进去。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=YTk3MjZkMmNiNTg2ZGZlNjMxNjU4YTkzM2ViNmYyY2RfMTRhODFkMTU3ZTY4ZGMyMTdiMDkxOTQzMDM5OGQ2MjhfSUQ6NzEzNDYzMDY5Mjc0Nzk4NDg5N18xNjYxNTA2NDQ2OjE2NjE1OTI4NDZfVjM" alt="boxcnRax6NdMnkHV6UhFkZVzHIg" style="width:621px;height455.0px;" />

- 其中`evenLisener`从`client.eventListenerFactory`中获得，可在`client build`的时候配置，放出事件执行的监听回调。

- `codec`作为具体网络io的工具类，由`exchangeFinder.find`获取。

- `exchangeFinder`则是在`RetryAndFollowUpInterceptor`拦截器中被初始化，代码如下。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZWQ3MmMwZWNlYzMwYzk0ZGQyOWNjZGNjZDE0ZWUzMDZfOGUwZDQ1OGQzNjllZmY4MTY5NGNkNTI1MjY3MmQ0YThfSUQ6NzEzNDYzMjEyNDA5OTY0MTM0OF8xNjYxNTA2NDQ3OjE2NjE1OTI4NDdfVjM" alt="boxcn2cCfhg4aYKcu608e04328e" style="width:787px;height491.0px;" />

通过断点调试，该方法在 `RetryAndFollowUpInterceptor` 每次进行请求和重试的时候被调用。根据`newExchangeFinder` 参数来决定是否创建。在重试的时候不重建，而在重定向或者第一次请求的时候则会重建。

`ExchangeFinder`的主要功能是通过一系列的策略满足连接的复用查找工作

尝试查找交换的连接以及随后的任何重试。这使用以下策略：

1. 如果当前调用已经有一个可以满足请求的连接，则使用它。对初始交换及其后续使用相同的连接可能会改善局部性。

2. 如果池中有可以满足请求的连接，则使用它。请注意，共享交换可以向不同的主机名发出请求！有关详细信息，请参阅RealConnection.isEligible 。

3. 如果没有现有连接，请列出路由（可能需要阻止 DNS 查找）并尝试建立新连接。当发生故障时，重试迭代可用路由列表。

如果在 DNS、TCP 或 TLS 工作正在进行时池获得了合格的连接，则此查找器将首选池连接。只有池化的 HTTP/2 连接用于此类重复数据删除。

可以取消查找过程。

此类的实例不是线程安全的。每个实例都被线程限制在执行call的线程中。

他的构造函数包含四个参数

- 第一个`connectionPool`连接池也可以在Client.Builder中配置。默认构造的代码如下，目前，此池最多可容纳 5 个空闲连接，这些连接将在 5 分钟不活动后被驱逐。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MjA0YTJkMzJiODJkZTNjNTVjNTdmYTAwNDJkZWJkOGNfMWIwYzdjNzkzZDQ2MTgxN2RhOWQzMmU1MzM3ZGRmNDNfSUQ6NzEzNDYzMzg4NTQyMTc5NzQwNF8xNjYxNTA2NDQ3OjE2NjE1OTI4NDdfVjM" alt="boxcnfe5YYB6Q7AGRYJnWlvOLcb" style="width:764px;height358.0px;" />

- 第二个参数`address`，提供了后续的代理Proxy，路由Route对象的生成，以及重要的寻址工作。

- 第三个`call`则是把`RealCall`作为参数传递进去

- 第四个参数`eventListener`提供了事件监听回调的途径

之后通过`ExchangeFinder.findConnection()`找到合适的`Connection`（找到后会自动连接），通过`newCodec`创建出了`ExchangeCodec`实例，并且内部通过`http2Connection`的判断做了http1和http2的适配。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MjcwMThkODY4ZGNmOWZlYjViOTlmNjNiMzVlNGZmM2NfNDQzMGQ1YTA5ZTZkODI0N2RlZTQ5ZWY5OGNmMzg0YzlfSUQ6NzEzNDYzNzMzMDE2ODEzNTY4NF8xNjYxNTA2NDQ3OjE2NjE1OTI4NDdfVjM" alt="boxcnvudtsDSVFxmUnJBonCle1g" style="width:726px;height360.0px;" />

留个问题，具体是怎么查找连接并进行三次握手的呢？

后面继续探索。

### HTTP1.1 和HTTP2的适配

对于未知`http2Connection`继续探索，断点他的唯一赋值处。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZmFmYWZmZTdjMGVlZGZmYWMyMTYyZjRkZjQyYWZjOTlfODhmMDdjN2E5NmFlYjliMDk5M2U2YzlkYWE2N2QzMDlfSUQ6NzEzNDYzODU2NDU0MTU5NTY1MV8xNjYxNTA2NDQ4OjE2NjE1OTI4NDhfVjM" alt="boxcnUb2D6AKoiPy0un7saxIItj" style="width:632px;height315.0px;" />

找到了关键点，是否调用`starthttp2`还是由`protocol` 的值判断。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MjBhNTg5NWJhMmY3MDM2MTJlZDQxNjQ2ZDc5MmNjZTRfNmQzMzY2ZjI4NTU4MTZlZGYzZmIwMGY2NjUyNTQ4MTNfSUQ6NzEzNDYzODc0MDYzOTA4ODY0MV8xNjYxNTA2NDQ4OjE2NjE1OTI4NDhfVjM" alt="boxcnzxPvnfKHNHQZr7wC9VMAPc" style="width:864px;height309.0px;" />

主要的原理就是。`route.address.protocols`里面包含了`http2`协议，就启动`http2`适配。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=Nzg4ZGEzNTNmMWQzZmFkOGU4ZTI5MGNhOWI2YWVmZWZfNjZjMTI0MTVlMDE2MDI1MDdjYjQ0ZDBiMzVlNTE0YWNfSUQ6NzEzNDYzODg3NjkxNjEzNzk4OF8xNjYxNTA2NDQ4OjE2NjE1OTI4NDhfVjM" alt="boxcnl5ww5147Whf7ygWsCLjTHg" style="width:544px;height270.0px;" />

那么问题来了，`route.address.protocols`的值又是什么时候被赋予的呢？这个问题先留着，带着疑问接着往下看。

### ExchangeFinder.find()找到RealConnection并开始三次握手连接

`ExchangeFinder.find()`就是`ExchangeFinder`的核心方法，承担了寻找合适的`Connection`的任务，寻找过程中发生了代理选择，dns寻址任务，并完成了和服务器的连接工作。

可谓是网络连接中非常关键的方法。

该方法的查找逻辑如下。

![dd8907ca-492a-4157-bdea-c52025f54b2a](/Users/guodeqing/Downloads/dd8907ca-492a-4157-bdea-c52025f54b2a.svg)

源码中包含了连接池复用，路由`Route`地址`Address`查询细节。

### FindConnection 生成RealConnection

Route是什么？

连接用于到达抽象源服务器的具体路由。创建连接时，客户端有很多选项：

- HTTP 代理：可以为客户端显式配置代理服务器。否则使用proxy selector 。它可能会返回多个代理来尝试。

- IP 地址：无论是直接连接到源服务器还是代理，打开套接字都需要一个 IP 地址。 DNS 服务器可能会返回多个 IP 地址进行尝试。

每条路线都是这些选项的特定选择。

问题来到了`RealConnection.route`，该变量在构造函数中被赋值。再回到`ExchangeFinder.findConnection()`方法，`RealConnection`实例的初始化就在其中。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=NGJlYWYwYjA0NWU0MjYyNWEzZTMyZTdmNTRhMDA4YTlfYTViYWRkZDgwZjVjZjU4NTliYjBhMGY2MGJhNDAwM2ZfSUQ6NzEzNDY3MDc2MTgwOTIzMTg3NV8xNjYxNTA2NDQ5OjE2NjE1OTI4NDlfVjM" alt="boxcnc53MA8AXsjVkqLrq7SDZvc" style="width:856px;height185.0px;" />

`Route`在前面的逻辑中通过`routeSelector`被初始化

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=YjU1MWFhY2VlYjkwNTQ5MTZhMTU2YWM1NmZkY2YxOTZfMzFkOWJlZGMwMjc0NmE5ODEwYjRhZDdiMTMwN2EwY2NfSUQ6NzEzNDY3MTA5MTQ1NzAxNTgxMV8xNjYxNTA2NDQ5OjE2NjE1OTI4NDlfVjM" alt="boxcnzi2EQHTbsieYfnxsKmQUTf" style="width:948px;height505.0px;" />

`Route`赋值的链条很明确：`RouteSelector().next()` 获取`RouteSelection`。

`RouteSelection.next()`获取`Route`。

其中的获取细节慢慢道来。

`RouteSelection`生成的时候，`Route`以`address` `proxy` 和`inetSocketAddress`为参数被构造出来。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZGIxZmY5NGM3NTI0MTc4ZTk0ZTE2YmE2NGE0YjNkZmJfMGZmYzVmZDRhZjBlZTM4MmQyMGQxMzg1ZmUxYjU4ZGNfSUQ6NzEzNTMzMTEyNjQ0NDg2NzU4Nl8xNjYxNTA2NDQ5OjE2NjE1OTI4NDlfVjM" alt="boxcn8tBh80ROH4FeF7u5HyuATg" style="width:1078px;height378.0px;" />

其中`RouteSelector.proxies`在RouteSelector构造的时候被初始化。由

`address.proxySelector.select(address.url) `取得。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MDVlZTU3MGZlMWIyMmJiODZkNmU0MDVhYjIyNTc4OGVfMWI4NTRmOGQwYzIxMzkzYWM5YTAzZGUxODEzZDQyNDVfSUQ6NzEzNTMzMzgwOTk0ODg1MjIyOF8xNjYxNTA2NDUwOjE2NjE1OTI4NTBfVjM" alt="boxcntrsH1dKmn7wvgLWuNqGGJd" style="width:1566px;height852.0px;" />

所以可见关键点还是`address`。

回到前文中`Exchange`那一小节。其中`address`就是在`ExchangeFinder`构造的时候被创建的。我们回顾下`RealCall.enterNetworkInterceptorExchange()`方法：

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MDBjMDUzNWIxMDQ0YjlmMjBhMTNlN2NlODNlMzQ3ZGRfOTJlMDQ0OGFlZTcwMTc3ZWJhMWNlYjJmMWUzM2E0MGVfSUQ6NzEzNTMzNzEzMjQwMjkxNzM3OV8xNjYxNTA2NDUwOjE2NjE1OTI4NTBfVjM" alt="boxcnvWwdT2lKnQN6uOgt7NQVAg" style="width:425px;height159.0px;" />

`Address`中很多的参数都是直接取的`client`的字段。而`OkHttpClient`的字段也大多从Builder中直接取过来。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=N2NlZWNhNDE1MzkyOTY3ZTNjMmRlZWRiN2U2YjUzZWNfY2VmZWQ4YWIxNjdlMzhiMGFlNmZiNDczMjkyYjI2MzNfSUQ6NzEzNTM1NjIzNDY4MTIyMTEyNF8xNjYxNTA2NDUwOjE2NjE1OTI4NTBfVjM" alt="boxcn0VkYSC8CRL52cSced74UGs" style="width:902px;height650.0px;" />

可以看到`Address.protocols`的默认值就是DEFAULT_PROTOCOLS，默认协议数组里面包含了HTTP_2和HTTP_1_1。所以会默认支持http的请求。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZWU2Yzg0ZDRhODMxOTkyMjM0ZTVkODZjMjk5NjA4NTBfMDMzYTE4ZTQ1NTY2MGUzY2VkNmI0MzI5ZWY3MDAxYThfSUQ6NzEzNTM1Njc3MTE3NTEzNzI4MV8xNjYxNTA2NDUwOjE2NjE1OTI4NTBfVjM" alt="boxcnOA4eGKv4KVFA0TwFGfk2je" style="width:1288px;height102.0px;" />

如果没有设置代理和代理选择器。其中默认的`proxySelector`则是获取的系统代理选择器。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=YzRhY2M4ODdiZTU3ZWU5NzZkYzdmNTNkYWRjNTc3ZTZfZmZmMzg2YTAzM2MxNTdhOTZiZTk5ZWJjZDk3YWI2YWNfSUQ6NzEzNTcyMTM3MDAxNjk1NjQyMF8xNjYxNTA2NDUxOjE2NjE1OTI4NTFfVjM" alt="boxcnOJ6PM1sUf6PN32cz8rSICh" style="width:1332px;height604.0px;" />

根据源码分析图解一下，Address选择代理，生成Route路由再到构建连接的过程如下所示：

![dd8907ca-492a-4157-bdea-c52025f54b2a](/Users/guodeqing/Downloads/dd8907ca-492a-4157-bdea-c52025f54b2a.svg)

至此，`ConnectInterceptor` 中短短一行代码的原理已经清晰了,只有看了细节之后才知道，原来这一行代码做了那么多事情。

```Kotlin
RealCall.initExchange()
```

完成连接后，会根据http协议版本生成`ExchangeCodec`用来进行实际的数据传输，实际实现类型是`Http2ExchangeCodec`和`Http1ExchangeCodec`。

在进行数据读写的时候，则利用`Http2Stream`或者`Http1Stream`进行数据传输。

Http2不同于http1的一请求一回应，而是分为多个`stream`流，每个`stream`可以有多个请求，多个响应（Server Push）。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=MzcwYjBhZDgzMjE1Njg1OTY3NzMyZTMzM2NmNjc4OWJfZjI2MGQzMDQ1MzA0MDEyZGE3ODFlNjI5ODU2MzdmY2NfSUQ6NzEzNjA2NDQwMDg1MDk2MDM4NV8xNjYxNTA2NDUxOjE2NjE1OTI4NTFfVjM" alt="boxcnbKeUBoqzXwPomiuhDI0Roc" style="width:894px;height357.0px;" />

响应的body则是以`okio` `Sink`的方式放回。

<img src="https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZDIxZTI3ZWM2MjVlZDJjMTI4ZjlhZDc0NThlOWM1ZDZfYjY5M2VhOWYzMGUwYWVhYzAxOTAzNjk4ZjMwNjg2NWZfSUQ6NzEzNjA2NDk3MjAzMTE2NDQyMF8xNjYxNTA2NDUxOjE2NjE1OTI4NTFfVjM" alt="boxcnkpMooHtG5YWN8MxTVYc9Vd" style="width:599px;height175.0px;" />

`Sink`相当于输出流（OutputStream），进行网络IO的输出。

所以`CallServerInterceptor`中的io读写也就有了眉目。就是利用`Exchange`中转，交由`Http2ExhcnageCodec`进行数据传输。其中再利用`Http2Stream`进行`okio`数据流的读写。

# 完结

至此，okHttp的源码详解告一段落。也是在学习的过程中了解到不少平时所知甚少的http状态码，也学习到了网络请求代理寻址过程。

收货颇丰，浅尝辄止。

`okhttp`的迷雾散去了一块。

`okio`的汪洋大海还是一片迷雾。