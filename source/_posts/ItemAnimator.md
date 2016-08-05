---
title: 解析DefaultItemAnimator源码了解ItemAnimator动画处理器实现
date: 2016-03-13 10:46:44
tags: [Android, 列表动画, RecyclerView]
category: Android
---

之前提到过RecyclerView给我们提供了插拔式的，高度解耦的，异常灵活的数据集合展示控件。
这次会根据google提供的抽象的插拔接口来对RecyclerView数据集合item的创建/移除进行动画扩展，
并在此基础上对次扩展进行再次的抽象封装以适用于日常的使用。
依照管理，效果图：

![从左划入动画效果](http://nightfarmer.github.io/public/static/image/leftInAnimator.gif) ![从小变大动画效果](http://nightfarmer.github.io/public/static/image/inoutAnimator.gif)

<!-- more -->
RecyclerView提供了一个叫做setItemAnimator的方法来设置Item的增加/删除/修改/交换等数据集合变更时对应的动画。
这方法原形是这样的：
```java
    /**
     * Sets the {@link ItemAnimator} that will handle animations involving changes
     * to the items in this RecyclerView. By default, RecyclerView instantiates and
     * uses an instance of {@link DefaultItemAnimator}. Whether item animations are
     * enabled for the RecyclerView depends on the ItemAnimator and whether
     * the LayoutManager {@link LayoutManager#supportsPredictiveItemAnimations()
     * supports item animations}.
     *
     * @param animator The ItemAnimator being set. If null, no animations will occur
     * when changes occur to the items in this RecyclerView.
     */
    public void setItemAnimator(ItemAnimator animator)
```
这个方法的官方解释大致是这样的：ItemAnimator是一个对item变更的的动画处理器，默认的RecyclerView实例都会指定一个叫做DefaultItemAnimator的动画处理器。
主动调用这个方法，且这个方法的参数如果为null的情况下Recyclerview的item将不会有动画

一般情况下我们遇到这种形式的API都会首先想到定义一个自定义的处理器并直接实现这个接口，这样做当然没问题，然而对于这个接口来说，
当你看到需要实现的方法以及这些方法的注释的时候你肯定会被吓到，因为太多了。
很明显，这是一个比较底层的接口。对于日常的开发来说这明显工作量太大了，有没有更加简便的方式来实现呢？
其实setItemAnimator这个方法的注释中已经提到了一个类，没错它就是DefaultItemAnimator，这是RecyclerView默认的一个动画处理器，而且效果也很不错，是一个alpha递增的一个动画。
好，那我们就从这个类来着手吧，当看到他的源码的时候就会发现，这个类其实并不是直接实现ItemAnimator接口的，而是继承了一个叫SimpleItemAnimator的类，而SimpleItemAnimator这个类是一个实现ItemAnimator的一个抽象类，这时候能很明显的发现SimpleItemAnimator是对ItemAnimator接口繁多方法的一个简单的封装，来让动画处理器扩展起来更加的方便。

我们继续研究DefaultItemAnimator这个方法，仔细阅读源码之后，可用从里面抽出几个比较核心的方法
- runPendingAnimations：执行已经准备好的动画，这个方法中会根据判断来调用animateRemoveImpl/animateAddImpl/animateMoveImpl/animateChangeImpl这样的具体的动画的执行。
- animateRemove：**移除**动画执行前被调用，执行一些准备工作，并把应该执行动画的item的Holder放到全局变量中供runPendingAnimations执行动画时使用。
- animateRemoveImpl：被runPendingAnimations调用，传入itemHolder，并对itemview执行动画。
- animateAdd：**新增**动画执行前被调用，执行一些准备工作，并把应该执行动画的item的Holder放到全局变量中供runPendingAnimations执行动画时使用。
- animateAddImpl：被runPendingAnimations调用，传入itemHolder，并对itemview执行动画。
- animateMove：**移动**动画执行前被调用，执行一些准备工作，并把应该执行动画的item的Holder放到全局变量中供runPendingAnimations执行动画时使用。
- animateMoveImpl：被runPendingAnimations调用，传入itemHolder，并对itemview执行动画。
- animateChange：**变更**动画执行前被调用，执行一些准备工作，并把应该执行动画的item的Holder放到全局变量中供runPendingAnimations执行动画时使用。
- animateChangeImpl：被runPendingAnimations调用，传入itemHolder，并对itemview执行动画。

而实际的动画执行其实就是放在animate*Impl这样的方法中的，下面以add的动画为例
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
上面是摘自DefaultItemAnimator的源码，在animateAdd中设置了itemview在动画执行前的状态，也是就是透明的，同时将持有itemview的holder放入全局的动画列表mPendingAdditions中。
在animateAddImpl中创建了一个渐变的动画并执行，同时在动画执行的回调中调用了一些通知方法。

到此为止，我们从对DefaultItemAnimator源码解析的过程中发现，我们要实现我们自己的动画其实并不难，一个比较激进的做法就是把DefaultItemAnimator的源码拷贝一份，把动画准备和执行的部分替换为我们的自定义动画，虽然看起来不是很有好，但这何尝不是一个方法，而且是用的google官方的实现。
需要实现开篇的那种效果我们只需要修改两行代码：
把ViewCompat.setAlpha(holder.itemView, 0);改为ViewCompat.setTranslationX(holder.itemView, -holder.itemView.getRootView().getWidth());
把animation.alpha(1)改为把animation.translationX(0).setInterpolator(new OvershootInterpolator(1))
具体的实践就不在这里写了，确实能够完美运行。

但之后我们就要考虑一些其他事情了，前面提到，RecyclerView为我们提供了插拔试的动画扩展，但是上述的插拔是否显得太过笨重了呢？毫无疑问，是非常笨重，所以下面就要说一下如何针对itemAnimator实现一个更加轻便的扩展。[（点击跳转）](/2016/03/13/BaseItemAnimator/)



