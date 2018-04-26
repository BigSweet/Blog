首先看下效果图和整个项目的结构
![](http://upload-images.jianshu.io/upload_images/7568660-e94dc20ae03944a6.gif?imageMogr2/auto-orient/strip)

![image.png](http://upload-images.jianshu.io/upload_images/7568660-7f221c02394eca20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很简单的一个项目，这也是我在简书写的第一个项目，虽然简单，但是我觉得很漂亮

现在开始分析代码

首先看下MainActivity的代码
```
public class MainActivity extends AppCompatActivity {
    private int pagerWidth;
    private ViewPager mViewPager;
    private LinearLayout mNumLayout;//小圆点的layout
    private List<String> imgList; //图片资源list
    private RelativeLayout ll_main; //viewpager的父布局

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ll_main = (RelativeLayout) findViewById(R.id.activity_main);
        mViewPager = (ViewPager) findViewById(R.id.viewPager);
        mNumLayout = (LinearLayout) findViewById(R.id.bottom_layout);
        mViewPager.setOffscreenPageLimit(3);
        initImg();
        //设置viewpager的宽度为屏幕的3/1 看起来比较好看,设置每个item的边距-10 34-43行
        pagerWidth = (int) (getResources().getDisplayMetrics().widthPixels * 3.0f / 9.0f);
        ViewGroup.LayoutParams lp = mViewPager.getLayoutParams();
        if (lp == null) {
            lp = new ViewGroup.LayoutParams(pagerWidth, ViewGroup.LayoutParams.MATCH_PARENT);
        } else {
            lp.width = pagerWidth;
        }
        mViewPager.setLayoutParams(lp);
        mViewPager.setPageMargin(-10);
        //将主布局的触摸事件分发给viewpager。这样整个界面都可以响应viewpager的滑动事件，
        // 如果不设置，滑动事件只有中间viewpager的宽度区域
        ll_main.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View view, MotionEvent motionEvent) {
                return mViewPager.dispatchTouchEvent(motionEvent);
            }
        });
        //设置切换时候的动画效果
        mViewPager.setPageTransformer(true, new GallyPageTransformer());
        //这个就是设置adapter
        mViewPager.setAdapter(new MyViewPagerAdapter(this, imgList));
        //设置默认第二项，没有为什么就是看起来漂亮(从0开始的，所以是setcurrentitem(1))
        mViewPager.setCurrentItem(1, true);
        //设置下面的小点点默认第二项
        mNumLayout.getChildAt(1).setBackgroundResource(R.drawable.msg_img_ellipse13x);

        mViewPager.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {

            }

            @Override
            public void onPageSelected(int position) {
                //首先还原所有的小点点
                for (int i = 0; i < mNumLayout.getChildCount(); i++) {
                    mNumLayout.getChildAt(i).setBackgroundResource(R.drawable.msg_img_ellipse33x);
                }
                //在根据position拿到当前的位置，设置小点点的选中图片
                Button currentBt = (Button) mNumLayout.getChildAt(position);
                currentBt.setBackgroundResource(R.drawable.msg_img_ellipse13x);
            }

            @Override
            public void onPageScrollStateChanged(int state) {

            }
        });
    }

    //初始化数据
    public void initImg() {
        imgList = new ArrayList<>();
        imgList.add("https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1490984457722&di=6d7d3e20e07fc833fc606089d01132e6&imgtype=0&src=http%3A%2F%2Fimgst.izhangheng.com%2F2016%2F08%2Fnight-beauty-girl-3.jpg");
        imgList.add("https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=2946550071,381041431&fm=11&gp=0.jpg");
        imgList.add("https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1490984320392&di=8290126f83c2a2c0d45be41e3f88a6d0&imgtype=0&src=http%3A%2F%2Ffile.mumayi.com%2Fforum%2F201307%2F19%2F152440r9ov9ololkzdcz7d.jpg");
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.msg_img_ellipse33x);
        //根据图片list的大小，创建下面的小圆点按钮
        for (int i = 0; i < imgList.size(); i++) {
            Button bt = new Button(this);
            LinearLayout.LayoutParams params1 = new LinearLayout.LayoutParams(bitmap.getWidth(), bitmap.getHeight());
            params1.setMargins(dip2px(5), 0, 0, 0);
            bt.setLayoutParams(params1);
            bt.setBackgroundResource(R.drawable.msg_img_ellipse33x);
            mNumLayout.addView(bt);
        }
    }

    //dip转px的小工具方法
    public int dip2px(float dpValue) {
        final float scale = getResources().getDisplayMetrics().density;
        return (int) (dpValue * scale + 0.5f);
    }
}

```

首先实例化控件，

initImg();准备数据源

34-43行//设置viewpager的宽度为屏幕的3/1 看起来比较好看,设置每个item的边距-10 

46-51行//将主布局的触摸事件分发给viewpager。这样整个界面都可以响应viewpager的滑动事件，

// 如果不设置，滑动事件只有中间viewpager的宽度区域

接下来设置滑动事件，和适配器，我在代码上面都有注释，应该能看得懂

接下来看下MyApplication
```
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        Fresco.initialize(this);
    }
}
```

viewpager的中间图片我用的是SimpleDraweeView

所以在application中初始化了一下

接下来看下viewpager的适配器
```
public class MyViewPagerAdapter extends PagerAdapter {

    private List<String> mlists;
    private List<View> mSimpleDraweeViewList;
    private HashMap<Integer, View> mViewIntegerHashMap = new HashMap<>();
    private Context mContext;

    public MyViewPagerAdapter(Context context, List<String> lists) {
        mContext = context;
        this.mlists = lists;
        mSimpleDraweeViewList = new ArrayList<>();
    }

    @Override
    public int getCount() {
        return mlists.size();
    }

    @Override
    public boolean isViewFromObject(View view, Object object) {
        return view == object;
    }

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        View view;
        if (mViewIntegerHashMap.get(position) == null) {
            view = View.inflate(mContext, R.layout.main_top_viewpager_item, null);
            SimpleDraweeView simpleDraweeView = (SimpleDraweeView) view.findViewById(R.id.main_top_viewpager_item_simpl);
//            FrescoUtils.loadImage(mContext,mlists.get(position),simpleDraweeView,10,10,10,10);
//            simpleDraweeView.setImageResource(R.mipmap.ic_launcher);
            simpleDraweeView.setImageURI(mlists.get(position));
            container.addView(view);
            mSimpleDraweeViewList.add(view);
            mViewIntegerHashMap.put(position, view);
        } else {
            view = mViewIntegerHashMap.get(position);
        }
        return view;
     /*   SimpleDraweeView simpleDraweeView = new SimpleDraweeView(mContext);
        simpleDraweeView.setImageURI(mlists.get(position));
        container.addView(simpleDraweeView);
        mSimpleDraweeViewList.add(simpleDraweeView);
        return simpleDraweeView;*/
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        container.removeView(mViewIntegerHashMap.get(position));
    }
}
```

item布局为
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                xmlns:app="http://schemas.android.com/apk/res-auto"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
    >

    <com.facebook.drawee.view.SimpleDraweeView
        android:id="@+id/main_top_viewpager_item_simpl"
        android:layout_width="150dp"
        android:layout_height="180dp"
        app:roundedCornerRadius="5dp"
        app:roundingBorderColor="#ffffffff"
        app:roundingBorderWidth="2dp"/>
</RelativeLayout>
```
item布局就是一个SimpleDraweeView

主要看下instantiateItem方法

首先判断if(mViewIntegerHashMap.get(position) ==null) {

如果为空，就新建一个并且存起来，

下次直接拿出对应postiion的view返回

防止重复创建

最后看下GallyPageTransformer代码
```
package cn.swt.m.btviewpager;

import android.support.v4.view.ViewPager;
import android.util.Log;
import android.view.View;

/**
 * 介绍：这里写介绍
 * 作者：sweet
 * 邮箱：sunwentao@priemdu.cn
 * 时间: 2017/7/19
 */
public class GallyPageTransformer implements ViewPager.PageTransformer {

    private static final float min_scale = 0.85f;

    @Override
    public void transformPage(View page, float position) {

        //最小的缩放比例
        float scaleFactor = Math.max(min_scale, 1 - Math.abs(position));

        //旋转的度数
        float rotate = 40 * Math.abs(position);
        Log.d("ss", "position" + position);
        if (position < 0) {
            //中间图片左边的图片
            page.setScaleX(scaleFactor);
            page.setScaleY(scaleFactor);
            page.setRotationY(rotate);
        } else if (position >= 0 && position < 1) {
            //中间图片
            page.setScaleX(scaleFactor);
            page.setScaleY(scaleFactor);
            page.setRotationY(-rotate);
        } else if (position >= 1) {
            //中间图片右边的图片
            page.setScaleX(scaleFactor);
            page.setScaleY(scaleFactor);
            page.setRotationY(-rotate);
        }
    }
}

```

主要是这个position会跟着滑动的距离进行改变，所以想要动画效果看起来很流畅，就得利用这个position函数

代码不多 我都做了注释

好像还差一个mainactivity的布局

贴在这里
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
   >

    <RelativeLayout
        android:id="@+id/activity_main"
        android:layout_width="match_parent"
        android:layout_height="200dp"
        android:clipChildren="false"
        android:layerType="software">
        <ImageView
            android:layout_width="match_parent"
            android:layout_height="200dp"
            android:scaleType="fitXY"
            android:src="@drawable/tooopen_sy_169964521661"/>

        <android.support.v4.view.ViewPager
            android:id="@+id/viewPager"
            android:layout_centerInParent="true"
            android:layout_width="match_parent"
            android:layout_height="150dp"
            >
        </android.support.v4.view.ViewPager>

        <LinearLayout
            android:paddingTop="5dp"
            android:id="@+id/bottom_layout"
            android:layout_width="match_parent"
            android:layout_height="40dp"
            android:layout_alignParentBottom="true"
            android:gravity="center"
            android:orientation="horizontal">
        </LinearLayout>
    </RelativeLayout>
</RelativeLayout>

```

代码下载地址http://download.csdn.net/download/qq_15527709/9960527