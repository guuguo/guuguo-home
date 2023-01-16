---
title: 【小试牛刀】输出sdk的时候靠删改代码？试试gradle配置吧
---

> 靠着打代码，我终于走上了人生巅峰，看着支付宝里似乎永远花不完的余额，我不禁开始迷茫。后面的路在何方？
>
> 一阵恍惚，我从工位上醒了过来，夏日炎炎，办公室的风扇吹不走空气中的燥热。
>
> 妈呀果然是梦~~

# 一.背景

最近团队做一个提供 ocr 识别能力的 sdk 项目。身为一个普通码农，当然是领导说干啥就干啥，于是和算法对接吭哧吭哧打完了 身份证识别，手机号识别等卡证识别功能。开心，交付给了客户！！

不久领导又说搞个指尖识别的sdk提供给用户做指尖搜题用，和原来的项目放一块，搞个统一的能力sdk。但是输出sdk的时候要求剔除掉指尖识别之外的无关代码。为了赶进度挥洒之间，我又写完了，在打包的时候，直接拉了个新的输出分支剔除旧代码。就这样输出了。

后面又要一个新的sdk能力，图像质检 sdk 。

麻烦的问题就跳出来了：

*   **输出sdk的时候需要每次拉新分支剔除，并每次更新的时候合并**

*   **每个sdk能力都需要独立，又可以合并**

怎么办呢~~

# 二.思考

> 这个问题说是问题也不是问题，无非是有点繁琐，维护好几个不同能力的独立分支就完事了。

> 但是我懒呀~~

聪明的我就想到了两个方法。

1.  每个能力分出独立的模块，编译的时候打对应能力模块就好啦
2.  使用gradle 配置编译不同的源文件，根据配置来打出包含不同源码的sdk

**最后当然是选择了使用gradle** **sourceSets** **配置的方式来编译不同的sdk源码啦。**为什么不选分模块呢？因为对外提供的是aar包，而不是maven仓库地址。所以存在多个能力多个aar包的情况，尽量提供少的aar包吧，方便用户方便自己。

# 三.开干

1.  ### 先剥离不同sdk的源代码，放在不同的文件夹下面。通用代码放在默认的java文件夹中。

这是一个细活，避免互相依赖，尽量依赖倒置，解开耦合，让每个sdk能力都能独立编译，并且能一起编译。

![image](https://guuguohome.oss-cn-hangzhou.aliyuncs.com/temp_37e94fdf47f74a6ea17b5926f8924f33~tplv-k3u1fbpfcp-zoom-1.image.jpg)

2.  ### 在 gradle.properties 中新增sdkType配置

```
## 手机号，身份证银行卡等
#sdkType=ocr

## 图像质检
#sdkType=graphcheck

## 指尖检测
#sdkType=fingerdetect

## 依赖全部代码
sdkType=all

```

3.  ### 在 build.gradle 配置源文件路径

```
android {
    //...其他配置代码
    sourceSets {
        main {
            //默认文件夹
            def javaPaths = ["src/main/java"]
            def assetsPaths = ["src/main/assets"]

            def javaGraphCheck = "src/main/javaGraphcheck"
            def javaFingerDetect = "src/main/javaFingerdetect"
            def javaOcr = "src/main/javaOcr"

            def assetsGraphCheck = "src/main/assetsLens"
            def assetsOcr = "src/main/assetsOcr"
            def assetsFinger = "src/main/assetsFinger"

            if (sdkType == "graphcheck") {
                javaPaths.add(javaGraphCheck)

                assetsPaths.add(assetsGraphCheck)
            } else if (sdkType == "fingerdetect") {
                javaPaths.add(javaFingerDetect)

                assetsPaths.add(assetsFinger)
            } else if (sdkType == "ocr") {
                javaPaths.add(javaOcr)

                assetsPaths.add(assetsOcr)
            } else {
                javaPaths.add(javaGraphCheck)
                javaPaths.add(javaFingerDetect)
                javaPaths.add(javaOcr)

                assetsPaths.add(assetsGraphCheck)
                assetsPaths.add(assetsOcr)
                assetsPaths.add(assetsFinger)
            }
            java.srcDirs = javaPaths
            assets.srcDirs = assetsPaths
        }
    }
}

```

4.  ### 修改编译产物的名称(为了直观区分打出来的是啥)

```
android {
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            android.libraryVariants.all { variant ->
                variant.outputs.all {
                    outputFileName = "sail_ocr-${version}-${sdkType}.aar"
                }
            }
        }
    }
}

```

5.  ### 其他配置

dependencies 依赖 和 setting.gradle 中也可以根据 sdkType 来判断。如下就是我项目中的demo模块树。

```

if (sdkType == "graphcheck") {
    include ':lensdemo'
    include ':module_graphcheck'
    include ':module_ocr'
} else if (sdkType == "fingerdetect") {
    include ':demo_finger_detect'
    include ':module_scan_finger'
    include ':module_ocr'
    include ':module_base'
    include ':module_ui'

    include ':lib_base'
    include ':lib_utils'
} else if (sdkType == "ocr") {
    include ':demo_ocr'
    include ':module_ocr'
    include ':module_base'
    include ':module_ui'

    include ':lib_base'
    include ':lib_utils'
} else {
    include ':app'
    include ':demo_ocr'
    include ':lensdemo'
    include ':module_graphcheck'
    include ':module_scan_finger'
    include ':module_profile'

    include ':module_ocr'
    include ':module_base'
    include ':module_ui'
    include ':profile_photo_frame'

    include ':lib_base'
    include ':lib_utils'

}

```

**现在只需要在gradle. properties中修改sdkType的值，就可以直接编译出对应的sdk包**
# 四.结尾


这样的工作并不难，做完之后却有小小的成就感。

抽出那么半天一天时间专门用来剥离代码，解除耦合，调试错误。琐碎得工作却让时间流逝得飞快。

> 很多事情都不难，只是迟迟没有人去做罢了。