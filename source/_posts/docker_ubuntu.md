---
title: 【小试牛刀】MacOS在Docker中运行ubuntu环境
---
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcd8c87205c44aa1aee971f2c27c1f9f~tplv-k3u1fbpfcp-zoom-crop-mark:1304:1304:1304:734.awebp)

# 前言

安卓做累了，看点杂学。
最近尝试使用`lvgl` 进行linux gui 开发，中间安装编译环境需要使用linux环境。然而我只有mac系统。所以尝试使用docker来提供linux环境。这样就不需要安装虚拟机啦~

# 一，Docker是什么？
- IT 软件中所说的 “Docker” ，是指容器化技术，用于支持创建和使用 [Linux® 容器](https://www.redhat.com/zh/topics/containers)。
- [开源 Docker 社区](https://forums.docker.com/)致力于改进这类技术，并免费提供给所有用户，使之获益。
- [Docker Inc.](https://www.docker.com/) 公司凭借 Docker 社区产品起家，它主要负责提升社区版本的安全性，并将技术进步与广大技术社区分享。此外，它还专门对这些技术产品进行完善和安全固化，以服务于企业客户。
# 二，安装Docker
Macos 安装Docker
```shell
$ brew install --cask --appdir=/Applications docker
==> Creating Caskroom at /usr/local/Caskroom
==> We'll set permissions properly so we won't need sudo in the futurePassword:          # 输入 macOS 密码
==> Satisfying dependencies==> Downloading https://download.docker.com/mac/stable/21090/Docker.dmg
######################################################################## 100.0%
==> Verifying checksum for Cask docker==> Installing Cask docker==> Moving App 'Docker.app' to '/Applications/Docker.app'.
&#x1f37a;  docker was successfully installed!
```
安装完成之后就能在应用程序当中找到`Docker.app`了
![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=MjVjMWYzYWMzMTk0M2U3NzZhNmZhY2YzNGMyYzIxNTdfU1c1ZGhja2l3MDdtbTRPSGdpRUg1UzNJcFhLZXhzR0lfVG9rZW46Ym94Y25lMnNIQ3VmbVcyY1o5NzdKVkdpYVlXXzE2MjkwOTY3MTE6MTYyOTEwMDMxMV9WNA)

# 三，安装ubuntu镜像
```shell
 docker run ubuntu:18.04 
```
直接使用该命令，docker会判断本地镜像的存在，并自动下载安装。

## 1. 使用docker界面运行linux环境

运行完毕之后，`docker app` 的images目录下面就会出现ubuntu选项，直接点击run 运行`ubuntu`环境。

![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=MDZmOGMzNTA3NTVkYjczNGQxZjllNzcxMjk2NjI0ZjBfQ21NOGlOV2l2T3o4bG5sbEhMRTZydVh0MTV4WUdLYmJfVG9rZW46Ym94Y24yd2NmNmhuTFJvRDRtenU0NnIyRlFmXzE2MjkxMDAwMjE6MTYyOTEwMzYyMV9WNA)

将会弹出该环境下的命令行窗口：

![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=NjJlNjYwOTdiMWU3YmRkMjRkYmYzM2EzMGM3MjY2ZmNfVTd2aEVNdWpsVnlnN1ZMbXF4eHB1ZWNiT0hCUk5NelVfVG9rZW46Ym94Y25XeXZoa1o4d0hIaHNCRnIycHYxRmVkXzE2MjkxMDAwMjE6MTYyOTEwMzYyMV9WNA)

## 2. 使用docker命令运行linux环境

直接使用docker exec 命令

```shell
docker exec -it <container> bash
#docker exec -it 02e2b0ef7c0a bash
```

运行`cat /etc/issue` ，可以看到该linux系统是 `Ubuntu 18.04.5 LTS`

```shell
# cat /etc/issue
Ubuntu 18.04.5 LTS \n \l
```

很明显，这个linux环境就是ubuntu环境了。这个是时候成功拥有了一个ubuntu容器环境。在里面想干啥就干啥~

## 额外的

我们在编辑一些文本文件和环境变量的时候都需要使用到文本编辑器，因为系统没有内置的vim，所以通过`apt-get`安装一个:

```shell
apt-get update 
apt-get install vim
```

# 四，传输文件到容器中
当然我们有时候在主系统下载文件，需要传输到ubuntu环境中，就需要用到docker 的cp命令。使用方式如下。
先使用`docker ps`拿到容器的psid
```shell
 % docker ps               
CONTAINER ID  IMAGE      COMMAND         CREATED       STATUS       PORTS   NAMES
02e2b0ef7c0a  ubuntu:18.04  "bash"          About an hour ago  Up About an hour       laughing_bhabha
6c06fef8e55b  docker:latest  "docker-entrypoint.s…"  About an hour ago  Up About an hour       objective_chebyshev
```
然后使用cp命令拷贝文件到容器中，命令是`docker cp 本地文件路径 ID全称:容器路径` 。
样例如下：
```shell
docker cp /Users/guodeqing/Downloads/ARM_Compiler_5.05_update_1_cracked.tar.gz 02e2b0ef7c0a:/home
```
# 结尾
成功使用docker跑起了ubuntu容器环境，接下来就是一些linux的基础操作了(各种google命令粘贴)。我是用它来设置lvgl的编译环境。
看起来确实可以利用cocker来替代linux 虚拟机了。