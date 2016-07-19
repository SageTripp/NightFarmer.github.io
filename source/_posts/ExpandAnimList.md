---
title: 设计一个带有动画的可用展开/折叠的列表控件
date: 2016-03-19 10:46:44
tags: [Android, 列表动画，RecyclerView]
category: Android
---

今天看到某个订餐app中有一个折叠菜品详情的特效，看起来非常不错，反编译源码之后发现是使用ListView做的。
但是最近对RecyclerView这个控件情有独钟，于是就尝试着使用RecyclerView进行了实现。
记录一下：

![展开/折叠效果](http://nightfarmer.github.io/public/static/image/ExpandAnimList.gif)

<!-- more -->

对于普通列表样式的RecyclerView我们再熟悉不过了，Activity+Adapter。
如果需要实现刚才效果，我们需要一个特殊的Item的布局文件：
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_margin="10dp"
    android:orientation="vertical">

    <TextView
        android:clickable="true"
        android:id="@+id/tv_title"
        android:background="#7acaec"
        android:padding="10dp"
        android:text="标题"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <RelativeLayout
        android:id="@+id/layout_content"
        android:background="#ffdc8a"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">
        <LinearLayout
            android:layout_centerInParent="true"
            android:layout_width="wrap_content"
            android:layout_height="100dp">
            <ImageView
                android:layout_gravity="center"
                android:src="@mipmap/ic_launcher"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content" />
            <TextView
                android:layout_gravity="center"
                android:text="我是一堆内容，我是一堆内容，我是一堆内容，我是一堆内容"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content" />
        </LinearLayout>
    </RelativeLayout>
</LinearLayout>
```
这个布局分为上下两块儿，title部分是折叠时候显示的内容，这里我们只定义了一个TextView，如果有特殊需求可以设定一个容器。
下面的layout_content是展开时候所显示的布局，如果不考虑动画的情况下，我们只需要设置layout_content是否可见即可。

接下来我们就要考虑动画的实现了。
比较直接的做法，在title的点击监听里start一个Animator，看上去好像并没有什么不对，然而如果真的这样写了的话，我们会得到这样的结果：
```java
        ViewCompat.animate(view).scaleY(0).start();
```
![异常的折叠效果](http://nightfarmer.github.io/public/static/image/ExpandAnimList_bug.gif)
动画虽然执行了，但是整个列表并没有重新布局！

这时候我们需要用另外一种方法来实现动画了，ValueAnimator。
使用ValueAnimator设置数值动画执行的起止数值，并在数变更监听中更新itemview的尺寸并通知父容器重新布局。

```java
        int origHeight = view.getHeight();
        ValueAnimator animator = ValueAnimator.ofInt(origHeight, 0);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {

            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator) {
                int value = (Integer) valueAnimator.getAnimatedValue();
                ViewGroup.LayoutParams layoutParams = view.getLayoutParams();
                layoutParams.height = value;
                view.setLayoutParams(layoutParams);
            }
        });
        animator.addListener(new AnimatorListenerAdapter() {

            @Override
            public void onAnimationEnd(Animator animator) {
                view.setVisibility(View.GONE);
            }
        });
        animator.start();
```

这样就能在动画执行期间实时的更新整个列表的布局了
接下来我们需要让itemview支持点击折叠操作，假设我们的所有item初始情况下都是折叠的状态，定义一个Map<Integer, View> expandedViewMap，用来保存已经展开的item对应的position和layout_content，对于一个position，如果在expandedViewMap中存在key，则这个item是折叠状态，反之这是展开状态。
修改itemview的onclick回调方法：
```java
    boolean expanded = expandedViewMap.containsKey(position);
    if (expanded) {
        expandedViewMap.remove(position);
        //TODO 执行收缩动画
    } else {
        expandedViewMap.put(position, layout_content);
        //TODO 执行展开动画
    }
```
收缩动画就是上面那段，然后我们还需要实现一个展开的动画
```java
        view.setVisibility(View.VISIBLE);
        final int widthSpec = View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED);
        final int heightSpec = View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED);
        view.measure(widthSpec, heightSpec);

        ValueAnimator animator = ValueAnimator.ofInt(0, view.getMeasuredHeight());
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {

            @Override
            public void onAnimationUpdate(ValueAnimator valueAnimator) {
                int value = (Integer) valueAnimator.getAnimatedValue();
                ViewGroup.LayoutParams layoutParams = view.getLayoutParams();
                layoutParams.height = value;
                view.setLayoutParams(layoutParams);
            }
        });
        animator.start();
```
在这短代码里我们首先指定layout_content不指定宽高的情况再次测量尺寸，并拿到测量后的高度，其实就是layout_content在wrap_content情况下的高度。创建一个从0到这个高度的动画，并在回调中更新布局。
这个时候item以及可用响应我们的点击事件，并执行相应的展开和折叠动画了，但是我们还有一个地方需要做处理，在RecyclerView中的ItemView会随着列表的滚动对其进行回收和复现，我们需要在ItemView被回收利用的时候对它的状态进行初始化，也就是在onBindViewHolder中设置layout_content的高度及Visible：
```java
    @Override
    public void onBindViewHolder(MyHolder holder, int position) {
        //....
        boolean expanded = expandedViewMap.containsKey(position);
        if (expanded) {
            //设置layout_content为展开状态
        } else {
            //设置layout_content为折叠状态
        }
    }
```
在注释代码的地方对layout_content的状态进行重置，这样就可用在滚动list的时候仍然保持正确的展开状态达到开篇的那种效果了。




