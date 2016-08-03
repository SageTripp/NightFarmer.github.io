---
title: 为ViewPager创建可自由设置高亮样式的TabBar
date: 2016-07-20 10:46:44
tags: [Android, ViewPager]
category: Android
---

Android提供了ActionBar/TabHost以及V7包中提供的MaterialDesign风格的TabLayout，供ViewPager切换时进行标识和切换page使用。
这种情况下或者没有滑动效果，或者存放不下太多tab，或者UI风格比较固定，在使用过程中不能畅快得实现想要的效果。
今天定义了一个可用设定演示的可滑动的tabView供以后使用，顺便做了gradle依赖。
看下效果：

![PageViewTab示例](http://nightfarmer.github.io/public/static/image/PageViewTab1.gif) ![PageViewTab示例](http://nightfarmer.github.io/public/static/image/PageViewTab2.gif) ![PageViewTab示例](http://nightfarmer.github.io/public/static/image/PageViewTab3.gif)

使用方法：
在project的build文件中加入：
<!-- more -->

```
	allprojects {
		repositories {
			...
			maven { url "https://jitpack.io" }
		}
	}
```
在module中加入：
```
	dependencies {
	        ....
	        compile 'com.github.NightFarmer:PageViewTab:1.0.1'
	}
```




![PageViewTab示例](http://nightfarmer.github.io/public/static/image/PageViewTab.gif)
