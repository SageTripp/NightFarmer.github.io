---
title: 创建一个简单的android桌面APP
date: 2016-07-29 15:51:56
tags: [Android]
category: Android
---

**本篇通过创建一个简单的桌面应用程序来记录桌面应用的开发流程**

创建一个桌面App并不难，只是日常的工作重心皆在普通的应用程序开发上，对桌面app并没有过多的关注，
下面通过新建一个简单的启动器来熟悉并记录桌面app的开发。
效果：
![简单的桌面](http://nightfarmer.github.io/public/static/image/home_app.gif)
<!-- more -->

### 注册Activity
为了让我们的应用程序入口能够作为桌面应用让android系统识别，我们需要在AndroidManifest.xml文件中为我们的Activity增加category
```xml
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />

                <category android:name="android.intent.category.HOME"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
        </activity>
```
如果我们的桌面应用程序在被打开的时候并不需要欢迎页或者其他描述性的界面的话，我们只需要将LauncherActivity增加HOME和DEFAULT两个声明，
这样这个Activity就可用被识别为可作为桌面启动器的界面，当点击home按键时就会被操作系统弹出供用户选择。

### 获取本机所有应用程序信息
我们在注册Activity之后，作为桌面现在是没有任何内容的，打开之后里面只有一个HelloWorld。
当前市面上各种主流的桌面应用中，基本上全都是自定义的控件，来灵活满足用户的日常手势操作，因为本篇只是作为桌面应用程序的开发流程向导，重心并不在自定义控件上，所以这里使用日常的表格控件如(GridView,RecyclerView)做为我们展示应用程序入口图标的控件。
我们在布局文件中新增RecyclerView控件，并为这个控件编写Adapter和ItemLayout。(具体过程略)

然后我们需要获取本机的所有应用程序信息来作为展示的数据源
```java
	//获取应用程序数据
        Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
        mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);
        List<ResolveInfo> mApps = getPackageManager().queryIntentActivities(mainIntent, 0);
	//整理图标和应用名称
        ArrayList<ItemBean> itemBeans = new ArrayList<>();
        for (ResolveInfo appInfo :
                mApps) {
            ItemBean itemBean = new ItemBean();
            itemBean.name = appInfo.loadLabel(getPackageManager()).toString();
            itemBean.icon = appInfo.loadIcon(getPackageManager());
            itemBean.appInfo = appInfo;
            itemBeans.add(itemBean);
        }
        recyclerView.setAdapter(new OnePageAdapter(itemBeans));
```

在应用程序初始化时我们需要将应用的图标和名称全部处理成可用状态，避免在RecyclerView滚动期间多次获取造成卡顿。
ItemBean是我们定义的一个用于存储图标、名称以及应用信息的一个简单类
这样在Adapter中通过给ItemView的各个控件赋值来达到展示应用信息的效果
```java
        holder.app_icon.setImageDrawable(itemBean.icon);
        holder.app_name.setText(itemBean.name);
        holder.itemBean = itemBean;
```

### 启动本机指定应用程序
上面我们已经可用看到所有的本机应用，我们需要给这些item添加点击监听，用于启动应用
点击监听的添加这里不再赘述，下面是启动应用的方法
```java
        String pkg = itemBean.appInfo.activityInfo.packageName;
        String cls = itemBean.appInfo.activityInfo.name;
        ComponentName component = new ComponentName(pkg, cls);
        Intent intent = new Intent();
        intent.setComponent(component);
        itemView.getContext().startActivity(intent);
```

### 使用系统壁纸
至此，作为桌面启动器的基本功能已经具备了，接下来需要给这个桌面App美化一下。
主流的桌面应用都是可用灵活设置壁纸的，而作为一个Activity设定背景图标并不难，通过background或者ImageView都可以实现。
但是android为我们提供了一个样式，通过这个样式我们可以使用系统默认的壁纸，并且可以通过其他应用修改系统壁纸的方法来实现修改我们的桌面app壁纸的效果
```xml
    <style name="AppTheme" parent="android:Theme.Wallpaper.NoTitleBar">
        <!-- Customize your theme here. -->
        .....
    </style>
```
设置这个样式之后不但让Activity有了系统壁纸，而且隐藏了TitleBar，看起来像是一个桌面了。

### 修改系统壁纸
这里附上修改系统壁纸的方法，用于桌面应用的setting或者其他地方来灵活使用
```java
        //生成一个设置壁纸的请求
        final Intent pickWallpaper = new Intent(Intent.ACTION_SET_WALLPAPER);
        Intent chooser = Intent.createChooser(pickWallpaper,"选择壁纸");
        //发送设置壁纸的请求
        startActivity(chooser);
```

### 设置状态栏透明(状态栏沉浸)
到此为止我们的桌面应用和主流的桌面仍然有一些差别，比如没有下面的固定的图标位，不能翻页，不能拖拽图标等等，但是这些都可用通过灵活布局和扩展来进行实现，就不再一一编写了。
这里额外加上一个api19之后的一个特性，让状态栏变成透明的，让整个桌面更加简洁
新建values-v19资源文件夹，文件夹下新增styles.xml，在样式文件中新增
```xml
    <item name="android:windowTranslucentStatus">true</item>
```
并在Activity的根布局中加入android:fitsSystemWindows="true"
这样在api19之后的android设备上就会出现透明状态栏的特性了。


后记：本篇只是记录了开发桌面应用和普通应用程序的主要区别以及核心流程，至于各种特效以及酷炫的特性就属于开发普通应用程序的范畴了，桌面应用程序作为日常和用户交互最多的app，不仅需要友好的UI更需要流畅操作体验，需要精雕细琢才能开发出一个良好的桌面应用程序。



