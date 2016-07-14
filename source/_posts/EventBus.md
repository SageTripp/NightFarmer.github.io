---
title: 通过EventBus实现android各组件间数据通讯
date: 2016-04-13 11:48:15
tags: [Android, EventBus]
category: Android
---

EventBus就是publish/subscribe消息总线，主要功能是替代Intent,Handler,BroadCast在Fragment，Activity，Service，线程之间传递消息。
它的三要素：
Event：事件。可以是任何的对象。
Subscriber：事件订阅者，接收特定的事件。方法以onEvent**开头，一共有四个方法onEvent，onEventMainThread，onEventBackgroundThread，onEventAsync。它们之间的区别在于在不同的线程。等会会有一一举例。
Publisher:事件发布者，用于通知Subscriber有事件发生，可以在任何的地方发布事件。使用也是简单，只要调用post(Object)方法就可以了。
<!-- more -->
下面说一下这个库到底如何使用
首先定义一个事件对象

```java
public class MyEvent {
    public String data;
    public int otherData;
}
```

在Activity中实现onEvent**方法

```java
public class Activity1 extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //...
        EventBus.getDefault().register(this);
        //...
        final MyEvent event = new MyEvent();
        event.data = "消息携带的数据";
        event.otherData = 123;
        Log.i("Activity1", "发送事件："+Thread.currentThread().getId());
        EventBus.getDefault().post(event);

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException ignore) {
                }
                Log.i("Activity1", "发送事件："+Thread.currentThread().getId());
                EventBus.getDefault().post(event);
            }
        }).start();
    }


    /**
     * 接收事件和post事件在同一个线程中执行
     * @param event 消息体
     */
//    @Subscribe
    @Subscribe(threadMode = ThreadMode.POSTING)
    public void onEvent(MyEvent event){
        Log.i("Activity1", "onEvent:"+Thread.currentThread().getId());
    }

    /**
     * 接收事件永远在UI线程中执行
     * @param event 消息体
     */
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onEventMainThread(MyEvent event){
        Log.i("Activity1", "onEventMainThread:"+Thread.currentThread().getId());
    }

    /**
     * 如果在子线程中post事件，则会在相同子线程接收
     * 如果在UI线程中post事件，则会启动一个子线程接收事件
     * @param event 消息体
     */
    @Subscribe(threadMode = ThreadMode.BACKGROUND)
    public void onEventBackgroundThread(MyEvent event){
        Log.i("Activity1", "onEventBackgroundThread:"+Thread.currentThread().getId());
    }

    /**
     * 新建一个子线程来接收事件
     * @param event 消息体
     */
    @Subscribe(threadMode = ThreadMode.ASYNC)
    public void onEventAsync(MyEvent event){
        Log.i("Activity1", "onEventAsync:"+Thread.currentThread().getId());
    }
}


```

执行结果
```
07-14 11:13:02.011 3137-3137/com.nightfarmer.sample I/Activity1: 发送事件：1
07-14 11:13:02.011 3137-3137/com.nightfarmer.sample I/Activity1: onEvent:1
07-14 11:13:02.011 3137-3197/com.nightfarmer.sample I/Activity1: onEventAsync:19405
07-14 11:13:02.011 3137-3137/com.nightfarmer.sample I/Activity1: onEventMainThread:1
07-14 11:13:02.021 3137-3200/com.nightfarmer.sample I/Activity1: onEventBackgroundThread:19406
07-14 11:13:05.021 3137-3201/com.nightfarmer.sample I/Activity1: 发送事件：19407
07-14 11:13:05.021 3137-3201/com.nightfarmer.sample I/Activity1: onEvent:19407
07-14 11:13:05.021 3137-3201/com.nightfarmer.sample I/Activity1: onEventBackgroundThread:19407
07-14 11:13:05.021 3137-3200/com.nightfarmer.sample I/Activity1: onEventAsync:19406
07-14 11:13:05.021 3137-3137/com.nightfarmer.sample I/Activity1: onEventMainThread:1
```

根据注解threadMode属性的不同，我们接受到事件的线程也有相应的不同
所以根据上面的结果可以很好的理解各个onEvent的区别：

POSTING:事件在哪个线程发布出来的，就会在这个线程中运行，也就是说发布事件和接收事件线程在同一个线程。

MAIN:事件无论是从哪个线程发布出来的，都会在UI线程中执行。

BACKGROUND:事件是在UI线程中发布出来的，那么就会在子线程中运行，如果事件本来就是子线程中发布出来的，那么就直接在该子线程中执行。

ASYNC：使无论事件在哪个线程发布，都会创建新的子线程。
