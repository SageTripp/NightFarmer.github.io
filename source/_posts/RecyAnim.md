---
title: 给滚动列表项增加渐显载入动画
date: 2016-05-24 15:26:17
tags: [Android, RecyclerView]
category: Android
---

之前还在使用ListView作为展示列表数据的控件时尝试过给列表项增加载入动画，但是ListView是一个效果比较单一的控件，
在想要实现表格布局或者是瀑布流的载入动画的时候还要分别给GridView和瀑布流控件分别实现，这是比较麻烦的。
在之前的博客中提到过RecyclerView这个控件，这是在V7包中提供的一个功能非常强大而且扩展性非常强的一个控件，
用它可以实现列表、表格、瀑布流等效果，只需要指定不同的LayoutManager即可，具体的用法在之前的博客中已经记录过，
在这里就不再赘述。
今天记录的是给RecyclerView增加item的载入动画，这样在借助RecyclerView的特性下在不同效果的布局模式中就可以显示相同的效果，
而不用分别进行实现，下面是效果：
![列表动画](http://nightfarmer.github.io/public/static/image/listanim.gif) ![grid动画](http://nightfarmer.github.io/public/static/image/gridanim1.gif) ![瀑布流动画](http://nightfarmer.github.io/public/static/image/gridanim2.gif)
<!-- more -->

依照惯例，我们定义一个带有RecyclerView的Activity，然后放一些测试数据进去：

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.recyclerView);

        recyclerView.setLayoutManager(new LinearLayoutManager(this));
//        recyclerView.setLayoutManager(new GridLayoutManager(this ,3));
//        recyclerView.setLayoutManager(new StaggeredGridLayoutManager(3, OrientationHelper.VERTICAL));

        TestAnimAdapter testAnimAdapter = new TestAnimAdapter();
        recyclerView.setAdapter(testAnimAdapter);
    }
}
```

然后是Adapter

```java
public class TestAnimAdapter extends RecyclerView.Adapter<TestAnimAdapter.MyViewHolder> {

    int lastPosition = -1;

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View inflate = LayoutInflater.from(parent.getContext()).inflate(R.layout.layout_list_item, parent, false);
        return new MyViewHolder(inflate);
    }

    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        holder.tv_title.setText("测试" + position);
    }

    @Override
    public int getItemCount() {
        return 1000;
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

现在用RecyclerView实现的简单的带有1000条测试数据的列表UI构建完毕，接下来就是如何添加动画了。
RecyclerView在item即将进入可视区域时会调用onBindViewHolder方法，传入被recycle的holder进行初始化数据，
那么我们就可以在这里下手，在item初始化数据的时候指定一个动画，来让item进入用户视野的时候伴随着动画的执行。
下面以一个渐现动画为例：
```java
    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        holder.tv_title.setText("测试" + position);

        int adapterPosition = holder.getAdapterPosition();
        Animator animator = ObjectAnimator.ofFloat(holder.itemView, "alpha", 0, 1f);
        animator.setDuration(1000).start();
    }
```

上面这段为item进入视野时增加了一个渐现的动画，而实际运行的时候好像和我们的预期有一些偏差
![和预期有偏差的动画效果](http://nightfarmer.github.io/public/static/image/listanim2.gif)
我们发现在列表上拉的时候也会有一个动画效果，而这并不是我们想要的，这需要怎么处理呢？
我们需要动画在一个item只会在首次进入视野的时候执行动画，而不是每次都执行。
holder的position代表这item的位置，我们存储一个代表当前加载到的最大位置的变量，如果加载到的item的position大于这个变量，则是首次加载。
接下来改造代码：
```java
    int lastPosition = -1;

    @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        holder.tv_title.setText("测试" + position);
		
        int adapterPosition = holder.getAdapterPosition();
        if (adapterPosition > lastPosition) {
            Animator animator = ObjectAnimator.ofFloat(holder.itemView, "alpha", 0, 1f);
            animator.setDuration(1000).start();
            lastPosition = adapterPosition;
        } else {
            ViewCompat.setAlpha(holder.itemView, 1);
        }
    }
```

OK，搞定！
这时候再执行我们的代码就可以实现开篇上面的动画了，至于Grid和瀑布流(StaggeredGrid)的样式我们并不需要单独实现，只要更改RecyclerView的LayoutManager即可：
```java
//        recyclerView.setLayoutManager(new LinearLayoutManager(this));//列表
        recyclerView.setLayoutManager(new GridLayoutManager(this ,3));//3列gird
//        recyclerView.setLayoutManager(new StaggeredGridLayoutManager(3, OrientationHelper.VERTICAL));//3列瀑布流

```

