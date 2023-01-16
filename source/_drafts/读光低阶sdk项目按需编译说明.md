# 读光低阶sdk项目按需编译说明

# 使用方式，如何按需使用
### 一，找到android sdk 项目的`build.gradle` 文件
路径为`project/android_files/mind-duguangSDK/app/build.gradle`
### 二，修改文件中的sdkType
现有三个类型  
1. ocr     ocr能力sdk(包含身份证识别，银行卡识别，手机号面单识别等)
2. Lens   图像识别能力sdk(图像边缘检测和图像矫正能力等)
3. hand   指尖检测能力
以字符串形式配置，用英文逗号隔开，如包含ocr和lens能力，则配置为
```groovy
def sdkType="ocr,lens"
```
![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=YWZiNzI4NGYxMDI1YzgwNWVjYjNkOGRjNzAzMDI2N2ZfcUdybkRhVThhVnFPMXQ3dkhDWXBCdEtHbVl6YVR1T1dfVG9rZW46Ym94Y25TOHd6ak9NOGkya21vZG5Hamo5aGtiXzE2MjkxODU0Nzk6MTYyOTE4OTA3OV9WNA)
可以看到上图有3个配置项，第一个是version中配置版本号的更改，第二个是通过licenceCheck 配置sdk是否启用鉴权。第三个就是配置sdkType，选择要变异的sdk源码。

### 三，编译出sdk
通过`assembleRelease`命令就可以打包出aar包了
在`mind-duguangSDK`目录下面，直接运行以下命令

```shell
./gradlew :app:aR
```
如果环境没问题，可以看到最后是 **BUILD SUCCESSFUL**
接下来就可以在build目录下面找到aar文件了，路径如图所示
![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=MjhkYjBiYWJkMWUwODE1ODliZDNhNTk4NGQ0Nzg2MmNfZVp2azV2Q3RFV3NTYVBpRHJPZWpzU25JNzVjQTBVaHVfVG9rZW46Ym94Y244Z0NnU25JcFZCd1lBUVd4Z2RJM0N2XzE2MjkxODU0Nzk6MTYyOTE4OTA3OV9WNA)

# sdk 设计思路
按需编译主要分为如下三步
### 1. 修改djjnni 文件
按照类型区分不同djjnni定义文件，区分出三个能力到三个不同的djjnni文件。(如果有改动在对应文件内修改，如果有能力新增则添加类型)
![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=MDM4ZjMxNjBkYjEwMDFmYTExZDQzYjcwZjkyZTVjMTlfQ3lnaldXbHNZOGM0eFpQaUtDbmtla3VwcGZzalFyRGpfVG9rZW46Ym94Y250TkpqU1NXblJ2d2dtS2EzS2JVR0VjXzE2MjkxODU0Nzk6MTYyOTE4OTA3OV9WNA)
1. ### 通过`/api_generator/tools/generate.sh`脚本生成对应代码
执行该脚本生成`ocr,lens,hand,common`四个不同的文件夹

![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDZiMjY0ODdmMWVhYzhkZmIxZTFkOWExOGUzODkxZWZfVUdTN0xOZndXYnRUR0p0UTBOSlhDam1ZQmo2OXl3a0JfVG9rZW46Ym94Y25VZU5rUDBZN3lUV0E1QllBa09aMHpkXzE2MjkxODU0Nzk6MTYyOTE4OTA3OV9WNA)
前面三个是不同的能力文件夹，用于选择性配置，common则是都要依赖的基础文件夹。
(该脚本在android preBuild 的时候会自动去调用，理论上不用管。如果能力新增，添加方法调用)
### 2. build_android.sh 链接对应文件夹
该脚本位于`/android_files/build_android.sh`
调用的时候 -b 指定编译类型，不指定，默认编译ocr,lens,hand 三种全部类型，指定的话编译指定的类型，用逗号隔开。  -b 代表不做cmake 编译操作。 调用示例如下(该脚本在android preBuild 的时候会自动去调用，理论上不用管。如果能力新增，修改该脚本)
```
./build_android -t ocr,hand,len -b
```
关键代码在`ln_source_files` 方法中，如下
```shell
# ln auto_gen java to android project
source_java="${base_path}/api_generator/auto_gen/java/"
dest_java="${libray_path}/app/src/main/java/com/alibaba/android/mind/duguang"

# 创建源代码目标文件夹(目标文件夹作为编译期的native源码文件夹)
mkdir $dest_header
mkdir $dest_java
mkdir $dest_src
mkdir $dest_jni

# 链接文件夹到目标文件夹中
ln -sf ${source_src}/common/* $dest_src
ln -sf ${source_jni}/common/* $dest_jni
ln -sf ${source_java}/common/* $dest_java
ln -sf ${source_header}/common/* $dest_header   #yaobin

# 遍历所有sdk类型，将旗下的所有代码平铺链接到目标文件夹中
for addr in $buildTypes; do
    ln -sf ${source_src}/$addr/* $dest_src
    ln -sf ${source_jni}/$addr/* $dest_jni
    ln -sf ${source_java}/$addr/* $dest_java
    ln -sf ${source_header}/$addr/* $dest_header    #yaobin
done
```
### 3. build.gradle 脚本
脚本在`android_files/mind-duguangSDK/build.gradle`
通过以下几个变量定义的地方修改编译配置
```groovy
group = getEnvValue("GROUP_ID", "com.alibaba.mind.duguang")
version = getEnvValue("MUPP_VERSION_NAME", "1.6.5-SNAPSHOT")
///def sdkType = "ocr,lens,hand"
def licenceCheck = true
///编译sdk类型
def sdkType = "ocr"
```
sdkType 主要用于调用build_android.sh 脚本时候的参数。主要在预编译的时候 做了以下两步
- 通过djjnni生成 android 和ios通用桥接代码。
- 调用buildAndroid脚本重新建立源代码链接关系。
```groovy
task cleanFiles(type: Delete) {
    delete file("./build")
    delete file("./.externalNativeBuild")
}
///调用generateDjinni 脚本。根据djjnni 定义文件，生成不同的jni等代码
task generateDjinni() {
    dependsOn cleanFiles
    doLast {
        println("generateDjinni")
        exec {
            workingDir "../../../api_generator/tools"
            commandLine 'sh', 'generate.sh'
        }
    }
}
///调用buildAndroid 脚本。平铺link 不同的源码文件夹
task buildAndroid() {
    dependsOn generateDjinni
    doLast {
        exec {
            println("buildAndroid")
            workingDir "../../"
            commandLine 'sh', 'build_android.sh'
            args('-t',sdkType,'-b')
        }
    }
}

preBuild.dependsOn(buildAndroid)
```