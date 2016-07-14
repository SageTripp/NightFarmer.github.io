---
title: AndroidStudio-Kotlin插件安装配置及工程配置调整
date: 2016-03-25 16:12:29
tags: [Android, Kotlin]
category: Android
---

Kotlin 是一个基于 JVM 的新的编程语言，由 JetBrains 开发。
Kotlin可以编译成Java字节码，也可以编译成JavaScript，方便在没有JVM的设备上运行。
JetBrains，作为目前广受欢迎的Java IDE IntelliJ 的提供商，在 Apache 许可下已经开源其Kotlin 编程语言。

总体说, 对于大部分普通程序员, 可算比较完美了(综合考量语言自身\平台及库\IDE等工具\背后支持公司). 目前主要风格还是偏OO, 如果可以再偏FP一点会更好. 像是一个Scala与C#的合体, 比Scala简单得多; 比C#更干净, 可谓是android平台的swift。 
<!-- more -->
有关kotlin的优势这里久不一一列举了，下面记录下在使用AndroidStudio开发android应用程序或者使用intellij idea的情况如何使用kotlin

打开 Settings，切换到 Plugins->Browse repositories，搜索kotlin，找到Kotlin的插件并点击右侧的install按钮即可进行下载安装，如图：

![kotlin插件安装](http://nightfarmer.github.io/public/static/image/kotlin插件安装.png)

安装完成后重启即可，这个插件包含了kotlin语言原生部分以及android开发的扩展，所以AndroidStudio和intellij idea的安装过程是一样的。
至此，kotlin插件已经安装完成，如果只是使用kotlin开发普通java程序，现在就可用正常使用了，如果想使用kotlin进行android开发，那么还需要进行一些额外的配置。

首先需要在Project的build.gradle文件的dependencies中加入
```
        classpath "org.jetbrains.kotlin:kotlin-android-extensions:$kotlin_version"

```
然后在需要使用kotlin的module的build.gradle文件中加入
```
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'


buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
```
在这个文件的dependencies中加入
```
    compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    compile 'org.jetbrains.anko:anko-sdk15:0.8.3'
    // sdk19, sdk21, sdk23 are also available
    compile 'org.jetbrains.anko:anko-support-v4:0.8.3'
    // In case you need support-v4 bindings
    compile 'org.jetbrains.anko:anko-appcompat-v7:0.8.3'
```

重新构建一次项目就可用使用kotlin进行android开发了，上面的配置接入了anko扩展，这个是kotlin官方对针对kotlin的语法特性对android开发进行了额外的补充，用起来非常的方便，在之后的文章中会再介绍这个扩展。

