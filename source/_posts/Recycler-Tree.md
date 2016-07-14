---
title: 实现基于RecyclerView的树形组件封装
date: 2016-06-24 15:26:17
tags: [Android, RecyclerView]
category: Android
---

最近因项目需求需要一个可以进行展开收缩的多级树形控件，首先考虑的是ExpandableListView，但是这个控件只支持一级展开不符合需求，
之后在百度谷歌了一下发现有用List实现的多级树形控件，预览效果是不错的，但是经测试大量数据展开时会造成严重的卡顿现象，也不符合需求。
最终决定以RecyclerView封装一个可服用的多级树形组件，以满足大量数据的展开折叠，以及展开折叠的动画效果。

最终效果图：
<!-- more -->
![员工效果图](http://nightfarmer.github.io/public/static/image/staffTree.gif)


这个树形组件是在RecyclerView的基础上对Adapter进行了封装，在使用时可以使用原生的RecyclerView设定为树形结构的Adapter即可实现树形的效果。

首先我们定义一个枚举来标识树形节点的类型(叶子/分支)
```java
/**
 * 节点类型
 * Created by zhangfan on 16-6-24.
 */
public enum NodeType {
    /**
     * 分组节点，可打开
     */
    BRANCH,
    /**
     * 叶子节点
     */
    LEAF
}

```

接下来我们考虑树形节点的数据存储模型，有两种方案，一种是定义一个节点抽象类，另一种是定义一个节点接口。
这两种方案对于开发者来说，抽象类可以存储组件开放过程中需要的一些临时数据，更加方便，而接口则需要组件使用其他折中的方法来存储节点的临时数据
而对于调用者来说，继承一个接口会更加的灵活，而且不需要额外的数据转换开销，而继承抽象类则会约束使用者只能将父类指定为组件提供的类，在通过其他途径拿到节点数据之后还要进行额外的转换(如数据库/网络等)。

这里为了确保组件在使用过程中能更加的简洁清爽，更小的入侵性，我们采用接口来定义树形节点：
```java
/**
 * 树形节点
 * Created by zhangfan on 16-6-24.
 */
public interface TreeNode {

    /**
     *获取子节点
     */
    ArrayList<? extends TreeNode> getChildren();

    /**
     *获取节点名称
     */
    String getTitle();

    /**
     *获取节点类型
     */
    NodeType getNodeType();
}

```

这样我们的数据模型已经定义完毕，然后我需要定义一个拥有树形特性的Adapter来让RecyclerView具有树形控件的特性。
```java

/**
 * 树形adapter
 * Created by zhangfan on 16-6-24.
 */
public abstract class RecyclerTreeAdapter<N extends TreeNode, T extends RecyclerView.ViewHolder> extends RecyclerView.Adapter<T> {

    ArrayList<N> dataList = new ArrayList<>();

    HashMap<N, Boolean> nodeChildrenStatusMap = new HashMap<>();

    HashMap<N, Integer> nodeLevelMap = new HashMap<>();

    HashSet<N> checkedNodes = new HashSet<>();

    /**
     * @return 每级缩进，单位px
     */
    protected abstract int indentPerLevel();

    /**
     * @param dataList 已传入的数据参数在控件展开和折叠后会产生变化，不可在其他地方引用或使用
     */
    public RecyclerTreeAdapter(List<N> dataList) {
        this.dataList.addAll(dataList);
    }

    public ArrayList<N> getCheckedNodes() {
        return new ArrayList<>(checkedNodes);
    }

    @Override
    public final T onCreateViewHolder(ViewGroup parent, int viewType) {
        NodeType nodeType;
        if (viewType == 0) {
            nodeType = NodeType.LEAF;
        } else {
            nodeType = NodeType.BRANCH;
        }
        return onCreateViewHolder(parent, nodeType);
    }

    protected abstract T onCreateViewHolder(ViewGroup parent, NodeType nodeType);

    @Override
    public final void onBindViewHolder(final T holder, int position) {
        N treeNode = dataList.get(position);
        Integer integerLevel = nodeLevelMap.get(treeNode);
        int level = integerLevel == null ? 0 : integerLevel;
        onBindViewHolder(holder, treeNode);
        final View itemView = holder.itemView;
        itemView.setPadding(level * indentPerLevel(), itemView.getPaddingTop(),
                itemView.getPaddingRight(), itemView.getPaddingBottom());
        itemView.setTag(treeNode);
        itemView.setClickable(true);
        itemView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                N treeNode = (N) itemView.getTag();
                switch (treeNode.getNodeType()) {
                    case BRANCH:

                        int location = dataList.indexOf(treeNode);
                        ArrayList<? extends TreeNode> children = treeNode.getChildren();
                        if (children.size() > 0) {
                            Boolean aBoolean = nodeChildrenStatusMap.get(treeNode);
                            boolean isShowChildren = aBoolean != null && aBoolean;
                            if (isShowChildren) {
                                List<N> allShowedChildren = getAllShowedChildren(treeNode);
                                dataList.removeAll(allShowedChildren);
                                notifyItemRangeRemoved(location + 1, allShowedChildren.size());
                                updateChildrenStatusMap(treeNode, false);
                            } else {
                                Integer integer = nodeLevelMap.get(treeNode);
                                int thisLevel = integer == null ? 0 : integer;
                                for (TreeNode n :
                                        children) {
                                    nodeLevelMap.put((N) n, thisLevel + 1);
                                }
                                dataList.addAll(location + 1, (Collection<? extends N>) children);
                                notifyItemRangeInserted(location + 1, children.size());
                                nodeChildrenStatusMap.put(treeNode, true);
                            }
                        }
                        break;

                    case LEAF:
                        boolean checked = checkedNodes.contains(treeNode);
                        setNodeChecked(treeNode, !checked);
                        onLeafTouch(treeNode);
                        break;
                }
            }
        });
    }

    private void updateChildrenStatusMap(N treeNode, boolean isShowChildren) {
        nodeChildrenStatusMap.put(treeNode, isShowChildren);
        ArrayList<? extends TreeNode> children = treeNode.getChildren();
        if (children == null) return;
        for (TreeNode n :
                children) {
            Boolean aBoolean = nodeChildrenStatusMap.get(n);
            if ((aBoolean != null && aBoolean) == isShowChildren) continue;
            updateChildrenStatusMap((N) n, isShowChildren);
        }
    }

    private List<N> getAllShowedChildren(N treeNode) {
        List<N> result = new ArrayList<>();
        Boolean aBoolean = nodeChildrenStatusMap.get(treeNode);
        boolean showed = aBoolean != null && aBoolean;
        if (showed) {
            ArrayList<? extends TreeNode> children = treeNode.getChildren();
            if (children == null) return result;
            result.addAll((Collection<? extends N>) children);
            for (TreeNode n : children) {
                result.addAll(getAllShowedChildren((N) n));
            }
        }
        return result;
    }


    protected void onLeafTouch(N treeNode) {
    }

    public void setNodeChecked(N treeNode, boolean checked) {
        if (checked) {
            checkedNodes.add(treeNode);
        } else {
            checkedNodes.remove(treeNode);
        }
        ArrayList<? extends TreeNode> children = treeNode.getChildren();
        if (children == null) return;
        for (TreeNode n : children) {
            setNodeChecked((N) n, checked);
        }
    }

    /**
     * 获取节点层级，只能获取已经加载的节点
     *
     * @param treeNode 节点
     * @return 节点层级，从0开始
     */
    public int getNodeLevel(N treeNode) {
        Integer integer = nodeLevelMap.get(treeNode);
        return integer == null ? 0 : integer;
    }


    /**
     * 数据和UI绑定
     *
     * @param holder   holder内itemView的tag方法已被使用，不建议重新设置该值
     * @param treeNode 节点数据
     */
    public abstract void onBindViewHolder(T holder, N treeNode);


    /**
     * 节点类型，1分支，0叶子
     *
     * @param position
     * @return
     */
    @Override
    public int getItemViewType(int position) {
        return NodeType.BRANCH.equals(dataList.get(position).getNodeType()) ? 1 : 0;
    }

    @Override
    public int getItemCount() {
        return dataList.size();
    }

}

```
在RecyclerTreeAdapter中根据节点类型来绑定节点的点击事件，分支节点的点击事件指定为展开/折叠子节点，叶子节点的点击事件则通过onLeafTouch方法回调给具体的Adapter实现类。
而树形结构的缩进则是通过计算出节点的所属层级，给相应节点的View设定leftPadding。
这里可能会注意到，在RecyclerTreeAdapter中定义了一些Map这样的结构数据，这是因为我们采用了接口来定义节点，所以节点的展开/收缩状态以及节点的层级数据都不能够存储在节点中，所以需要这些数据结构来存储这些堆数据。

至此，我们的树形控件已经全部封装完毕，下面看一下这个控件该如何使用
首先让我们的节点类实现TreeNode
```java
/**
 *
 * Created by zhangfan on 16-6-24.
 */
public class TestNode implements TreeNode {
    ArrayList<TestNode> children;
    String name;
    NodeType type = NodeType.BRANCH;

    public TestNode(ArrayList<TestNode> children, String name) {
        this.children = children;
        this.name = name;
    }

    @Override
    public ArrayList<? extends TreeNode> getChildren() {
        return children;
    }

    @Override
    public String getTitle() {
        return name;
    }

    @Override
    public NodeType getNodeType() {
        return type;
    }
}

```
然后定义一个Adapter继承自刚才的RecyclerTreeAdapter
```java
/**
 * 树形Adapter的实现
 * Created by zhangfan on 16-6-24.
 */
public class MyTreeAdapter extends RecyclerTreeAdapter<TestNode, MyTreeAdapter.MyHolder> {

    private final Context context;

    /**
     * @param dataList 已传入的数据参数在控件展开和折叠后会产生变化，不可在其他地方引用或使用
     */
    public MyTreeAdapter(List<TestNode> dataList, Context context) {
        super(dataList);
        this.context = context;
    }

    @Override
    protected int indentPerLevel() {
        return 50;
    }

    @Override
    protected MyHolder onCreateViewHolder(ViewGroup parent, NodeType nodeType) {
        View inflate = null;
        switch (nodeType) {
            case LEAF:
                inflate = LayoutInflater.from(parent.getContext()).inflate(R.layout.treeitem2, parent, false);
                break;
            case BRANCH:
                inflate = LayoutInflater.from(parent.getContext()).inflate(R.layout.treeitem, parent, false);
                break;
        }
        return new MyHolder(inflate);
    }

    @Override
    public void onBindViewHolder(MyHolder holder, TestNode treeNode) {
        holder.tv.setText("" + treeNode.getTitle() + ",第" + getNodeLevel(treeNode) + "层");
    }

    public class MyHolder extends RecyclerView.ViewHolder {
        TextView tv;

        public MyHolder(View itemView) {
            super(itemView);
            tv = (TextView) itemView.findViewById(R.id.title);
        }
    }

    @Override
    protected void onLeafTouch(TestNode treeNode) {
        super.onLeafTouch(treeNode);
        Toast.makeText(context, "点到叶子了:" + treeNode.getTitle() + ",第" + getNodeLevel(treeNode) + "层", Toast.LENGTH_SHORT).show();
    }

}

```
在这个adapter中我们可用根据节点类型的不同来生成同步类型的节点View
然后在Activity中造一些模拟数据给RecyclerView指定Adapter
```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RecyclerView recyclerView = (RecyclerView) findViewById(R.id.recyclerView);

        ArrayList<TestNode> list = new ArrayList<>();

        ArrayList<TestNode> nodes1 = new ArrayList<>();
        TestNode node1 = new TestNode(nodes1, "1层");
        list.add(node1);

        ArrayList<TestNode> nodes2 = new ArrayList<>();
        TestNode node2 = new TestNode(nodes2, "2层");
        nodes1.add(node2);

        ArrayList<TestNode> nodes3 = new ArrayList<>();
        TestNode node3 = new TestNode(nodes3, "3层");
        nodes2.add(node3);

        ArrayList<TestNode> nodes4 = new ArrayList<>();
        TestNode node4 = new TestNode(nodes4, "4层");
        node4.type = NodeType.LEAF;
        nodes3.add(node4);

        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        recyclerView.setAdapter(new MyTreeAdapter(list, this));
    }
}


```

大功告成，效果图：
![treeDemo](http://nightfarmer.github.io/public/static/image/treetest.gif)

代码地址：https://github.com/NightFarmer/RecyclerTree

