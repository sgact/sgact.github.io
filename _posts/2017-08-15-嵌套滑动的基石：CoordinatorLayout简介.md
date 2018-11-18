####一、从官方demo说起
当我们用Android Studio新建一个项目的时候，Android Studio提供了一系列MainActivity的模板，其中就包括Scrolling Activity。
![](http://upload-images.jianshu.io/upload_images/6563996-6b8f92fece3c1a70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
运行这个新建的项目，将看到下面的效果：
![](http://upload-images.jianshu.io/upload_images/6563996-e0a870f20f705e67.gif?imageMogr2/auto-orient/strip)
可以看到，当我滑动下面的文本区（ScrollView的子类）时，**不仅其本身发生了滑动，而且其他View,包括它自己也可以跟着滑动的进度发生一些列的变化**。
####二、什么是CoordinatorLayout
先不讲原理。CoordinatorLayout（协调器布局）是这样一个容器：可以为它的直接子节点定义一种依赖关系，使得当一个View发生改变时，依赖他的View也跟着改变。
####三、CoordinatorLayout布局
CoordinatorLayout直接继承于ViewGroup，那么怎么对他进行复杂的布局呢？
* 使用margin或layout_gravity
他们适用于几乎所有的布局
* 使用layout_anchor或layout_insetEdge
 layout_anchor还是蛮简单的，看下代码：
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/coordinator_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#ebf5ee"
    android:fitsSystemWindows="true">

    <TextView
        android:id="@+id/tv_center"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:layout_margin="25dp"
        android:background="#ff0000"
        android:gravity="center"
        android:padding="30dp"
        android:text="center text"/>

    <TextView
        android:id="@+id/tv_left"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center|start"
        android:background="#3163d7"
        android:padding="30dp"
        android:text="left"
        app:layout_anchor="@id/tv_center"
        app:layout_anchorGravity="left|start|center_vertical"/>

    <TextView
        android:id="@+id/tv_top"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center|top"
        android:background="#72c493"
        android:padding="30dp"
        android:text="top"
        app:layout_anchor="@id/tv_center"
        app:layout_anchorGravity="top|center_horizontal"/>

    <TextView
        android:id="@+id/tv_right"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center|end"
        android:background="#ad56bc"
        android:padding="30dp"
        android:text="right"
        app:layout_anchor="@id/tv_center"
        app:layout_anchorGravity="right|center_vertical"/>

    <TextView
        android:id="@+id/tv_bottom"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center|bottom"
        android:background="#a37551"
        android:padding="30dp"
        android:text="bottom"
        app:layout_anchor="@id/tv_center"
        app:layout_anchorGravity="bottom|center_horizontal"/>
</android.support.design.widget.CoordinatorLayout>
```
![](http://upload-images.jianshu.io/upload_images/6563996-2f0cfebf9f084b66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，使用
```
app:layout_anchor="@id/tv_center"
app:layout_anchorGravity="bottom|center_horizontal"
```
可以实现类似layout_below的效果，即View的位置是相对锚点的。
 - layout_insetEdge 和 layout_dodgeInsetEdges
这两个概念就有点不太好解释了，不过并没有难度。
谷爹的文档上是这样描述的
>Children can specify insetEdge to describe how the view insets the CoordinatorLayout. Any child views which are set to dodge the same inset edges by dodgeInsetEdges will be moved appropriately so that the views do not overlap.

这语法对我来说有点复杂。。。英语熟练的可以自行领悟，无视我下面说的话。
上面这段英文大概意思是讲layout_insetEdge 和 layout_dodgeInsetEdges是两个Gravity类型的属性，即在xml中可以这样写：
```
app:layout_insetEdge="top"
```
现在假设CoordinatorLayout中有两个View，一个设置了layout_insetEdge，另一个设置了layout_dodgeInsetEdges，而且**他们设置的值相同**,比如都设置了xxEdge="top"，那么当设置为layout_insetEdge="top"的View**从上向下移动**时，设置为layout_dodgeInsetEdges="top"的View会**向下闪躲**。
举个例子：

布局:
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <TextView
        android:layout_marginTop="-50dp"
        app:layout_insetEdge="top"
        android:id="@+id/tvMove"
        android:background="@android:color/holo_orange_light"
        android:layout_width="match_parent"
        android:layout_height="50dp"/>

    <TextView
        android:id="@+id/tvFloat"
        android:layout_margin="20dp"
        app:layout_dodgeInsetEdges="top"
        android:background="@android:color/holo_blue_bright"
        android:layout_width="100dp"
        android:layout_height="50dp"/>

</android.support.design.widget.CoordinatorLayout>
```
效果如下:

![效果图](http://upload-images.jianshu.io/upload_images/6563996-0b2fc63ec48df586.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
注意：我将第一个View的marginTop设为负数，让他在屏幕之外。
Activity代码：
```
    private void initView() {
        tvMove = (TextView) findViewById(R.id.tvMove);
        tvFloat = (TextView) findViewById(R.id.tvFloat);

        final ObjectAnimator oa = ObjectAnimator.ofFloat(tvMove, "Y", dp2px(-50), dp2px(0)).setDuration(2000);

        tvFloat.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                oa.start();
            }
        });
    }
```
给第一个View设置了一个属性动画，让他从上向下移动。最终的效果就是这样，很直观：

![效果图](http://upload-images.jianshu.io/upload_images/6563996-e3eba7fe7ee6e070.gif?imageMogr2/auto-orient/strip)
费了好大劲效果好像也没啥特别的么。。。
* 在behavior类用Java进行布局
这个后面再说，先放一放
