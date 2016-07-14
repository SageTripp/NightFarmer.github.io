---
title: AndroidStudio 自动生成 SerialVersionUID
date: 2016-05-30 20:26:17
tags: [Android, AS, 干货]
category: Android
---

上篇提到需要本地序列化的类实现Serializable接口时需要声明serialVersionUID变量，本篇简单说一下如何在 Android Studio 中自动帮助我们生成(不重复的) SerialVersionUID。<!-- more -->
打开 Settings，切换到 Editor->Inspections->Java->Serialization issues，找到 Serializationzable class without ‘serialVersionUID’，将其勾选即可，如图：

![开启SerialVersionUID提示](http://nightfarmer.github.io/public/static/image/开启SerialVersionUID提示.png)

其实这个意思就是在代码检查时看是否有 serialVersionUID 这个字段，没有就提示你。

![SerialVersionUID提示](http://nightfarmer.github.io/public/static/image/SerialVersionUID提示.png)

之后在实现了 Serializable 接口的类名上打开 Android Studio 的智能提示(智能检查)，就会多出了 Add ‘serialVersionUID’ field，如图：

![显示生成SerialVersionUID菜单](http://nightfarmer.github.io/public/static/image/显示生成SerialVersionUID菜单.png)

生成之后的代码


![自动生成SerialVersionUID](http://nightfarmer.github.io/public/static/image/自动生成SerialVersionUID.png)



