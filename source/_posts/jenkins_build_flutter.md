---
title: 【小试牛刀】自动化第一步，jenkins来搞事，flutter自动构建部署
---
> 我亲自拷贝，亲自粘贴，亲自打包。这样勤劳地干活，总得给我多发点工资啊老板
> 
> 拿来吧🤞

辛辛苦苦打完代码，还要亲自动手打包。那可太累了。现在的自动化构建工具非常方便，用jenkins弄个线上自动构建可以大大节约平时的时间。
这篇文章讲述了从工具安装环境安装，脚本编写到最后服务器部署的方方面面，看完自动化第一步简直so easy。
# 目录
1. 下载安装jenkins
2. 配置初始化jenkins
3. 创建git仓库的自动化构建
4. 服务器安装各种环境，以及添加自动打包构建flutter apk脚本
5. 使用nginx 服务器开放打包结果以供下载
6. 企业微信，钉钉或者飞书通知编译部署结果

# 一,下载安装jenkins
我的阿里云ecs服务器之前安装的是centos 6系统.
### 一个插曲:
刚开始尝试 在centos 6 服务器中安装jenkins,但是yum的阿里云镜像源已经不维护centos 6的了.想想反正里面没东西,就先把ecs服务器的系统重装到centos8了.
### 需要什么?:
先搜索[linux Jenkins安装教程](https://juejin.cn/post/6844903878987612174)。
安装jenkins需要依赖于其他运行环境,拉取代码也需要相应的git工具,总结一下 需要安装的库包括如下:
1. Java
2. git
3. Jenkins
### 安装java
安装之前先检查一下系统有没有自带open-jdk
直接在终端输入java命令,如果当前有java环境,就会直接输出一串东西,如果没有的话,就会提示找不到命令.
我们利用yum安装java
```shell
# 1更新yum源
# 2检查yum中有的java库
# 3安装java11
sudo yum update
sudo yum list java*
# 安装其他版本的java容易出错
sudo yum install java-1.8.0-openjdk-devel
```
 这样 java就成功安装到`usr/lib/java` 中了,这个路径默认就在环境变量中的.
### 安装git
Jenkins拉取git仓库代码的话肯定需要这个工具，直接使用yum安装
```shell
yum install git
```
### 安装jenkins
以下命令 依次执行，先是下载jenkins 依赖 加入到yum仓库中，再是通过yum安装jenkins。
```shell
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum install jenkins
```
经过这三个命令，理论上就安装成功了。

# 二,配置初始化jenkins
列出jenkins 相关的目录 ：
- `/usr/lib/jenkins/`：jenkins安装目录，war包会放在这里
- `/etc/sysconfig/jenkins`：jenkins配置文件，“端口”，“JENKINS_HOME”等都可以在这里配置。
- `/var/lib/jenkins/`：默认的JENKINS_HOME。
- `/var/log/jenkins/jenkins.log`：jenkins日志文件。
如果我们配置jenkins构建仓库的话，代码也会自动拉取到 `/var/lib/jenkins/workspace` 目录下。
### 修改jenkins用户为root
通过vim修改 `vim /etc/sysconfig/jenkins` 文件。找到`JENKINS_USER` 修改如下。
```shell
$JENKINS_USER="root"
```
### 修改端口
jenkins默认端口是8080，但是有可能已经有其他应用占用了，修改的话在上面的配置文件中找到 `$JENKINS_PORT`修改为对应端口就行了。
### 启动jenkins
```shell
java -jar /usr/lib/jenkins/jenkins.war 
```
启动之后访问地址就行了。比如 http://127.0.0.1:8080.
当然阿里云服务器得话需要配置安全组，开放8080端口tcp入站。
访问之后有个初始化过程，我们推荐方式安装jenkins插件。一路按照说明配置完就行了。
完事后就可以就可以进入工作台了。

# idea
### 创建项目
直接新建item
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/723b865f5c564ec3b17cc8f2ec14d3ac~tplv-k3u1fbpfcp-zoom-1.image)
选择 freestyle project 
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c463f459ce34ad4941867b15fc3ed3f~tplv-k3u1fbpfcp-zoom-1.image)
完成之后就能在all中看见新建的项目的.
### 配置项目
接下来进入项目中,点击配置,配置相关的git仓库,访问凭据以及构建脚本
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69f6341c24e94fe6861f9f1e23f7baa4~tplv-k3u1fbpfcp-zoom-1.image)
源码管理选择git,凭据选择ssh 或者用户名密码的方式.
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa98ddffd93b48b7899095edbba76fd1~tplv-k3u1fbpfcp-zoom-1.image)
 这样就配置好了git仓库. 在构建的时候,就会自动拉取代码到 `/var/lib/jenkins/`文件夹下面了.
另外要根据自己的项目类型便些合适的构建脚本,并且需要在服务器中安装好对应的环境.
我要尝试 构建flutter项目,先以flutter为例,web以及安卓或者ios同理.
# 四，构建flutter项目
flutter的构建作为开发者我们知道需要安装flutter环境,服务器中构建也是一样。在**诸多尝试失败之后，最后我成功使用snap安装上了flutter，并间接安装上了fvm工具**。
fvm是flutter的多版本管理工具,我们尝试安装fvm.我之前在macos上安装很轻松写意的,于是打算在centos上也装上.
### 尝试brew安装fvm失败
注意：本人尝试在centos上用brew安装fvm各种失败，有坑请绕过
1. root用户安装brew失败
2. 个人用户安装brew下载失败，访问github的域名有问题
3. 使用镜像安装失败，提示 `Another active Homebrew update process is already in progress`
4. 以及其他各种失败~ 很多,各种网络问题系统问题
最后放弃使用brew安装,准备使用dart安装pub的方式安装fvm
### 尝试在centos安装dart失败.
最后是找到[官方的centos 安装文档](https://github.com/dart-lang/sdk/wiki/Building-Dart-on-CentOS,-Red-Hat,-Fedora-and-Amazon-Linux-AMI),是利用编译dart的方式完成.最后还是失败了.中间踩了不少坑.
1. 获取devtool的时候.googlesource.com 访问不到,最后使用`gitee`克隆仓库解决
2. 使用fetch  获取dart源码的时候也是各种网络报错
最后经过各种挣扎决定放弃直接安装dart.准备采取安装flutter的方式用里面的自带dart曲线救国.
### 在centos安装flutter
[官方告诉](https://flutter.dev/docs/get-started/install/linux)我们使用snap安装flutter.
```shell
sudo snap install flutter --classic
```
我们当然没有安装snap,所以查找[snap安装教程](https://snapcraft.io/docs/installing-snap-on-centos)安装,最后终于成功了!!!
```shell
$ sudo dnf install epel-release
$ sudo dnf upgrade
$ sudo yum install snapd
$ sudo systemctl enable --now snapd.socket
$ sudo ln -s /var/lib/snapd/snap /snap
```
最后成功安装上了flutter~~~
### 安装fvm
snap安装完flutter之后直接有了环境变量.
接下来`flutter doctor` 安装环境,` dart pub global activate fvm` 安装fvm一气呵成。
最后配置环境变量
```shell
export PATH="$PATH":"$HOME/.pub-cache/bin"
```
完美安装，接下来使用fvm配置安全局flutter环境
使用` fvm global 2.5.0-5.2.pre` 配置全局flutter环境。

> 因为之前安装了flutter 所以，需要删除之前的flutter环境变量，将`~/fvm/default/bin`设置为新的flutter环境变量
### android sdk 配置
选择要安装的文件夹（我叫它`BASE_PATH`）并使用以下命令安装带有flutter的SDK：
**安装 SDK**
```shell
cd $BASE_DIR 
mkdir android-sdk 
cd android-sdk 
wget https://dl.google.com/android/repository/commandlinetools-linux-6200805_latest.zip 
unzip commandlinetools-linux-6200805_latest.zip 
./tools/bin/sdkmanager --sdk_root=$(pwd) "build-tools;28.0.3" "emulator" "platform-tools" "platforms;android-28" "tools"
```
### 打包apk
在flutter默认项目中，打包很简单，只需要`flutter build apk`就行了
这个命令写在`jenkins`的构建脚本中。
> 本来一切都挺好，应该顺利的，但是编译过程中，服务器直接卡死，几十分钟无法打包完毕。
> 服务器的性能根本吃不住安卓apk的编译。只能退而求其次，编译web了。
# 五，使用nginx 服务器放置打包结果
1. 按照[官方教程](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)安装nginx。配置好源后运行 `yum install nginx`
2. 运行nginx `sudo systemctl start nginx`
3. 设置nginx开启自启动 `sudo systemctl enable nginx`
4. 访问域名就能看到`nginx`页面了
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ebfc64532a64a16a57d0c9fb4592413~tplv-k3u1fbpfcp-zoom-1.image)
***以下是Nginx的默认路径：***
- Nginx配置路径：/etc/nginx/
- PID目录：/var/run/nginx.pid
- 错误日志：/var/log/nginx/error.log
- 访问日志：/var/log/nginx/access.log
- 默认站点目录：/usr/share/nginx/html
配置nginx: `vim /etc/nginx/nginx.conf`
```
# 虚拟主机server块
server {
    # 端口
    listen   8080;
    # 匹配请求中的host值
    server_name  localhost;

    
    # 文件路径
    location /files/ {
        # 查找目录
        root /data;
    }
```
配置完成后执行 `nginx -t` 看是否有错误，如果看到的是下面这种就是成功：
```shell
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
然后执行 `nginx -s reload` 更新 Nginx 配置文件
在编译完成之后，只需要将静态网页移交到网页服务器文件夹中就行了。
一切都很完美。
# 六，企业微信，钉钉或者飞书通知结果
搜索各自的webhook方式就行了。
我使用的是飞书通知，直接在飞书添加群机器人，拿到webhook链接做post请求就行了。
# 七，最终配置的构建脚本和成果
我最后的构建shell脚本如下：
```shell
#设置好环境变量
export PATH=$PATH:/root/fvm/default/bin
export ANDROID_SDK_ROOT=/usr/local/android-sdk-linux
#编译出产物
flutter build web --web-renderer canvaskit --no-sound-null-safety --release 
#删除旧的网站
rm -rf /usr/share/nginx/html/web
#部署网站
mv build/web/ /usr/share/nginx/html/
curl -X POST -H "Content-Type: application/json" -d '{"msg_type":"text","content":{"text":"网页部署成功，地址是 http://www.guuguo.top/ "}}' https://open.feishu.cn/open-apis/bot/v2/hook/***** 
```
执行编译的话能够自动编译flutter web，自动放置到静态资源服务器下面以供外网访问，还能提交到代码到coding中，把网页通过coding托管到腾讯云中对外提供。

成果如下，访问http://47.92.80.168/web/ 就能看到部署好的网页了。
第一个是阿里云nginx服务器，第二个是腾讯云免费托管的。

# 完结
> 最终受限于服务器的性能没能实现android apk打包，十分可惜，但是也算是学会了一些新东西，以及完成了web的打包发布工作。
>
> 以后不怕费电的话，可以拿自己的电脑当一台自动化服务器

完结撒花❀❀❀❀❀❀❀

