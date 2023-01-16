---
title:  Flutter 零成本搭建个人小博客
---
> 一直会喊我郭师傅的同学进了百度。为她感到高兴的同时心里也有一丝苦涩。 一样的职业，我还在小公司徘徊挣扎，人家已经脱离了苦海。 基础薄弱！静不下心！技术浅尝辄止！五年多的经验却还如新手一般只会搬砖！！！
>
> 既然**写博客**能沉淀自己的知识体系，成为面试的加分项。 那还不快去做！！

# 给自己搭一个零成本的博客网页，加油郭师傅！！

亲手打造 [**谷果之家**](http://guuguo.top/)

# 一 · 怎么下手？

身为开发者，我们都知道，搭建博客，至少需要一个服务器来存储博客网页，一个域名对外访问。在这个友好又和善的世界里，有没有慈善家给我们免费提供这种玩意呢？

我就找到了下面这几个备选方案

- [github page ](https://pages.github.com/)(移动用户需要翻墙，处于半墙状态，所以放弃了)
- Coding page (最后的选择)
- gitee page
- [leancloud](https://www.leancloud.cn/) 免费开发版
- 阿里云oss 存图片 (基本不用钱，使用空间小的时候不用钱)

试试用上面的东西来搞一下。

# 二. 尝试可行性

1. github 新建仓库，开启github pages [参考这个博客](https://zhuanlan.zhihu.com/p/38480155)
2. flutter构建简单的app demo
3. 编译出web产物push到github(或者gitee 或者coding)

热泪盈眶，经过一番操作后，成功在在网页上打开了我的[demo网页](http://guuguo.top/guuguo-home)。

![img](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_4dbd9025df9b475ba9860c3160ff0bcf~tplv-k3u1fbpfcp-zoom-1.image.jpg)

不过搞东西难免踩坑，在此记录下我遇到的几个小坑

- Flutter web 页面展示的时候中文先显示{口口}，一会儿后再正确展示

> 原因是flutter web 在运行的时候 如果渲染模式使用的是 canvaskit时就会有该bug，可以强制指定渲染模式为html

> 在编译的时候添加参数:`flutter build web --web-renderer html`

- 编译产物push到github 之后，打开网址展示的是一片空白？？？

> stack overflow 大神给出了[答案](https://stackoverflow.com/questions/64415471/flutter-web-on-github-pages-not-showing-content)，在flutter web html入口处删掉 `<base href="/">`。 再次push，一切变为正常。

- 阿里云解析解析域名不可用，最后还是使用新网解析
- Github pages 的页面有些网络无法访问，需要翻墙且服务满，最后切换到了coding的网站托管

# 三.搭建leancloud 后端云服务

在leancloud 中创建账号，新建app。使用开发版云存储，一天免费三万次请求~哈哈哈 完全够用了

> 最好有个已经备案的域名~ 如果没有已经备案的域名，leancloud只能使用国际版服务器，速度很慢~

#### 云存储建表

Leacloud 自带几个表，我再开发中用到了自带的_Role 和 _User表，顾名思义就是角色和用户表

在我的场景中用法十分简单。Role 规定了三种角色

1. 管理员 (admin) —————我自己
2. 普通用户(normal) ------普通用户 暂时用不上
3. 游客(tourist)———游客 有人访问博客时候默认登录的账号

然后就是用户表和 文章表了。不再赘述

![img](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_00f4a9003dcf4cef99c80762109aa106~tplv-k3u1fbpfcp-zoom-1.image.jpg)

#### 后端交互

因为leancloud 暂时没有`flutter web`的sdk,所以web前端与后端交互的方式是restfulapi的方式

> ps:**云存储有面向flutter 客户端的sdk,因为做的是web所以用不到**

使用方式参考[官方文档](https://leancloud.cn/docs/rest_api.html)

主要开发流程

- 首页 游客登录+获取文章列表
- 管理员通过隐藏入口登录，可以上传文章删除文章
- end

# 四.产出结果并部署

经过三四天时间的摸爬滚打加王者荣耀划水(成功打上了王者)，终于将勉强可用的版本部署到了coding的托管网站上。

页面简简单单，寥寥几张图直接可以展示。

[guuguo.top](http://guuguo.top/)

![img](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_96584ecd7e8d4573aa24ae68ff302115~tplv-k3u1fbpfcp-zoom-1.image.jpg) ![img](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_7c0711a8b3384aef9512310fa1f13a0b~tplv-k3u1fbpfcp-zoom-1.image.jpg) ![img](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_5c05374063204b2ea818f942a26d8a49~tplv-k3u1fbpfcp-zoom-1.image.jpg)

# 五.结尾

我总是追不上别人，在别人后面踉踉跄跄。看着别人的生活幸福，工资高企，默默羡慕

人总是要从自己的舒适区走出去，碰点壁再回头也比永远缩着要好

一旦迈出第一步，人总会硬着头皮继续走下去

人总是要从自己的舒适区走出去，碰点壁再回头也比永远缩着要好

一旦迈出第一步，人总会硬着头皮继续走下去