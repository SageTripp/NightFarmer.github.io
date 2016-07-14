---
title: 源码解析自定义View的onMeasure方法
date: 2016-02-01 15:26:17
tags: [Android, 自定义控件]
category: Android
---

日常开发过程中定义一个控件的常用方法无非是继承View然后在onMeasure方法中指定控件的尺寸，或者直接继承自RelativeLayout等成熟的控件不用重写onMeasure方法。而接下来我们需要通过走读源码来了解如何通过复写onMeasure方法来让控件可以在xml文件中通过layout_width这样的属性来指定尺寸。
<!-- more -->
我们在使用系统提供的控件时，几乎都会使用 layout_width（和 layout_height，下同） 这样的属性来设置控件的宽和高，其实这就是在通过 XML 的方式告诉系统对于该 View 的控件如何去 measure。那么下面就简单自定义一个 View，咱们也给自己的 View 用 layout_width 指定个宽高试试。
自定义View

```java
public class MyView extends View {
    // 从 xml 中使用控件必须要写的构造方法 
    public MyView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
}
```

布局文件
```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <com.nightfarmer.viewmeasure.MyView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="#abcdef" />

</RelativeLayout>

```

我们先后改一下 layout_width 和 layout_height 属性，当其属性为 30dp, match_parent 的时候，发现控件显示的大小确实符合我们的预期。但是，当我们指定其为 wrap_content 的时候，却意外发现和 match_parent 效果是相同的，即铺满了整个父级容器，为什么会这样，我们需要根据源码来看一下了。
在 View 中，有个方法叫做 onMeasure，也就是本节要讨论的主题，源码如下：
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
要看懂以上代码，需要先知道什么是 MeasureSpec。简单来说，这个类可以帮我们保存控件测量的模式和测量的大小，本质上是一个32位的 int 值，其中高2位为测量的模式，低30位为测量的大小。这么说没意思，直接看源码，它是 View 的一个静态内部类：
```java
public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

        // 使用高2位来保存测量模式
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        public static final int EXACTLY     = 1 << MODE_SHIFT;
        public static final int AT_MOST     = 2 << MODE_SHIFT;

        // 根据传入的测量大小和测量模式返回 MeasureSpec 对象，其实就是一个int
        public static int makeMeasureSpec(int size, int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }

        // 从传入的 MeasureSpec 中取得测量模式，取其高2位
        public static int getMode(int measureSpec) {
            return (measureSpec & MODE_MASK);
        }

        // 从传入的 MeasureSpec 中取得测量大小，取其低30位
        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }
}
```

看源代码就很清晰 MeasureSpec 这么短小精悍的类是怎么工作的了，主要就是用到了移位和关系与逻辑运算来操作，下面来具体说说三种 MeasureSpec Mode 的区别是什么。
- EXACTLY
精确模式，当 layout_width 指定为 100dp 和 match_parent 时，即使用这种模式。
- AT_MOST
最大值模式，当 layout_width 指定为 wrap_content 时，控件大小随着控件子空间或内容的变化而变化，此时控件的尺寸只要不超过父控件允许的最大值即可。
- UNSPECIFIED
是未指定尺寸，这种情况不多，一般都是父控件是AdapterView，通过measure方法传入的模式。
另外，specSize 的单位是 px，而不是 dp，可以自己输出看一下。（我320dpi的模拟器，输出结果为 xml 中指定 dp 数值的2倍）看懂了什么是 MeasureSpec，下面可以再回顾我们刚才贴出来的 View 中 onMeasure 默认的代码了：
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

```
其实就是通过 setMeasureDimension 来设置 View 的大小的，里面传入的分别是宽和高，可以自己重写这个方法然后写个具体的值进去看看效果（其实就是无论在 xml 中怎么指定大小，都会以你自己在这里写的值为准）。那么继续看 getDefaultSize() 这个方法干了些什么：

```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        // 注意 AT_MOST 后面没有 break 啊
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
}
```

好好看看这段代码，发现 AT_MOST 和 EXACTLY 两种模式都是返回的 specSize 啊，当指定为 wrap_content 的时候就是 AT_MOST 模式，那最大值就是父容器的大小了，所以会出现我们在文章开始提到的那个问题–指定为 wrap_content 时控件会铺满父级容器。
跟着源码过了一遍原理，那么下面我们再来提一个需求–当指定为 wrap_content 的时候，将控件大小设置为 200 px，而不是任由它铺满整个父级容器，想一想这个应该怎么来实现呢？
首先，肯定是要重写 onMeasure() 方法了，在其 setMeasuredDimension() 方法中传入我们设置好的宽和高，就可以实现我们的需求了，那么子 View 的 onMeasure 代码如下:
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	setMeasuredDimension(measureWidth(widthMeasureSpec), measureHeight(heightMeasureSpec));
}
```
我们又自己写了两个方法当做 setMeasuredDimension 的参数，下面分析其中的一个即可，比如 measureWidth():
```java
private int measureWidth(int widthMeasureSpec) {
	int result = 0;
	// 获取参数中的测量模式和测量大小
    int specMode = MeasureSpec.getMode(widthMeasureSpec);
	int specSize = MeasureSpec.getSize(widthMeasureSpec);

	if (specMode == MeasureSpec.EXACTLY) {
		// 精确模式，没的说，大小该是多少就是多少
		result = specSize;
	}else{
		result = 200;
		// wrap_content 的话，根据刚才的需求，大小只能是 200 和 specSize 的最小值
		if (specMode == MeasureSpec.AT_MOST) {
			result = Math.min(result, specSize);
        }
	}

	// 注意返回值单位是 px，不是 dp
	return result;
}
```
此时，当我们在 xml 中指定控件大小为 wrap_content 的时候，大小就会是 200 px了，而不是铺满了整个父级容器，效果如图：

我是图

