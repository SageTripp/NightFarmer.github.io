---
title: 自定义控件-使用Canvas绘图定义一个手写板
date: 2015-08-24 15:26:17
tags: [Android, 自定义控件]
category: Android
---

最近因项目需求，需要有一个控件，可用提供用户手写签名并保持保存为图片。
听上去是一个非常酷炫的功能，然而实现起来其实并不复杂，下面会记录一下这个控件的开发过程。
<!-- more -->
先看下效果：
![签名效果图](http://nightfarmer.github.io/public/static/image/paintView.gif)

首先需要定义一个叫做PaintView的类继承自View，并定义一个Paint用来绘制笔画
```java

/**
 * Created by zhangfan on 16-6-17.
 */
public class MyView extends View {

    private Paint brush = new Paint();

    public MyView(Context context) {
        super(context);
	init();
    }

    public MyView(Context context, AttributeSet attrs) {
        super(context, attrs);
	init();
    }

    public MyView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
	init();
    }

    private void init(){
        brush.setAntiAlias(true);//抗锯齿
        brush.setDither(true);//防抖动
        brush.setColor(Color.BLUE);
        brush.setStyle(Paint.Style.STROKE);
        brush.setStrokeJoin(Paint.Join.ROUND);
        brush.setStrokeWidth(10f);
    }

}

```

这时候绘制相关的准备工作都已准备完毕，接下来就是如何绘制用户的笔迹了。
我们重写onTouchEvent来监听用户的触摸操作，定义一个Path变量来记录的用户的触摸轨迹。
并重写onDraw方法来绘制。
```java

    private Path path = new Path();

    @Override
    protected void onDraw(Canvas canvas) {
        canvas.drawPath(path, brush);
    }


    public boolean onTouchEvent(MotionEvent event) {
        float pointX = event.getX();
        float pointY = event.getY();
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                path.moveTo(pointX, pointY);
                return true;
            case MotionEvent.ACTION_MOVE:
                path.lineTo(pointX, pointY);
                break;
            default:
                return false;
        }
        // 通知重绘
        postInvalidate();
        return false;
    }
```

到此为止我们的控件已经可用了，不过我们还需要一个方法来情况画布的内容，如下：

```java
   public void reset(){
        path.reset();
        postInvalidate();
   }
```

完工。


