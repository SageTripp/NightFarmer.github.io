---
title: 调用Android设备中已经安装的软件打开各种类型的指定文件
date: 2016-01-02 10:46:44
tags: [Android, File]
category: Android
---

最近因项目需求需要在android应用程序中下载一些附件，并打开这些附件，比如音视频视频以及图片这些。
开始还好，文件类型不是很多，但是后来需求又加上doc/xls/ppt等，后来又兼容了pdf。
这时候已经被需求改的烦不胜烦，觉得有必要针对打开本地文件做一个通用的封装了，判断File的类型，然后用指定类型的intent去通知系统。
比如这样：FileUtil.openFile(context, file)

<!-- more -->

首先我们知道通知系统打开一个指定类型的文件一般情况需要这样做：
```java
        Intent intent = new Intent();
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        //设置intent的Action属性
        intent.setAction(Intent.ACTION_VIEW);
        //设置intent的data和Type属性。
        intent.setDataAndType(fileUrl, fileType);
        //通知系统打开文件
        context.startActivity(intent);
```

这是一段比较通用的代码，根据不同的需求传入fileUrl和fileType，在实际使用中fileType会根据不同文件类型而不同。
比如
apk后缀的需要传入application/vnd.android.package-archive
pdf后缀的需求传入application/pdf
。。。。


为了保证我们的工具类能够根据不同的文件类型能够拿到正确的文件类型，这时候需要准备一个相对丰富全面的文件后缀-文件类型的映射。
而因为工作需求我收集了下面这些，并讲让他们存入了一个HashMap

```java
public class FileUtil {

    private static final Map<String, String> typeMap = new HashMap<>();

    static {
        typeMap.put(".3gp", "video/3gpp");
        typeMap.put(".apk", "application/vnd.android.package-archive");
        typeMap.put(".asf", "video/x-ms-asf");
        typeMap.put(".avi", "video/x-msvideo");
        typeMap.put(".bin", "application/octet-stream");
        typeMap.put(".bmp", "image/bmp");
        typeMap.put(".c", "text/plain");
        typeMap.put(".class", "application/octet-stream");
        typeMap.put(".conf", "text/plain");
        typeMap.put(".cpp", "text/plain");
        typeMap.put(".doc", "application/msword");
        typeMap.put(".docx", "application/vnd.openxmlformats-officedocument.wordprocessingml.document");
        typeMap.put(".xls", "application/vnd.ms-excel");
        typeMap.put(".xlsx", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        typeMap.put(".exe", "application/octet-stream");
        typeMap.put(".gif", "image/gif");
        typeMap.put(".gtar", "application/x-gtar");
        typeMap.put(".gz", "application/x-gzip");
        typeMap.put(".h", "text/plain");
        typeMap.put(".htm", "text/html");
        typeMap.put(".html", "text/html");
        typeMap.put(".jar", "application/java-archive");
        typeMap.put(".java", "text/plain");
        typeMap.put(".jpeg", "image/jpeg");
        typeMap.put(".jpg", "image/jpeg");
        typeMap.put(".js", "application/x-javascript");
        typeMap.put(".log", "text/plain");
        typeMap.put(".m3u", "audio/x-mpegurl");
        typeMap.put(".m4a", "audio/mp4a-latm");
        typeMap.put(".m4b", "audio/mp4a-latm");
        typeMap.put(".m4p", "audio/mp4a-latm");
        typeMap.put(".m4u", "video/vnd.mpegurl");
        typeMap.put(".m4v", "video/x-m4v");
        typeMap.put(".mov", "video/quicktime");
        typeMap.put(".mp2", "audio/x-mpeg");
        typeMap.put(".mp3", "audio/x-mpeg");
        typeMap.put(".mp4", "video/mp4");
        typeMap.put(".mpc", "application/vnd.mpohun.certificate");
        typeMap.put(".mpe", "video/mpeg");
        typeMap.put(".mpeg", "video/mpeg");
        typeMap.put(".mpg", "video/mpeg");
        typeMap.put(".mpg4", "video/mp4");
        typeMap.put(".mpga", "audio/mpeg");
        typeMap.put(".msg", "application/vnd.ms-outlook");
        typeMap.put(".ogg", "audio/ogg");
        typeMap.put(".pdf", "application/pdf");
        typeMap.put(".png", "image/png");
        typeMap.put(".pps", "application/vnd.ms-powerpoint");
        typeMap.put(".ppt", "application/vnd.ms-powerpoint");
        typeMap.put(".pptx", "application/vnd.openxmlformats-officedocument.presentationml.presentation");
        typeMap.put(".prop", "text/plain");
        typeMap.put(".rc", "text/plain");
        typeMap.put(".rmvb", "audio/x-pn-realaudio");
        typeMap.put(".rtf", "application/rtf");
        typeMap.put(".sh", "text/plain");
        typeMap.put(".tar", "application/x-tar");
        typeMap.put(".tgz", "application/x-compressed");
        typeMap.put(".txt", "text/plain");
        typeMap.put(".wav", "audio/x-wav");
        typeMap.put(".wma", "audio/x-ms-wma");
        typeMap.put(".wmv", "audio/x-ms-wmv");
        typeMap.put(".wps", "application/vnd.ms-works");
        typeMap.put(".xml", "text/plain");
        typeMap.put(".z", "application/x-compress");
        typeMap.put(".zip", "application/x-zip-compressed");
        typeMap.put("", "*/*");
    }
}

```
基本上覆盖比较主流的文件后缀，如果以后遇到漏掉了的会继续往里面追加。
好了，接下来我们需要写一段代码来根据文件来判断文件类型了。

```java
    private static String getFileType(File file) {
        String type = "*/*";
        String fName = file.getName();
        //获取后缀名前的分隔符"."在fName中的位置。
        int dotIndex = fName.lastIndexOf(".");
        if (dotIndex < 0) {
            return type;
        }
        /* 获取文件的后缀名*/
        String end = fName.substring(dotIndex, fName.length()).toLowerCase();
        if ("".equals(end)) return type;
        String typeFound = typeMap.get(end);
        return typeFound == null ? type : typeFound;
    }
```

然后把我们刚开始的打开文件的代码替换为可用通用执行的：
```java
    public static void openFile(Context context, File file) {
        Intent intent = new Intent();
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        //设置intent的Action属性
        intent.setAction(Intent.ACTION_VIEW);
        //获取文件file的类型
        String type = getFileType(file);
        //设置intent的data和Type属性。
        intent.setDataAndType(Uri.fromFile(file), type);
        //跳转
        context.startActivity(intent);
    }
```

到此为止就可用按照我们开始的设想来使用这个方法来打开文件啦:

```java
    FileUtil.openFile(context, file)
```
