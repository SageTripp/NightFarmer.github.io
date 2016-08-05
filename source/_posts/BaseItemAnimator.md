---
title: 通过扩展ItemAnimator实现对Item的创建/移除动画进行抽象封装
date: 2016-03-13 12:46:44
tags: [Android, 列表动画, RecyclerView]
category: Android
---

[上篇](/2016/03/13/ItemAnimator/)提到通过复制DefaultItemAnimator的源码来实现自定义的ItemAnimator。
本篇用来记录如何对ItemAnimator的实现进行再次的抽象封装，以适用于更加便捷的使用。

为了能够灵活定义不同的动画，我们需要把实现animator的部分抽象出来放到子类中处理。
<!-- more -->
下面以add的动画为例
```java
    @Override
    public boolean animateAdd(final ViewHolder holder) {
        resetAnimation(holder);
        ViewCompat.setAlpha(holder.itemView, 0);
        mPendingAdditions.add(holder);
        return true;
    }

    private void animateAddImpl(final ViewHolder holder) {
        final View view = holder.itemView;
        final ViewPropertyAnimatorCompat animation = ViewCompat.animate(view);
        mAddAnimations.add(holder);
        animation.alpha(1).setDuration(getAddDuration()).
                setListener(new VpaListenerAdapter() {
                    @Override
                    public void onAnimationStart(View view) {
                        dispatchAddStarting(holder);
                    }
                    @Override
                    public void onAnimationCancel(View view) {
                        ViewCompat.setAlpha(view, 1);
                    }

                    @Override
                    public void onAnimationEnd(View view) {
                        animation.setListener(null);
                        dispatchAddFinished(holder);
                        mAddAnimations.remove(holder);
                        dispatchFinishedWhenDone();
                    }
                }).start();
    }
```

我们把调整动画的时候需要修改的地方放到抽象方法中，而ItemAnimator需要对动画的监听我们放到一个定义好的Listener中供直接调用，
那么我们代码就可以变成这样：
```java
    protected void beforeAnimateAdd(RecyclerView.ViewHolder holder){}

    @Override
    public final boolean animateAdd(final RecyclerView.ViewHolder holder) {
        resetAnimation(holder);
        beforeAnimateAdd(holder);//调用抽象方法，让子类来提供具体实现
        mPendingAdditions.add(holder);
        return true;
    }

    protected abstract void runAnimateAdd(RecyclerView.ViewHolder holder){};

    private void animateAddImpl(final RecyclerView.ViewHolder holder) {
        runAnimateAdd(holder);//调用抽象方法，让子类来提供具体实现
    }

    protected class DefaultAddVpaListener extends VpaListenerAdapter {
        RecyclerView.ViewHolder mViewHolder;
        public DefaultAddVpaListener(final RecyclerView.ViewHolder holder) {
            mViewHolder = holder;
        }

        @Override
        public void onAnimationStart(View view) {
            dispatchAddStarting(mViewHolder);
        }

        @Override
        public void onAnimationCancel(View view) {
            ViewHelper.clear(view);
        }

        @Override
        public void onAnimationEnd(View view) {
            ViewHelper.clear(view);
            dispatchAddFinished(mViewHolder);
            mAddAnimations.remove(mViewHolder);
            dispatchFinishedWhenDone();
        }
    }
```
这样我们就可用把这个类定义为BaseItemAnimator，我们实现自定义动画处理器的时候继承这个类，并只实现具体的动画即可
比如这样：
```java
/**
 * Created by zhangfan on 16-3-13.
 */
public class LeftInAnimator extends BaseItemAnimator {

    @Override
    protected void beforeAnimateAdd(RecyclerView.ViewHolder holder) {
        ViewCompat.setTranslationX(holder.itemView, -holder.itemView.getRootView().getWidth());
    }

    @Override
    protected void runAnimateAdd(RecyclerView.ViewHolder holder) {
        ViewCompat.animate(holder.itemView)
                .translationX(0)
                .setDuration(getAddDuration())
                .setInterpolator(new OvershootInterpolator(1))//会让我们的动画有一个酷炫的回弹效果
                .setListener(new DefaultAddVpaListener(holder))//一定不能少
                .start();
    }
}

```
相对于上篇的复制整个java文件来说，这样写无疑是更加的简便了，上面只是抽象了item的add动画，其他类型的动画同理可得。
上面的代码用到了一个叫ViewHelper.clear(view)的方法，因为我们抽出的动画监听中onAnimationCancel时我们并不知道当前执行的动画是何种类型，所以这时候定义了一个叫做ViewHelper.clear的方法来让view的所有属性都回归原始，贴上代码：

```java
public final class ViewHelper {

  public static void clear(View v) {
    ViewCompat.setAlpha(v, 1);
    ViewCompat.setScaleY(v, 1);
    ViewCompat.setScaleX(v, 1);
    ViewCompat.setTranslationY(v, 0);
    ViewCompat.setTranslationX(v, 0);
    ViewCompat.setRotation(v, 0);
    ViewCompat.setRotationY(v, 0);
    ViewCompat.setRotationX(v, 0);
    ViewCompat.setPivotY(v, v.getMeasuredHeight() / 2);
    ViewCompat.setPivotX(v, v.getMeasuredWidth() / 2);
    ViewCompat.animate(v).setInterpolator(null).setStartDelay(0);
  }
}
```



