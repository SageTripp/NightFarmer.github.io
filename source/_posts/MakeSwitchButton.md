---
title: 自定义控件-定义一个开关按钮
date: 2015-04-07 23:41:26
tags: [Android, 自定义控件]
category: Android
---

**本篇文章用于记录用于开放绘制BitMap相关的自定义控件**
Android应用程序中有一类自定义控件是由canvas绘制bitmap来构建视图，并通触摸事件或其方法来控制绘制视图的状态。<!-- more -->
本篇通过记录一个开关按钮的定义来熟悉此类自定义控件的开发流程。
先看一下效果图：

图

废话不多说，首先我们定义一个叫做SwitchView的类继承自View来作为我们的自定义控件
代码如下:
```java
public class SwitchView extends View {

    public SwitchView(Context context) {
        super(context);
    }

    public SwitchView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public SwitchView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
}
```
这这时候我们没有做任何自定义的处理，我们定义一个方法用来设置这个控件的图片资源，
我们定义的这个控件包括两个状态，打开和关闭，所以我们需要有两张背景图片来标识控件的开关状态，
也需要一个按钮图片来让用户来进行操作以及对控件的状态进行展示。
```java
    private Bitmap backGroundOn;
    private Bitmap backGroundOff;
    private Bitmap switchButton;

    private float viewWidth;
    private float buttonWidth;

    public void setImageResource(int switchButton, int backGroundOn, int backGroundOff) {
        this.switchButton = BitmapFactory.decodeResource(getResources(), switchButton);
        this.backGroundOn = BitmapFactory.decodeResource(getResources(), backGroundOn);
        this.backGroundOff = BitmapFactory.decodeResource(getResources(), backGroundOff);

        viewWidth = this.backGroundOn.getWidth();
        buttonWidth = this.switchButton.getWidth();
    }

```
我们定义了一个叫做setImageResource的方法，在这个方法里我们通过图片资源参数生成了相应的bitmap用于控件的绘制
同时获取了背景图片的宽度来作为控件的宽度，button的宽度用于之后状态的判断和触摸事件的反馈。
接下来先绘制一个状态下的控件视图，复写View的onDraw方法和onMeasure方法分别用于绘制和测量控件
```java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Bitmap backGround = backGroundOff;
        canvas.drawBitmap(backGround, new Matrix(), new Paint());
        canvas.drawBitmap(switchButton, 0, 0, new Paint());
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(backGroundOn.getWidth(), backGroundOn.getHeight());
    }

```
上面这段代码并不多，在onDraw方法中先绘制背景图，然后将button按钮绘制在背景图上方，初始位置为0,0坐标
在onMeasure方法中指定控件的宽高为背景图的宽和高，这时候我们就可以在xml布局文件中调用我们的自定义控件，
指定资源文件后就可以看到初始状态下的UI：

图

但是此时这个控件只有一个关闭状态，下面我们为这个控件增加打开状态并新增状态标识以及切换状态的方法
```java
    private float buttonX = 0;
    private boolean checked;

    public boolean isChecked() {
        return checked;
    }

    public void setChecked(boolean checked) {
        if (this.checked == checked) return;
        this.checked = checked;
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Bitmap backGround;
        if (!pressed) {
            if (checked) {
                buttonX = viewWidth - buttonWidth;
            } else {
                buttonX = 0;
            }
        }
        if (buttonX<viewWidth / 2){
            backGround = backGroundOff;
        }else {
            backGround = backGroundOn;
        }

        canvas.drawBitmap(backGround, new Matrix(), new Paint());
        canvas.drawBitmap(switchButton, buttonX, 0, new Paint());

    }
```
新增两个变量，checked用于设定和获取开关按钮的不同状态，buttonX则用于记录不同状态下按钮的绘制起始位置，
之所以将buttonX作为控件的成员变量而不是方法内的变量，是因为我们还要在下面的触摸事件中设置这个值。
这时候我们已经可以通过checked的get/set方法来切换控件的开关状态了，我们还要为这个控件增加触摸事件来响应用户的点击以及拖拉操作。
复写View的onTouchEvent事件来监听用户的手势
```java
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                pressed = true;
                break;

            case MotionEvent.ACTION_MOVE:
                pressed = true;
                buttonX = event.getX() - buttonWidth / 2;
                if (buttonX < 0) buttonX = 0;
                if (buttonX > viewWidth - buttonWidth) buttonX = viewWidth - buttonWidth;
                break;

            case MotionEvent.ACTION_UP:
                pressed = false;
                checked = event.getX() > viewWidth / 2;
                if (listener != null) {
                    listener.onChange(this, checked);
                }
                break;

            default:
                pressed = false;
                break;
        }
        invalidate();
        return true;
    }
```
触摸事件的监听的核心逻辑在move和up中，move中判断触摸点位于View的x坐标，并根据x坐标来设定button的绘制起始x坐标
在up中，根据触摸离开时的坐标点来盘点控件应该切换到哪个状态，这里我们根据触摸的x坐标位于控件中心点的左右来作为判断的依据。
> 上面代码中的press变量来存贮和标识控件是否处于触摸操作中，这个变量在绘制时用到，如果处于触摸中则使用实时的button绘制坐标，反之则根据开关状态将button绘制到控件两侧

上面还在up中调用了监听回调，这是对控件开关状态切换的监听，补全状态监听的代码，如下：

```java
    private OnCheckedStateChangedListener listener;

    public void setOnCheckedStateChangedListener(OnCheckedStateChangedListener listener) {
        this.listener = listener;
    }

    public void setChecked(boolean checked) {
        if (this.checked == checked) return;
        this.checked = checked;
        invalidate();
        if (listener != null) {
            listener.onChange(this, checked);
        }
    }

    public interface OnCheckedStateChangedListener {
        void onChange(SwitchView switchView, boolean checked);
    }

```

这时候我们的自定义开关控件已经大致定义完成了，当然也可以为这个控件新增一些自定义属性，如在xml布局文件中指定图片资源等。





