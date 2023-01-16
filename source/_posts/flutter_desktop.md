---
title: 【flutter起步走】自家博客网站显示不出图片？搞个客户端吧！
---
> 噼噼啪啪，键盘被飞快地敲击。代码像流水一样流淌在屏幕上。不一会儿，这个十分复杂地helloword程序就被我轻松地实现了。
> ctrl+R 运行程序
> 随着编译日志地持续输出
> 程序崩溃了



# 一. 什么问题

就在已经过去的不久前，我兴致勃勃地用 flutter 搭了一个自己的免费（白嫖）[博客网站](http://guuguo.top)，一切都挺好，但是有个问题却出现了。

那就是跨域限制带来的图片无法访问。

**而这个问题，掘金，简书都有自己的解决方案。**



# 二.解法思考

**先说结论:给博客做一个windows和macos的桌面客户端**

这个问题下意识想到的解法就是在服务端解析替换相关链接。分以下三步

1. 在上传文章的时候触发解析匹配图片url
2. 批量下载匹配到的图片
3. 批量上传下载的图片到云存储服务器
4. 替换 `markdown` 文章内的图片链接，并保存文章

这样就将跨域的他站图片变成了不跨域的本站资源(当然文章和图片都是自己的)

然而行不通，**本人搭建的免费博客并没有用到计算服务器**。

### 改进方案

**借助flutter跨平台，在客户端进行上述的几步操作。**

上面的方法行不通，是因为没有计算服务器，并且跨域请求也不能在网页客户端进行。

那能不能绕过浏览器限制，借助flutter强大的跨平台能力，构建出一个能进行解析下载上传的客户端app呢？答案是肯定的。

# 三.flutter 项目中增加macos 和 windows 平台支持

[Flutter 官方文档中写明的方式开启flutter 中的macos 和windows支持](https://flutter.dev/desktop)，步骤如下

#### 编译前提

Macos 需要安装xcode(有插件的话需要安装cocoaPods)

Windows 需要安装 visual studio

#### 开启flutter支持

```
$ flutter config --enable-windows-desktop
$ flutter config --enable-macos-desktop
$ flutter config --enable-linux-desktop
```

#### 创建平台相关源码，在项目目录敲以下代码(注意后面有个点)

```
flutter create --platforms=windows,macos,linux .
```

#### 然后重启ide，就可以run 代码到对应平台了。项目结构大概长这样

```
project
    -lib
    -android
    -ios
    -macos
    -windows
```

# 四.添加解析功能，编译出pc app

> 工作量只有以下几步

1. 下载图片  easy
2. 正则解析图片链接  easy
3. 上传图片  **hard**
4. 替换链接

**然而上传图片看起来简单，却花费了我好长的时间**

关键点是阿里云oss的api尝试了好久~

具体过程看[Flutter 使用api 上传图片到阿里云OSS ](https://juejin.cn/post/6969510221395787806)

**下载解析和替换代码先随便写写个支持掘金和简书的,代码如下**

```dart
Future dealReplaceUrl() async {
  //正则获取文章中的图片所有链接
  final images = await _checkContentImgUrl();
  //遍历处理图片
  for (var imgUrl in images) {
    if (imgUrl?.isNotEmpty != true) continue;
    //分割链接获取文件名
    final name = path.split(imgUrl!.split('?').first).last;
    //下载图片文件 到本地缓存文件夹
    String filePath = await DioHelper.getSaveCachePath(fileName: name);
    debug("filePath" + filePath);
    if (!await File(filePath).exists()) {
      await DioHelper.get().download(imgUrl, savePath: filePath);
    }
    //上传图片文件到oss
    var url = await DioHelper.get().uploadToOss(File(filePath));
    //替换图片链接成自己的
    articleEntity.content = articleEntity.content!.replaceFirst(imgUrl, url);
  }
}
```

获取图片链接的正则如下

```dart
///匹配 [!img](后面的链接
var reg="(?<=!\[img\]\()https?.+?(?=\))"
```

#  五.即将完成，门面图标替换

客户端都生成了，看默认的flutter图标一下子有点丑丑的了，去给自己找个好看的图标替换一下吧。

直接去iconfont网站找一个好看的多色图标直接用。

### [iconfont-阿里巴巴矢量图标](https://www.iconfont.cn/)

> *iconfont*-国内功能很强大且图标内容很丰富的矢量图标库，提供矢量图标下载、在线存储、格式转换等功能。

Macos 的图标生成可以去App Store 随便搜索一个icon生成工具，下面以iExportHelper 为例：

![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=MjQzNmEyZTEyOGE4OTg1MzhkOWFhMTYxMDdmNDJjMjVfdzlrWnhDV2c2YlZSU3VYSENlTGtJUGR4eUtIdDdvQmJfVG9rZW46Ym94Y25mSGl5T0pIV0E1aHdaT1RxWDhCZ3ZrXzE2MjI3NzcwNzE6MTYyMjc4MDY3MV9WNA)

拖入图片点击生成，生成的对应`AppIcon.appiconset` 替换对应平台代码文件夹下面的图标文件。

如 替换` macos->Runner->Assets.xcassets` 的 `AppIcon.appiconset`。



# 五.结尾

> 噼噼啪啪，键盘被飞快地敲击。代码被成功运行，macos上启动了我的博客客户端。点击编辑发布，看到文章的第三方图片链接被成功替换成自己oss的链接。
> 心里松下了一口气。
> 文章里能成功显示图片了