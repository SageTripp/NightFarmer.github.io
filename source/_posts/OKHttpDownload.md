---
title: OKHttp3-扩展带有进度监听的下载回调
date: 2016-01-12 10:46:44
tags: [Android, okhttp]
category: Android
---

**本篇用于记录使用OKhttp3网络框架如何下载文件**
okhttp是一个非常优秀的网络请求框架，本着不重复早轮子的原则，很多项目中都采用了okhttp来处理网络请求，
然而okhttp本身并没有提供关于文件下载的相关封装，这里我们会记录如何基于okhttp网络请求框架来下载文件。<!-- more -->

在不使用参数，只有url的情况下，okhttp最基本的调用方法如下：

```java

    OkHttpClient okclient = new OkHttpClient()
    Request request = Request.Builder()
                .url(url)
                .build();

    okclient.newCall(request).enqueue(callback);

```

如何在这种情况下下载文件呢？
这里我们定一个叫做FileDownLoadHandler的抽象类实现Callback，来实现自定义的回调。
> 因为项目里采用了kotlin和java混合编译，而这个回调是采用kotlin编写的，这里就贴上kotlin的代码了

```java
abstract class FileDownLoadHandler(var file: File) : Callback {

    companion object {
        var error: HashMap<Int, String> = HashMap()

        init {
            error.put(1, "连接失败，请重试")
            error.put(2, "下载失败，手机空间不足")
            error.put(3, "下载失败，请重试")
        }
    }

    var hanlder = Handler()

    override fun onFailure(call: Call?, e: IOException?) {
        hanlder.post { callFailure(1) }
    }

    override fun onResponse(call: Call?, response: Response) {
        if (!response.isSuccessful) throw IOException("Unexpected code " + response);

        val byteStream = response.body().byteStream()
        val length = response.body().contentLength();
        val os: FileOutputStream
        try {
            os = FileOutputStream(file);
        } catch(e: Exception) {
            hanlder.post { callFailure(2) }
            return
        }
        var bytesRead = -1;
        val buffer = ByteArray(1024)
        var process = 0L;

        try {
            do {
                bytesRead = byteStream.read(buffer)
                if (bytesRead == -1) break;
                process += bytesRead
                os.write(buffer, 0, bytesRead);
                hanlder.post {
                    onProcess(process, length)
                }
            } while (true)
        } catch(e: Exception) {
            hanlder.post {
                callFailure(3)
            }
        }


        hanlder.post {
            onSuccess(file)
            onFinish()
        }
    }

    fun callFailure(errorCode: Int, msg: String? = null) {
        onFailure(msg ?: error[errorCode] ?: "", errorCode)
        onFinish()
    }


    abstract fun onProcess(current: Long, total: Long)

    open fun onFinish() {
    }

    abstract fun onSuccess(file: File)

    abstract fun onFailure(msg: String, errorCode: Int);
}
```
> kotlin的语法会和java有一些区别，鉴于这里并没有用到kotlin的高级特性，就不做一一解释了
上面代码中我们实现了okhttp的CallBack接口，并定义了带有一个文件参数的构造方法，参数中的文件类用于下载并在本地存储文件。
在CallBack接口的不同方法中我们实现了对文件的下载以及失败和成功的回调，并定义了成功和失败回调的抽象方法。

接下来我们看一下这个回调类在项目中的用法：
```java
        HttpCall.INSTANCE.downLoad(url, new FileDownLoadHandler(file) {
            @Override
            public void onFailure(@NotNull String msg, int errorCode) {
                mSVProgressHUD.dismissImmediately();
                alertView2.show();
		//开始下载，显示下载dialog
            }

            @Override
            public void onSuccess(@NonNull File file) {
                mSVProgressHUD.dismiss();
                //下载成功
		//.....
            }

            @Override
            public void onFinish() {

            }

            @Override
            public void onProcess(long current, long total) {
                if (mSVProgressHUD.getProgressBar().getMax() != mSVProgressHUD.getProgressBar().getProgress()) {
                    int progress = (int) (current * 100.0 / total);
                    mSVProgressHUD.getProgressBar().setProgress(progress);
                    mSVProgressHUD.setText("更新进度 " + progress + "%");
                } else {
                    mSVProgressHUD.dismiss();
                }
		//下载进度更新回调，更新dialog进度
            }
        });
```

是不是很方便呢？


