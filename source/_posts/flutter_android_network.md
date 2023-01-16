---
title: 【修修补补】flutter和android项目使用国内镜像和梯子，完美下载依赖
---
![烦人的网络问题](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_103830-99fdaa4f69de1172.jpg)

> 我遇见过一个又一个的坑，有些坑跳过去了。 而有些坑直直得摔在里面，久久未能直起身来。

这篇文章分享如何设置flutter和android镜像仓库。分享绕网工具的使用，以及使用 `proxifier` 针对应用程序或者域名和ip做单独的代理设置（该工具支持mac 和 windows）。
# 前言
最近换工作了，接手的是一个持续开发维护了两年多的flutter项目。里面有不少的依赖库，安卓的，flutter的都有。
每次接手新项目下载各种依赖库都是一个很麻烦的事情，毕竟flutter和安卓都是谷歌的。然而google在国内显然是不可达状态。
之前外包到大公司干活，人家网络环境自带翻墙，自然没有这些问题，开发起来爽歪歪。现在回归正常的网络环境，网络问题就又来了。

我刚来报到，自然是领电脑，配置环境，下编译器一套齐活。然后就开始直面网络问题。
# 第一步：配置flutter 依赖仓库镜像
这一步flutter 官网就和我们说了，自然配置环境的时候直接就做了。步骤如下：
先是配置环境变量：我是zsh环境，所以直接在`~/.zshrc`中添加下面的配置代码。
```groovy
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```
然后通过`source ~/.zshrc` 直接生效环境变量
接下来再使用flutter pub get 命令的时候会生成`pubspec.lock`,里面记录了当前依赖项的详细配置信息。
下面就展示了相关依赖信息，包含了版本号，以及描述信息等
![pubspec.lock](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_103830-2c22f9c2238b827e.jpg)
上面也能看到，url已经是 https://pub.flutter-io.cn 了，而不是https://pub.dev 。说明仓库已经切换到更快的镜像仓库了。
# 第二步：使用[阿里云公共代理库](https://developer.aliyun.com/article/754038)解决大部分maven仓库依赖下载问题
flutter项目中有用到许多android插件的开发，里面自然也依赖了一些第三方库，比如友盟之类的。也有不少依赖直接使用的是jcenter仓库。然而jcenter和bintray已经要被关闭了。
下面是jcenter的关闭时间表
| 2021 年 3 月 31 日  | JCenter 将不再接受任何提交。                                 |
| ------------------- | ------------------------------------------------------------ |
| 2021年4月12日、26日 | 该服务将关闭以提醒用户有关将于 5 月 1 日停用的服务。（具体营业时间将在 Bintray 状态页面中公布。） |
| 2021 年 5 月 1 日   | Bintray 服务将不再可用。                                     |
| 2022 年 2 月 1 日   | JCenter 将不再可用于非 Artifactory 客户端。                  |

很明显能看到，在2021年9月2号的现在。jcenter已经凉凉了。
所以同事已经下过依赖的没事，我这个新环境就下载不到jcenter依赖了。报错
![错误提示](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_103830-903f643c1e509194.jpg)
`connect to maven.google.com timed out`之类的。
这时候可以利用[阿里云镜像仓库](https://developer.aliyun.com/article/754038)来替代jcenter甚至是google仓库，完美解决大部分依赖的网络问题。
我们找到对应的android项目：
在 `project/android/build.gradle`中修改：

```groovy
buildscript {
    repositories {
        maven{ url 'https://maven.aliyun.com/nexus/content/repositories/google'}
        maven{  url 'https://maven.aliyun.com/nexus/content/groups/public'}
        maven { url 'https://repo1.maven.org/maven2/' }
}
allprojects {
    repositories {
        maven{ url 'https://maven.aliyun.com/nexus/content/repositories/google'}
        maven{  url 'https://maven.aliyun.com/nexus/content/groups/public'}
        // 友盟统计
        maven { url 'https://repo1.maven.org/maven2/' }
    }
}
```
public 是代理的jcenter 和central仓库，google 就是代理的google()仓库。
这样配置能够解决大部分问题。
Tips: 中间碰到一个友盟依赖仓库的问题，也是由于jcenter不维护了，所以友盟更换了仓库地址
说明在这里：[友盟仓库迁移说明](https://info.umeng.com/detail?id=443&cateId=1)
只需要找到gradle中的：
```groovy
maven { url 'https://dl.bintray.com/umsdk/release' }
```
将其改为新仓库地址就行了。
```groovy
maven { url 'https://repo1.maven.org/maven2/' }
```
# 第三步：代理走不通的网络请求
如果一些依赖确实需要走google仓库等，但是网络不可及咋办呢。老老实实弄绕网工具吧。
1. 弄个绕网工具服务器
   1. [购买三方绕网工具服务，链接是我自用的推荐](https://waimaolove.me/auth/register?code=jlWb)  （这个网站也可以下载v2ray 的各个平台客户端）
   2. 自己购买国外服务器或者vps搭建ssr或者v2ray(我以前亚马逊，谷歌和微软azure各白嫖了一年，可以去看看官网新用户免费策略)
2. 使用proxyfier 工具分析那些域名请求不可达
3. 针对不可达网络专门设置代码
   1. 设置v2ray 或者 ssr的 pac代理，对于google.com 等不可达网络设置代理(pac 可以使用gwfList的地址。[被墙的地址列表](https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt))
   ![pac设置](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_103830-c0ef0c174c9b3991.jpg)

1. 使用proxifier设置代理规则,下图针对 *.google.com 设置了代理
![proxifier主界面](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_103830-8f9ad3b68a1e84f9.jpg)
![配置google代理](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_103830-093019adb27d09f2.jpg)
# 最后
终于成功下载完所有依赖，到了编译步骤。
然而又报错了。
排查了半天，我将我下载的jdk16换成jdk11,在`~/.zshrc`中配置好环境变量。
终于成功跑起了app。
完事。