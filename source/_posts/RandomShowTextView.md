---
title: 实现一个随机字符显现/隐藏的TextView
date: 2016-03-13 12:46:44
tags: [Android, 动画，自定义控件]
category: Android
---

偶然看到一个非常不错的特效，尝试着实现了并记录下开发过程。
效果图：
![随机字符显现/隐藏](http://nightfarmer.github.io/public/static/image/RandomShowTextView.gif)

<!-- more -->

实现文本的显示一般使用Canvas绘图或者使用TextView，TextView为我们提供了很多对文本的封装包括文字大小，颜色，内外边框，对其方式，字体等等，所以在TextView满足需求的时候尽量不使用Canvas来进行文本显示，这里我们选择使用TextView作为我们特效控件的基础。

google为我们提供了SpannableString类对TextView进行同一个控件显示不同文本效果提供了支持，所以我们可用通过这个类来控制不同位置的字符的alpha来达到我们的目的，对于SpannableString的具体用法这里不做过多阐述，参考[这里](http://blog.sina.com.cn/s/blog_5da93c8f0100ul3z.html)

首先我们定义一个叫做RandomShowTextView的自定义类继承自TextView
```java
public class RandomShowTextView extends TextView {
    private int mDuration = 2500;
    ValueAnimator animator;
    ValueAnimator.AnimatorUpdateListener listener = new ValueAnimator.AnimatorUpdateListener() {
        @Override
        public void onAnimationUpdate(ValueAnimator valueAnimator) {
            Float percent = (Float)valueAnimator.getAnimatedValue();
            resetSpannableString(percent);        
        }
    };

    public RandomShowTextView(Context context) {
        super(context);
        init();
    }

    public RandomShowTextView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init(){
        this.mIsVisible = false;
        animator = ValueAnimator.ofFloat(0.0f, 2.0f);
        animator.addUpdateListener(listener);
        animator.setDuration(mDuration);
    }
}

```

我们初始化时定了一个0-2的ValueAnimator并设置了2500毫秒的动画时长，同时设置了动画监听，在回调方法中获取到动画期间实时的值。
在动画回调期间，我们调用了一个叫做resetSpannableString的自定义方法，在这个方法中我们需要根据不同0-2期间不同值的回调对TextView中的文本进行更新

```java
    private boolean mIsReset = false;
    private void resetSpannableString(double percent){
        SpannableString mSpannableString = new SpannableString(this.mTextString);
        int color = getCurrentTextColor();
        for(int i=0; i < mSpannableString.length(); i++){
            mSpannableString.setSpan(
                    new ForegroundColorSpan(Color.argb(clamp(mAlphas[i] + percent), Color.red(color), Color.green(color), Color.blue(color))), i, i + 1, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
        mIsReset = true;
        setText(mSpannableString);
        invalidate();
    }

    private int clamp(double f){
        return (int)(255*Math.min(Math.max(f, 0), 1));
    }

    private void resetAlphas(int length){
        mAlphas = new double[length];
        for(int i=0; i < mAlphas.length; i++){
            mAlphas[i] = Math.random()-1;
        }
    }

    private void resetIfNeeded(){
        if (!mIsReset){
            mTextString = getText().toString();
            resetAlphas(mTextString.length());
            resetSpannableString(0);
            mIsReset = false;
        }
    }

    @Override
    public void setText(CharSequence text, TextView.BufferType type) {
        super.setText(text, type);
        resetIfNeeded();
    }
```
我们复写了setText方法，在此方法中设置动画之前需要准备的初始化数据，包括文本的内容，文本字符的随机出现顺序以及动画进度。
在resetAlphas中我们用一个数组来存储每个字符的对应的初始alpha值，这个值我们是取了-1到0的随机数，而这个随机数和我们的动画进度值0到2相加，结果的值区间为1到2，而动画期间的值区间为-1到2，alpha我们取大于0小于1的值，造成初始值越小的字符越晚出现，值越大的字符就越早达到alpha1。

这样TextView设置字符随机透明度的方法已经完成了，接下来需要加上一些供外部调用的开关方法：
```java
    private boolean mIsVisible;
    public void toggle(){
        if (mIsVisible) {
            hide();
        } else {
            show();
        }
    }

    public void show(){
        mIsVisible = true;
        animator.start();
    }

    public void hide(){
        mIsVisible = false;
        animator.start();
    }
```

同时在ValueAnimator的监听回调中调整一下对显示和隐藏的处理
```java
            resetSpannableString(mIsVisible ? percent : 2.0f - percent);
```

这样，我们的控件就定义完成了，我们可以想普通的TextView那样使用它，包括设置字体颜色大小等，而当我们调用toggle/show/hide这写方法时，就会出现我们扩展的随机显隐动画了。

