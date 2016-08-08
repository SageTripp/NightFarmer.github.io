---
title: 通过自定义控件绘制实现雷达/水波纹的动画扩散效果
date: 2016-08-08 11:48:17
tags: [Android, 自定义控件, 动画]
category: Android
---

前些天发现支付宝的“咻一咻”挺有意思的，点击之后会出现波纹状的涟漪向四周扩散，波纹的深浅会随着扩散的距离逐渐减弱，最后消失不见，而且这这些波纹的扩散距离以及产生频率是固定不变的，然后循环执行下去。
效果图：
![水波扩散效果](http://nightfarmer.github.io/public/static/image/CircleWave.gif)
<!-- more -->

考虑到绘制的UI是随着时间的变化而改动的，我们首先会想到使用动画，除了动画我们还要考虑的是如果把这个波纹扩散的过程抽象出一个循环。
这里有两个思路
**一、** 把单个波纹的扩散作为动画，从波纹的产生到波纹的结束视为执行一个动画，而多个波纹的扩散相当于多个动画依次执行，而我们抽象出来的循环则是写一个无限循环，并在每个循环之间delay一个时间值，每次循环时则会创建一个Animator实例并立即start。如此这个连续的波纹就可以理解为不断的执行Animator，有多少个涟漪就有多少个处于执行状态的Animator。
**二、** 定义一个Value动画，定义出每个涟漪之间的间隔，并根据Value循环的逻辑对界面上的每个涟漪的位置及透明进行计算，在ondraw方法中进行统一绘制。

这两种思路在理论上都是行的，但是从开销层面来考虑的话方案一并不是一个非常好的方法，虽然它的逻辑看起来非常清晰，也是按自然的执行方式来进行执行动画的(产生-扩散-消失-循环)，但是当界面上存在很多个涟漪的情况下会造成非常多的动画在同时执行，这会造成非常大的开销。

所以这里我们采用方案二，使用ValueAnimator来执行一个0到1的动画，并循环执行。
在这种情况下的循环相当于是一个涟漪的循环，那么我们需要根据这个涟漪来计算出其他涟漪应该存在的位置。

```java
/**
 * Created by zhangfan on 16-8-8.
 */
public class CircleWaveView extends View {
    private float maxWaveRadius = 300;//扩散半径
    private long waveTime = 2000;//一个涟漪扩散的时间
    private int waveRate = 4;//涟漪的个数
    //...get/set方法略
    private Paint paint;
    private float[] waveList;//涟漪队列
    private int centerX;//控件中心x坐标
    private int centerY;//控件中心y坐标

    //...构造方法略
    private void init(Context context) {
        paint = new Paint();
        waveList = new float[waveRate];
        ValueAnimator valueAnimator = ValueAnimator.ofFloat(0, 1);
        valueAnimator.setInterpolator(new LinearInterpolator());
        valueAnimator.setDuration(waveTime);
        valueAnimator.setRepeatCount(ValueAnimator.INFINITE);
        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                Float value = (Float) animation.getAnimatedValue();
                for (int i = 0; i < waveList.length; i++) {
                    float v = value - i * 1.0f / waveRate;
                    if (v < 0 && waveList[i] > 0) {
                        v += 1;
                    }
                    waveList[i] = v;
                }
                invalidate();
            }
        });
        valueAnimator.start();
    }

```
我们定义了一个waveList数组来存储所有涟漪的扩散进度，在第一个涟漪的动画监听中，计算出后边几个涟漪的进度，当第一个涟漪循环到初始位置时，其他涟漪的进度是负值，我们来给这些涟漪的进度加1来弥补这个差值，而当整个动画开始时，第一个涟漪之后的涟漪其实是并没有产生的，也就是说他们的值为0，这时候我们判断如果这个涟漪上一次的进度为0则不加1。

如此，在绘制的时候一次对这些涟漪进行绘制即可
```java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        for (Float waveRadius : waveList) {
            paint.setColor(Color.argb((int) (255 * (1 - waveRadius)), 60, 114, 236));//根据进度产生一个argb颜色
            canvas.drawCircle(centerX, centerY, waveRadius * maxWaveRadius, paint);//根据进度计算扩散半径
        }
    }
```
>> 这里的绘制颜色只做示例，如有需求可自行扩展。

这里的centerX和centerY是在onMeasure方法中获得，保证波纹的起始位置位于控件中心点。
```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        centerX = getMeasuredWidth() / 2;
        centerY = getMeasuredHeight()/2;
    }
```

完工。
