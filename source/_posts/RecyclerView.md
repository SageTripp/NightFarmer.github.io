---
title: RecyclerView的使用
date: 2015-06-03 18:25:07
tags: [Android, RecyclerView]
category: Android
---

该控件用于在有限的窗口中展示大量数据集，其实这样功能的控件我们并不陌生，例如：ListView、GridView。
那么有了ListView、GridView为什么还需要RecyclerView这样的控件呢？整体上看RecyclerView架构，提供了一种插拔式的体验，高度的解耦，异常的灵活，
通过设置它提供的不同LayoutManager，ItemDecoration , ItemAnimator实现高度自由的效果。
> RecyclerView is a more advanced and flexible version of ListView. 
This widget is a container for large sets of views that can be recycled and scrolled very efficiently. 
Use the RecyclerView widget when you have lists with elements that change dynamically.
<!-- more -->

 根据官方的介绍RecylerView是ListView的升级版，既然如此那RecylerView必然有它的优点，现就RecylerView相对于ListView的优点罗列如下：
* RecylerView封装了viewholder的回收复用，也就是说RecylerView标准化了ViewHolder，编写Adapter面向的是ViewHolder而不再是View了，复用的   逻辑被封装了，写起来更加简单。
* 提供了一种插拔式的体验，高度的解耦，异常的灵活，针对一个Item的显示RecylerView专门抽取出了相应的类，来控制Item的显示，使其的扩展性非常强。例如：你想控制横向或者纵向滑动列表效果可以通过LinearLayoutManager这个类来进行控制(与GridView效果对应的是GridLayoutManager,与瀑布流对应的还有StaggeredGridLayoutManager等)，也就是说RecylerView不再拘泥于ListView的线性展示方式，它也可以实现GridView的效果等多种效果。你想控制Item的分隔线，可以通过继承RecylerView的ItemDecoration这个类，然后针对自己的业务需求去抒写代码。
* 可以控制Item增删的动画，可以通过ItemAnimator这个类进行控制，当然针对增删的动画，RecylerView有其自己默认的实现。

而这里粗略记录一些RecyclerView在日常开发工作中的用法。

想要在项目中使用RecyclerView的话需要在gradle.build文件中新增对RecyclerView的依赖
```
compile 'com.android.support:recyclerview-v7:23.0.0'
```
接下来就可以直接布局文件或者代码中像普通控件一样使用它了，如：
```xml
    <android.support.v7.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>
```
那么最基础的，如何用RecyclerView实现ListView的效果呢
我们像ListView那样，在布局文件中添加如上的一段代码，代表这个控件。
在Activity中获得这个控件，并设置LayoutManager和Adapter
```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.recyclerView);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));//设置LayoutManager
        recyclerView.setAdapter(new MyTestAdapter());//设置Adapter
    }
}
```
对于上面这段代码，有两行是存在疑惑的，LayoutManger和Adapter分别是什么。
从字面上来看我们创建了一个LinearLayoutManager的对象并设置给recyclerView，这代表着这个RecyclerView是默认的线性的列表布局
而MyTestAdapter是我们定义的为recyclerview听过数据源和item布局用的，至于如何定义这个类，下面我们会详细说明。

定义一个RecyclerView的Adapter我们需要定义一个类并继承自RecyclerView.Adapter，并在类的内部定义一个内部类继承自RecyclerView.ViewHolder，
同时将这个ViewHolder的子类作为RecyclerView.Adapter的泛型进行约束，效果如下：
```java
public class MyTestAdapter extends RecyclerView.Adapter<MyTestAdapter.MyViewHolder> {

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        return null;
    }

    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {

    }

    @Override
    public int getItemCount() {
        return 0;
    }

    public class MyViewHolder extends RecyclerView.ViewHolder{
        public MyViewHolder(View itemView) {
            super(itemView);
        }
    }
}

```
adapter继承自RecyclerView.Adapter这个抽象类的同时需要实现三个抽象方法分别是：
onCreateViewHolder	实例化item布局View，并作为ViewHolder的构造参数穿件实例后返回，作为item循环使用的持有者。
onBindViewHolder	在item被展示前对itemview的持有者及itemview进行数据初始化。
getItemCount		设定item的个数。

这里adapter的意义和ListView的adapter的意义基本上是一样的，用于初始化item的View和数据。
下面来看下这个adapter是如何使用的
和ListView一样，我们需要一个item的布局文件用来创建itemview
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tv_title"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:layout_margin="10dp"
        android:background="#d7a547"
        android:gravity="center"
        android:textColor="@android:color/white" />
</RelativeLayout>
```
接下来实现我们的Adapter
```java
public class MyTestAdapter extends RecyclerView.Adapter<MyTestAdapter.MyViewHolder> {

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View itemView = LayoutInflater.from(parent.getContext()).inflate(R.layout.layout_recycler_item, parent, false);
        return new MyViewHolder(itemView);
    }

    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        holder.tv_title.setText("item" + position);
    }

    @Override
    public int getItemCount() {
        return 1000;//这里我们设置了一千条模拟数据，在实际使用过程可以根据数据源的size来获取实际的数据长度。
    }

    public class MyViewHolder extends RecyclerView.ViewHolder {
        TextView tv_title;

        public MyViewHolder(View itemView) {
            super(itemView);
            tv_title = (TextView) itemView.findViewById(R.id.tv_title);
        }
    }
}

```
上面这段代码应该很容易理解，看下效果：
![纵向列表](http://nightfarmer.github.io/public/static/image/Recycler1.gif)

同时RecyclerView也为我们提供了横向列表的展示方式，我们只需要设置LinearLayoutManager的构造参数即可：
```java
recyclerView.setLayoutManager(new LinearLayoutManager(this, LinearLayoutManager.HORIZONTAL, false));
```
效果：
![横向列表](http://nightfarmer.github.io/public/static/image/Recycler-h.gif)

这个特性是不是很酷炫？还有更酷炫的！我们只需要更改RecyclerView的LayoutManager为GridLayoutManager就可以将控件的展现形式从List改为Grid
```java
recyclerView.setLayoutManager(new GridLayoutManager(this, 4));//数据将按4列进行展示
```
效果：
![纵向Grid](http://nightfarmer.github.io/public/static/image/Recycler-grid.gif)
而GridLayoutManager同时也支持指定布局方向，我们可以在GridLayoutManager的构造方法中指定为横向的
```java
recyclerView.setLayoutManager(new GridLayoutManager(this, 4, GridLayoutManager.HORIZONTAL, false));
```
就是这样
![横向Grid](http://nightfarmer.github.io/public/static/image/Recycler-grid-h.gif)

鉴于本篇只是记录RecyclerView的基础用法，有关这个控件的更高级的用法就不在这里描述了。


