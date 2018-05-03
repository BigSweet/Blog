---
title: andorid面试题和笔记
date: 2018-04-26 17:48:16
---
##事件分发机制
首先事件指的是触摸事件，首先是viewgroup的事件分发，viewgroup里面有子view，ViewGroup的相关事件有三个：onInterceptTouchEvent、dispatchTouchEvent、onTouchEvent。View的相关事件只有两个：dispatchTouchEvent、onTouchEvent。

简单的来说就是viewgroup遍历自己的子view，如果子view中有viewgroup，就继续遍历这个viewgroup的子view，都是调用的dispatchTouchEvent来分发事件，dispatchTouchEvent会返回一个布尔值类型的参数，事件会一直分发，一直在某个view调用dispatchTouchEvent返回true,表示事件分发到此结束，返回true的这个view就是需要接受这个事件的view，

ViewGroup的dispatchTouchEvent是真正在执行“分发”工作，而View的dispatchTouchEvent方法，并不执行分发工作，或者说它分发的对象就是自己，决定是否把touch事件交给自己处理，而处理的方法，便是onTouchEvent事件，这里说到了view的onTouchEvent事件是这个时候开始执行，那么viewgroup的onTouchEvent什么时候执行呢？，当所有的子view调用dispatchTouchEvent都是返回false的时候，这个时候viewgroup的onTouchEvent就会执行

事实上，一次完整的Touch事件，应该是由一个Down、一个Up和若干个Move组成的
但是dispatchTouchEvent只是分发了Down事件，只有返回true的时候，证明这个view需要 这个事件，然后在继续分发Up和Move事件给它，

ViewGroup还有个onInterceptTouchEvent，看名字便知道这是个拦截事件。这个拦截事件需要分两种情况来说明：

1.假如我们在某个ViewGroup的onInterceptTouchEvent中，将Action为Down的Touch事件返回true，那便表示将该ViewGroup的所有下发操作拦截掉，这种情况下，mTarget会一直为null，因为mTarget是在Down事件中赋值的。由于mTarge为null，该ViewGroup的onTouchEvent事件被执行。这种情况下可以把这个ViewGroup直接当成View来对待。

2.假如我们在某个ViewGroup的onInterceptTouchEvent中，将Acion为Down的Touch事件都返回false，其他的都返回True，这种情况下，Down事件能正常分发，若子View都返回false，那mTarget还是为空，无影响。若某个子View返回了true，mTarget被赋值了，在Action_Move和Aciton_UP分发到该ViewGroup时，便会给mTarget分发一个Action_Delete的MotionEvent，同时清空mTarget的值，使得接下去的Action_Move(如果上一个操作不是UP)将由ViewGroup的onTouchEvent处理。
<!--more-->
Ontouch的优先级高于onclick,onclick的事件在ontouchevent中，只有在ontouch返回false的时候才会继续执行ontouchevent


##view的渲染机制
基础知识
CPU: 中央处理器,它集成了运算,缓冲,控制等单元,包括绘图功能.CPU将对象处理为多维图形,纹理(Bitmaps、Drawables等都是一起打包到统一的纹理).

GPU:一个类似于CPU的专门用来处理Graphics的处理器, 作用用来帮助加快格栅化操作,当然,也有相应的缓存数据(例如缓存已经光栅化过的bitmap等)机制。

OpenGL ES是手持嵌入式设备的3DAPI,跨平台的、功能完善的2D和3D图形应用程序接口API,有一套固定渲染管线流程. 附相关OpenGL渲染流程资料

DisplayList 在Android把XML布局文件转换成GPU能够识别并绘制的对象。这个操作是在DisplayList的帮助下完成的。DisplayList持有所有将要交给GPU绘制到屏幕上的数据信息。

1,Android系统每隔16ms重新绘制一次activity，也就是说你的app必须在16ms内完成屏幕刷新的所有逻辑操作，这样才能达到60帧/s。而用户一般所看到的卡顿是由于Android的渲染性能造成的。 

2,将复杂的UI转换成用户看得懂的图像并绘制到屏幕上是由格栅化操作完成的
格栅化将字符串，按钮，路径或者形状的一些高级对象拆分到不同的像素上在屏幕上进行显示，这是一个非常耗时的操作而手机里有一块硬件就是为了加速完成这个操作就是GPU（图像处理单元）
渲染过程

CPU在图像绘制之前向GPU输入指令这一过程通过OpenGL-ES 完成
也就是说在屏幕绘制UI对象的时候都需要在CPU中转化成多边形再传递GPU进行格栅化操作
cpu将对象转换为多边形耗时 同样上传到GPU也耗时所以我们要减少对象转换次数以及上传数据的次数,幸运的是OpenGL-ES API允许数据上传到GPU进行数据保存，当下一次绘制按钮的时候只要在GPU的存储器里引用它 所以渲染性能的优化就是尽快的上传数据到GPU尽可能长的在不修改数据的条件下保存数据

##动画原理 

首先看下view的源码
```
  public void startAnimation(Animation animation) {
        animation.setStartTime(Animation.START_ON_FIRST_FRAME);
        setAnimation(animation);
        invalidateParentCaches();
        invalidate(true);
    }
```
开始一个动画，在setAnimation之后，调用invalidate刷新自己的界面

Android 动画就是通过 ParentView 来不断调整 ChildView 的画布坐标系来实现的，
```
dispatchDraw()   
{   
    ....   
    Animation a = ChildView.getAnimation()   
    Transformation tm = a.getTransformation();   
    Use tm to set ChildView's Canvas;   
    Invalidate();   
    ....   
} 
```
ParentView 在分发绘制的时候，获取childview的动画属性值，不断的进行Invalidate,直到childview绘制完成

##ANR原理
android设计了一种机制，认为一些阻挡它生命周期的返回，不能无限制下去。当满足超时条件，会弹框提醒用户中止或者等待操作
   
系统都设计了哪些ANR： 
1:KeyDispatchTimeout(5 seconds) –主要类型 
按键或触摸事件在特定时间内无响应 
2:BroadcastTimeout(10 seconds) 
BroadcastReceiver 在特定时间内无法处理完成 
3:ServiceTimeout(20 seconds)  
Service 在特定的时间内无法处理完成 
除此之外,还有 ContentProvider,只是一般很少见。

发生场景
耗时的工作（比如数据库操作，I/O，连接网络或者别的有 
可能阻碍 UI 线程的操作）把它放入单独的线程处理. 
比如打开wifi（因为跨进程操作，有可能wifiserver那边处理超时） 
读写文件（操作是个iowait负载较大的行为，很容易anr） 
查询语句（在数据库内容暴增之后，出现严重的性能问题，产生anr） 
SharedPreferences 的commit操作，本身是个等待操作，在我们activity退出时，有时保存当前状态，方便恢复，会使用commit，如果我们也有一个此时在操作，因为这个操作是有个锁，引起anr 
list的排序。（算法的质量，以及当列表数目激增后，是否能快速算完，是个耗时操作，会产生anr） 
bitmap的运算，（旋转，特效处理等） 
ThreadPoolExecutor 线程池，当我们从这里获取一个线程时候，如果此时所有线程都被使用，就只能迫使等待，此时会出现anr

##oom是否可以try catch
oom属于错误不属于异常 不能try catch(个人观点)

##Android有多个资源文件夹，应用在不同分辨率下是如何查找对应文件夹下的资源的，描述整个过程
| dpi范围|密度 |
| ------------- |:-------------|
| 0dpi~120dpi | ldpi| 
| 120dpi~160dpi | mdpi| 
| 160dpi~240dpi | hdpi |
| 240dpi~320dpi | xhdpi |
| 320dpi~480dpi | xxhdpi |
| 480dpi~640dpi | xxxhdpi |

dpi= context.getResources().getDisplayMetrics().densityDpi
根据手机不同的dpi安卓会对应加载不同的资源文件

##java四种引用
###强引用
平时使用最多的
比如下面
Object object =new Object();
String str ="hello";
 强引用有引用变量指向时永远不会被垃圾回收，JVM宁愿抛出OutOfMemory错误也不会回收这种对象。
 
###软引用
如果一个对象具有软引用，内存空间足够，垃圾回收器就不会回收它；
如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。
软引用可用来实现内存敏感的高速缓存,比如网页缓存、图片缓存等。使用软引用能防止内存泄露，增强程序的健壮性。   
SoftReference的特点是它的一个实例保存对一个Java对象的软引用， 该软引用的存在不妨碍垃圾收集线程对该Java对象的回收。
也就是说，一旦SoftReference保存了对一个Java对象的软引用后，在垃圾线程对 这个Java对象回收前，SoftReference类所提供的get()方法返回Java对象的强引用。
另外，一旦垃圾线程回收该Java对象之 后，get()方法将返回null。


###弱引用
　弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。在java中，用java.lang.ref.WeakReference类来表示。
　
弱引用关联的对象只有弱引用与之关联的时候会被系统回收，如果存在强引用同时与之关联，则进行垃圾回收时也不会回收该对象（软引用也是如此）。
比如
```
 People people=new People("zhouqian",20);  
        WeakReference<People>reference=new WeakReference<People>(people);
```
WeakReference弱引用于强引用people关联，这个时候不会被系统回收

###虚引用
虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。在java中用java.lang.ref.PhantomReference类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。
　　要注意的是，虚引用必须和引用队列关联使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之 关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

##如何保证线程安全
基本上所有的并发模式在解决线程安全问题上，都采用“序列化访问临界资源”的方案，即在同一时刻，只能有一个线程访问临界资源，也称同步互斥访问。
通常来说，是在访问临界资源的代码前面加上一个锁，当访问完临界资源后释放锁，让其他线程继续访问。
在Java中，提供了两种方式来实现同步互斥访问：synchronized和Lock。

非线程安全是指多线程操作同一个对象可能会出现问题。而线程安全则是多线程操作同一个对象不会有问题。

线程安全必须要使用很多synchronized关键字来同步控制，所以必然会导致性能的降低。

所以在使用的时候，如果是多个线程操作同一个对象，那么使用线程安全的Vector；否则，就使用效率更高的ArrayList。