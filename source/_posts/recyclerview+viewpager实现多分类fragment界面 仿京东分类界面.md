---
title: recyclerview+viewpager实现多分类fragment界面 仿京东分类界面
date: 2018-04-26 17:48:16
---
好久没写博客了，今天决定写一篇简单的功能实现热热手

![在这里插入图片描述](https://img-blog.csdn.net/20181015142049937?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

这是我2018年10月份在京东app录制的他们的分类界面，今天主要就是实现这样的一个分类的界面


## 整理思路
首先整理思路啊。整体界面的实现方式可能很多，但是需要尽可能的用简单的方式，比如左边的分类界面和右边的一起看的话，好像用tablayout+viewpager也可以实现？虽然说他们都是垂直的，我们平时使用的是水平的，但是应该要实现的话也不是问题。
那这篇文章主要讲解的是稍微简单一点的，使用recyclerview+viewpager实现这个界面。
首先新建工程 导入基本的包
```
implementation   'com.android.support:recyclerview-v7:27.1.1'
```
在确定了界面的布局思路之后，首先看下我们的布局文件

## 首页布局
```
<?xml version="1.0" encoding="utf-8"?>


<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#ffffff">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:background="#ffffff"
        android:orientation="horizontal"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <android.support.v7.widget.RecyclerView
            android:id="@+id/follow_category_rv"
            android:layout_width="63dp"
            android:layout_height="match_parent"
            android:layout_marginLeft="3dp"
            android:layout_marginTop="3dp">

        </android.support.v7.widget.RecyclerView>


        <android.support.v4.view.ViewPager
            android:id="@+id/follow_fragment_viewpager"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

        </android.support.v4.view.ViewPager>
    </LinearLayout>

</android.support.constraint.ConstraintLayout>
```
很简单啊，左边一个recyclerview
右边一个viewpager
接下来看下我们的逻辑代码
## 首页代码
首先这里要说下，左边的分类很多情况下是通过接口得到数据后，在创建的，所以这里我们使用一些假数据
```
private var mAdapter: CategoryRecyclerAdapter? = null
    private var mPagerAdapter: FragmentPagerAdapter? = null
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        //模拟假数据 真是开发中可能是通过接口得到
        var list = mutableListOf<String>()
        for (i in 1..10) {
            list.add("test$i")
        }
        showCategory(list)
    }


    private fun showCategory(list: List<String>) {
        //给左边的recyclerView设置数据和adapter
        mAdapter = CategoryRecyclerAdapter()
        follow_category_rv.adapter = mAdapter
        follow_category_rv.layoutManager = LinearLayoutManager(this)
        mAdapter?.setData(list)

        //给右边的viewpager设置adapter
        mPagerAdapter = FragmentPagerAdapter(supportFragmentManager, list)
        follow_fragment_viewpager.adapter = mPagerAdapter

        //recyclerView的item点击事件
        mAdapter?.listener = { _, position ->
            follow_fragment_viewpager.currentItem = position
        }
    }
```
好久没用MK写博客了，为什么代码模块都是红色的。。
看下上面的代码 很简单，我也加了一些注释
关于一些adapter的代码 我会在文末给链接。
先来看看效果
![在这里插入图片描述](https://img-blog.csdn.net/20181015144804279?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
啊看起来还不错啊，但是viewpager是左右滑动的。而且左右滑动viewpager左边的分类也会变，这样好像有点奇怪。
我们接下来开始优化一下

首先来一个viewpager切换动画。
## viwepager切换动画代码

```
/**
 * introduce：这里写介绍
 * createBy：sunwentao
 * email：wentao.sun@yintech.cn
 * time: 9/10/18
 */
class DefaultTransformer : ViewPager.PageTransformer {

    override fun transformPage(view: View, position: Float) {
        var alpha = 0f
        if (position in 0.0..1.0) {
            alpha = 1 - position
        } else if (-1 < position && position < 0) {
            alpha = position + 1
        }
        view.alpha = alpha
        view.translationX = view.width * -position
        val yPosition = position * view.height
        view.translationY = yPosition
    }
}
```
给我们的viewpager设置上看看效果
```
follow_fragment_viewpager.setPageTransformer(true, DefaultTransformer())
```
![在这里插入图片描述](https://img-blog.csdn.net/20181015145307401?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
看起来还不错啊
但是响应viewpager的滑动事件还是左右，现在的gif图我是操作的左边的分类，右边的viewpager响应，如果反过来，viewpager切换还是需要左右滑动，这当然还是不行的。
ps: 有的人不需要viewpager能手动切换，只需要点击点击分类切换
## 上下滑动的viewpager
```

/**
 * introduce：这里写介绍
 * createBy：sunwentao
 * email：wentao.sun@yintech.cn
 * time: 9/10/18
 */
class VerticalViewPager : ViewPager {

    private var noScroll = false

    constructor(context: Context) : super(context) {}

    constructor(context: Context, attrs: AttributeSet) : super(context, attrs) {}

    private fun swapTouchEvent(event: MotionEvent): MotionEvent {
        val width = width.toFloat()
        val height = height.toFloat()
        event.setLocation(event.y / height * width, event.x / width * height)
        return event
    }

    override fun onInterceptTouchEvent(event: MotionEvent): Boolean {
        return super.onInterceptTouchEvent(swapTouchEvent(MotionEvent.obtain(event)))
    }

    override fun onTouchEvent(ev: MotionEvent): Boolean {
        return if (noScroll)
            false
        else
            super.onTouchEvent(swapTouchEvent(MotionEvent.obtain(ev)))
    }

    //如果你想要你的viewpager不要响应滑动 设置这个为true 同时注释掉onInterceptTouchEvent方法
    fun setNoScroll(noScroll: Boolean) {
        this.noScroll = noScroll
    }

}

```
首先看下代码这是我在网上找到的一个别人的实现方式，通过重写viewpager的这些方法，你就能得到如下的一个上下的viewpager。同时我在里面加了一个方法setNoScroll，注意看注释。这个是用来操作viewpager不响应滑动事件用的

![在这里插入图片描述](https://img-blog.csdn.net/20181015145832733?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE1NTI3NzA5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

到这里 好像差不多结束了
最后还是提一嘴
如果你的右侧的fragment是带有上拉加载更多的网我建议

有俩种实现方法
1，简单的实现就是静止viewpager滑动，只能通过左边的分类来控制。这样右边fragment中的操作是完全没问题的，

2，还有一种，就是你重写recyclerview 内部处理好滑动冲突，这样也是能响应上拉加载更多这个事件的
到这里差不多就没了 下面的是链接代码
分多的大佬直接csdn下载
没分的github下载
我真是他miao的考虑周到啊
csdn：https://download.csdn.net/download/qq_15527709/10721472
github：https://github.com/BigSweet/recycler_viewpager
