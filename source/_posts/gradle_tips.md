---
title: 【Android 开发】 Gradle 使用小技巧
---
> Gradle 是我们安卓者常常打交道的东西，在做一些编译期间的自动化操作时，gradle可谓非常有用武之地。了解下有哪些使用方式，也便于我们开发时候做一些编译优化。
**这里小结了四个平时比较常用的gradle小技巧**


# 一、Gradle 全局变量定义使用

1. `gradle.ext` 全局变量(可以在 `setting.gradle` 中使用)
使用场景很多，我们可以用来外部gradle脚本中左右一些常用属性的配置，在子项目的gradle 中执行方法一键配置等。
```
def hello = { it ->
    println "hello"
}
gradle.ext.hello = hello
//调用
gradle.ext.hello()
```
1. 项目级的全局拓展参数可以在rootProject中设置ext。如下，在最外层的 build.gradle 中设置
```
ext.kotlinVersion="1.0"
///使用
rootProject.ext.kotlinVersion
```


# 二、Gradle task 任务按顺序执行

1. 使用dependsOn (执行该命令前，会先执行dependsOn依赖的命令)
```
//执行 a4 会带上前面几个任务先执行
task a1() {
    doFirst {
        println "do a1";
    }
}
///参数中指定 dependsOn
task a2(dependsOn: a1) {
    doLast {
        println "do a2"
        exec {
            commandLine "ls"
        }
    }
}
///闭包中指定 dependsOn
task a3() {
    dependsOn 'a2'
    doLast {
        println "do a3";
    }
}
///外部指定 dependsOn
task a4() {
    doLast {
        println "do a4";
    }
}
a4.dependsOn(a3)
```
1. 使用 finalizedBy
```
///a1 跑完后紧跟着就会跑 a2
task a1() {
    doFirst {
        println "a1"
    }
}
task a2() {
    doFirst {
        println "a2"
    }
}
a1.finalizedBy(a2)
```


# 三、Plugin 命令查看依赖树，找出重复依赖和版本号不匹配问题源头

```
./gradlew app:dependencies
```
输出的日志很直观，树形列表，有问题直接搜索对应依赖库查看版本号
```
\--- androidx.databinding:databinding-compiler:4.1.3
     +--- androidx.databinding:databinding-compiler-common:4.1.3
     |    +--- androidx.databinding:databinding-common:4.1.3
     |    +--- com.android.databinding:baseLibrary:4.1.3
     |    +--- org.antlr:antlr4:4.5.3
     |    +--- commons-io:commons-io:2.4
     |    +--- com.googlecode.juniversalchardet:juniversalchardet:1.0.3
     |    +--- com.google.guava:guava:28.1-jre
     |    |    +--- com.google.guava:failureaccess:1.0.1
     |    |    +--- com.google.guava:listenablefuture:9999.0-empty-to-avoid-conflict-with-guava
     |    |    +--- com.google.code.findbugs:jsr305:3.0.2
     |    |    +--- org.checkerframework:checker-qual:2.8.1
     |    |    +--- com.google.errorprone:error_prone_annotations:2.3.2
     |    |    +--- com.google.j2objc:j2objc-annotations:1.3
     |    |    \--- org.codehaus.mojo:animal-sniffer-annotations:1.18
     |    +--- com.squareup:javapoet:1.10.0
     |    +--- org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.3.72
     |    |    +--- org.jetbrains.kotlin:kotlin-stdlib:1.3.72
     |    |    |    +--- org.jetbrains.kotlin:kotlin-stdlib-common:1.3.72
     |    |    |    \--- org.jetbrains:annotations:13.0
     |    |    \--- org.jetbrains.kotlin:kotlin-stdlib-jdk7:1.3.72
     |    |         \--- org.jetbrains.kotlin:kotlin-stdlib:1.3.72 (*)
     |    +--- com.google.code.gson:gson:2.8.5
     |    +--- org.glassfish.jaxb:jaxb-runtime:2.3.1
     |    |    +--- javax.xml.bind:jaxb-api:2.3.1
     |    |    |    \--- javax.activation:javax.activation-api:1.2.0
     |    |    +--- org.glassfish.jaxb:txw2:2.3.1
     |    |    +--- com.sun.istack:istack-commons-runtime:3.0.7
     |    |    +--- org.jvnet.staxex:stax-ex:1.8
     |    |    +--- com.sun.xml.fastinfoset:FastInfoset:1.2.15
     |    |    \--- javax.activation:javax.activation-api:1.2.0
     |    +--- com.android.tools:annotations:27.1.3
     |    \--- com.android.tools.build.jetifier:jetifier-core:1.0.0-beta09
     |         +--- com.google.code.gson:gson:2.8.0 -> 2.8.5
     |         \--- org.jetbrains.kotlin:kotlin-stdlib:1.3.60 -> 1.3.72 (*)
     +--- androidx.databinding:databinding-common:4.1.3
     +--- org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.3.72 (*)
     +--- com.google.auto:auto-common:0.10
     |    \--- com.google.guava:guava:23.5-jre -> 28.1-jre (*)
     +--- commons-io:commons-io:2.4
     +--- commons-codec:commons-codec:1.10
     +--- org.antlr:antlr4:4.5.3
     \--- com.googlecode.juniversalchardet:juniversalchardet:1.0.3
```


# 四、依赖gradle 文件

```
apply from:"../libconfig.gradle"
```
抽取通用的grdle脚本，放在一个独立的 gradle文件，在各个子 module 中 apply 该文件就行。对于多个子模块的的项目可以减少一些模板代码的使用，并且可以在修改的时候减少工作量。
`libconfig.gradle` 如下
```
apply plugin: 'com.android.library'
android {
    compileSdkVersion rootProject.ext.android.compileSdkVersion
    buildToolsVersion rootProject.ext.android.buildToolsVersion
    defaultConfig {
        minSdkVersion rootProject.ext.android.minSdkVersion
        targetSdkVersion rootProject.ext.android.targetSdkVersion
        versionCode rootProject.ext.android.versionCode
        versionName rootProject.ext.android.versionName
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    buildFeatures {
        dataBinding = true
    }
    compileOptions {
        sourceCompatibility = 1.8
        targetCompatibility = 1.8
    }
}
```
子项目`build.gradle`就可以减少对应的配置了
```
apply from:"../libConfig.gradle"
dependencies {
    api project(':lib_utils')
    api rootProject.ext.dependencies["immersionbar"]
    annotationProcessor rootProject.ext.dependencies["arouter-compiler"]
    implementation rootProject.ext.dependencies["arouter-api"]
    api 'androidx.appcompat:appcompat:1.3.0'
    api 'com.google.android.material:material:1.3.0'
    api 'androidx.constraintlayout:constraintlayout:2.0.4'
}
```