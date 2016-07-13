---
title: OKHttp3-多文件form表单形式上传的使用
date: 2016-01-15 10:46:44
tags: [Android, okhttp]
category: Android
---

**本篇用于记录使用OKhttp3网络框架如何提交多文件列表的form表单**
在日常使用中最常用的是okhttp为我们提供的键-值对数据提交方式，同时也提供了包含键-值/键-文件列表的数据提交方式。<!-- more -->
下面我们就看一下这种方式如何使用。

以项目中的图片提交为例：

```java

object HttpCall {
    private val MEDIA_TYPE_PNG = MediaType.parse("image/png");
    private val HEAD_PNG = Headers.of("Content-Disposition", "form-data; filename=\"img.png\"");


    private val okclient = OkHttpClient()

    fun post(context: Context, url: String, params: Map<String, String>?, files: Map<String, List<File>>?, callback: Callback): Call {
        val builder = MultipartBody.Builder()
                .setType(MultipartBody.FORM)

        val paramsF = params.orEmpty()
        val filesF = files.orEmpty()

        for ((key, value) in paramsF) {
            builder.addFormDataPart(key, value)
        }

        for ((key, value) in filesF) {
            if (value.isEmpty()) continue
            val filesBuilder = MultipartBody.Builder();
            for (oneFile in value) {
                filesBuilder.addPart(HEAD_PNG, RequestBody.create(MEDIA_TYPE_PNG, oneFile))
            }
            builder.addFormDataPart(key, null, filesBuilder.build())
        }

        val formBody = builder.build()
        val request = Request.Builder()
                .url(url)
                .post(formBody)
                .build();

        val call = okclient.newCall(request);
        if (callback is GsonHttpCallback<*>) callback.onStart()
        call.enqueue(callback);
        return call;
    }
}
```

> 上面代码依旧是kotlin的，会抽时间补上一个java版本的

