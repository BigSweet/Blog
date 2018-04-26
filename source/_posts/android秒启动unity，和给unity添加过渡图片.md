1，本篇不讲解unity如何集成，网上很多，主要讲解下面几个点
     一，最容易出现的bug
     二，快速启动unity
     三，帮unity添加过渡图和可能会遇到的问题



一，android 在退出unity的时候，unity会执行结束进程，同时也会结束掉你的APP的进程
解决办法是
重写unityplay，重写kill方法。搞定
```
public class MyUnityPlay extends UnityPlayer {
    public MyUnityPlay(ContextWrapper contextWrapper) {
        super(contextWrapper);
    }

    @Override
    protected void kill() {
    }


}
```


二，快速启动unity
首先看下unity导出后的默认代码
```
package com.sunwentao.liuliuqiu;

import com.unity3d.player.*;
import android.app.Activity;
import android.content.Intent;
import android.content.res.Configuration;
import android.graphics.PixelFormat;
import android.os.Bundle;
import android.view.KeyEvent;
import android.view.MotionEvent;
import android.view.View;
import android.view.Window;
import android.view.WindowManager;

public class UnityPlayerActivity extends Activity
{
    protected UnityPlayer mUnityPlayer; // don't change the name of this variable; referenced from native code

    // Setup activity layout
    @Override protected void onCreate (Bundle savedInstanceState)
    {
        requestWindowFeature(Window.FEATURE_NO_TITLE);
        super.onCreate(savedInstanceState);

        getWindow().setFormat(PixelFormat.RGBX_8888); // <--- This makes xperia play happy

        mUnityPlayer = new UnityPlayer(this);
        setContentView(mUnityPlayer);
        mUnityPlayer.requestFocus();
    }

    @Override protected void onNewIntent(Intent intent)
    {
        // To support deep linking, we need to make sure that the client can get access to
        // the last sent intent. The clients access this through a JNI api that allows them
        // to get the intent set on launch. To update that after launch we have to manually
        // replace the intent with the one caught here.
        setIntent(intent);
    }

    // Quit Unity
    @Override protected void onDestroy ()
    {
        mUnityPlayer.quit();
        super.onDestroy();
    }

    // Pause Unity
    @Override protected void onPause()
    {
        super.onPause();
        mUnityPlayer.pause();
    }

    // Resume Unity
    @Override protected void onResume()
    {
        super.onResume();
        mUnityPlayer.resume();
    }

    // Low Memory Unity
    @Override public void onLowMemory()
    {
        super.onLowMemory();
        mUnityPlayer.lowMemory();
    }

    // Trim Memory Unity
    @Override public void onTrimMemory(int level)
    {
        super.onTrimMemory(level);
        if (level == TRIM_MEMORY_RUNNING_CRITICAL)
        {
            mUnityPlayer.lowMemory();
        }
    }

    // This ensures the layout will be correct.
    @Override public void onConfigurationChanged(Configuration newConfig)
    {
        super.onConfigurationChanged(newConfig);
        mUnityPlayer.configurationChanged(newConfig);
    }

    // Notify Unity of the focus change.
    @Override public void onWindowFocusChanged(boolean hasFocus)
    {
        super.onWindowFocusChanged(hasFocus);
        mUnityPlayer.windowFocusChanged(hasFocus);
    }

    // For some reason the multiple keyevent type is not supported by the ndk.
    // Force event injection by overriding dispatchKeyEvent().
    @Override public boolean dispatchKeyEvent(KeyEvent event)
    {
        if (event.getAction() == KeyEvent.ACTION_MULTIPLE)
            return mUnityPlayer.injectEvent(event);
        return super.dispatchKeyEvent(event);
    }

    // Pass any events not handled by (unfocused) views straight to UnityPlayer
    @Override public boolean onKeyUp(int keyCode, KeyEvent event)     { return mUnityPlayer.injectEvent(event); }
    @Override public boolean onKeyDown(int keyCode, KeyEvent event)   { return mUnityPlayer.injectEvent(event); }
    @Override public boolean onTouchEvent(MotionEvent event)          { return mUnityPlayer.injectEvent(event); }
    /*API12*/ public boolean onGenericMotionEvent(MotionEvent event)  { return mUnityPlayer.injectEvent(event); }
}

```
去掉ondestory方法，当用户点击了unity的退出按钮的时候，会回调android的一个方法，比如是exit
,
```
  public void exit(){
    mUnityPlayer.pause();
        moveTaskToBack(true);
  }
```
首先暂停unity,然后将当年的界面移到后台，不关闭unityplayactivity这个界面，
有的人肯定会问 这样不会耗资源吗，只要你调用了mUnityPlayer.pause()
那么就不怎么会耗你的资源，从你的手机发烫的程度就可以知道，如果你开着unity10-20分钟你肯定能感觉你的手机很烫了，但是你在退出的时候调用暂停这个方法，手机就不会发烫，说明unity消耗的资源不多，
当然unity那边的场景在每次你点击退出的时候都应该清掉

从上面的代码可以看到我们并没有关闭unityplay这个界面，所以我们再次打开是很快的，基本上是秒开，
所以和unity的交互不能写在oncreate里面，应该写在unresume里面，这样每次你进来都能重新调用unity，并加载出界面比如下面这样
```
    // Resume Unity
    @Override
    protected void onResume() {
        super.onResume();

        new Handler().post(new Runnable() {
            @Override
            public void run() {

//                if (!isFirst){
                mUnityPlayer.resume();
//                }
                UnityPlayer.currentActivity.runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        if (CURREN_MODE == xxxx) {
                            UnityPlayer.UnitySendMessage("xxxxx", "xxxx", "");
                        } else if (CURREN_MODE == xxxx) {
                            UnityPlayer.UnitySendMessage("xxxxx", "xxxxx", xxx);
                        } else if (CURREN_MODE == xxxxx) {
                            UnityPlayer.UnitySendMessage("xxxxx", "xxxxx", xxx);
                        }
                    }
                });


            }
        });
    }
```
调用 mUnityPlayer.resume() 
让unity重新开始工作，接着发送交互信息，这样就实现了，你只需要加载一次unity界面，下次你在进来是非常快的(做过unity集成的都知道unity在安卓手机上启动是有多慢);


三，添加过渡页
首先看下代码
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        requestWindowFeature(Window.FEATURE_NO_TITLE);
        super.onCreate(savedInstanceState);
        getWindow().setFormat(PixelFormat.RGBX_8888); // <--- This makes xperia play happy
        mUnityPlayer = new MyUnityPlay(this);
        SpUtils.setConext(this);
        setContentView(mUnityPlayer);
        mUnityPlayer.requestFocus();
            try {
                bgView = new ImageView(UnityPlayer.currentActivity);
                bgView.setImageResource(R.drawable.unityloading2x);
                bgView.setScaleType(ImageView.ScaleType.FIT_XY);
                Resources r = mUnityPlayer.currentActivity.getResources();
                mUnityPlayer.addView(bgView, r.getDisplayMetrics().widthPixels, r.getDisplayMetrics().heightPixels);
            } catch (Exception e) {
                e.printStackTrace();
            }

    }
```

代码很简单，这样写不会出现黑屏的问题，我以前用的dialog显示一张图片，很大的概率出现黑屏，可能是dialog挡住了unityplay的绘制。
上面这样写是没问题的，
然后在unity开始执行第一帧的时候，调用你的一个方法，你在把这个image移除掉
```
   try {
                if (bgView == null)
                    return;
                UnityPlayer.currentActivity.runOnUiThread(new Runnable() {
                    public void run() {
                        mUnityPlayer.removeView(bgView);
                        bgView = null;
                    }

                });

            } catch (Exception e) {
                e.printStackTrace();
            }
```
这个方法你和unity那边沟通确定好就行了，方法名都是随便定义的

就这么多，有问题欢迎讨论


