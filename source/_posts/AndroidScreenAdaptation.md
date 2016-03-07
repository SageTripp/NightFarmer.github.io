---
title: Android屏幕分率适配
date: 2016-03-07 13:41:26
tags: [Android, 适配]
category: Android
---

**关于系统资源的配置的目录**
Android系统支持多配置资源文件，我们可以追加新的资源目录到Android项目中。<!-- more -->
命名规范：资源名字-限制符

|资源目录|适配屏幕|
|---------|:--------:|
|layout-small|小屏幕|
|layout|默认中等屏幕|
|layout-large|大屏幕|
|layout-xlarge|特大屏幕|
|layout-xlarge-land|特大屏幕 横屏|
|layout-land|横屏|
|layout-port|竖屏|
|drawable|默认中等密度|
|drawable-hdpi|高密度~240dpi|
|drawable-mdpi|中等密度~160dpi|
|drawable-xhdpi|更高密度~320dpi|
|drawable-tvdpi|介于mdpi~hdpi 约213dpi 主要应用在电视|
|drawable-nodpi|所有密度资源，无论什么密度屏幕都会适配|

> **注**：如果没有指定横屏或竖屏，则上面的布局和位图都适配横竖屏。如果要指定横屏，例如：`drawable-land-hdpi`，竖屏`drawable-port-hdpi`，还有关键是`drawable-xlarge`和`layout-xlarge`，对api level都要求在`9`之上，等于说，你用`android2.2`系统的平板或者手机根本不匹配`layout-xlarge`，因为api level是`8`。drawable-tvadpi这个api等级需要`13`以上。

&#160; &#160; 上面的layout-large这个目录其实是个范围。当系统根据当前屏幕的大小和密度，决定程序应该匹配那个目录。你也可以单独定制某些不符合谷歌标准的山寨版`layout-1024x600`（中间的符号是英文下的x字母），其中1024和600的单位是`dp`。你可以根据你设备的分辨率和密度，来判断你的设备需要定义那个文件。
&#160; &#160;但是，官方推荐使用尺寸来表示资源`layout-large`,不推荐使用分辨率`layout-1024x600`。

建议大家多看文档，官方说明：

xlarge screens are at least 960dp x 720dp

large screens are at least 640dp x 480dp

normal screens are at least 470dp x 320dp

small screens are at least 426dp x 320dp

上面是定义广义大小布局资源适配的一个范围，大家可以根据自己的设备知道系统会匹配那个文件的布局。

如果手上有个山寨华为的卖的比较火的mediapad，大家知道分辨率1280*800 密度尺寸7寸

通过勾股定了和分辨率可以得出其密度为215.69。然后根据dp=px/(dpi/160)，可以得出个范围593.471。所以这个设备系统会匹配layout-large这个资源布局文件。