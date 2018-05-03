---
title: retrofit2+rxjava2封装
date: 2018-04-26 17:48:14
---
最近看了很多的资料和retrofit2和rxjava2的封装，有一些感悟，所以写下来
这篇博客不讲retrofit2和rxjava2的用法，只讲封装
主要讲解
1，对后台返回的数据进行过滤，只拿data的数据
2，添加公用的请求头，和公共参数
3，添加log日志，对url,method,data打印log
首先看下最终的用法
```
   RetrofitManager.getInstance().getTest().subscribe(new BaseObserver<CateBean>() {
            @Override
            public void onHandleSuccess(CateBean cateBean) {

            }




        });
```
用法还是很简单的，真正的数据bean会被basedata包装后过滤返回,

大家都知道，一般后台返回的基础数据是
```
 private String err_msg;
  private int code;
    private T data;
```
<!--more-->
但是其实 我们只需要data的数据，所以在得到这个数据之前我们可以过滤一下，首先看下BaseObserver这个类
```
public abstract class BaseObserver<T> implements Observer<BaseData<T>> {
    //mDisposable.dispose();开光，用来切断连接
    private Disposable mDisposable;
    private final int SUCCESS_CODE = 0;
    @Override
    public void onSubscribe(Disposable d) {
        mDisposable = d;
    }

    @Override
    public void onNext(BaseData<T> value) {
        if (value.getCode() == SUCCESS_CODE) {
            T t = value.getData();
            onHandleSuccess(t);
        } else {
            onHandleError(value.getCode(), value.getErr_msg());
        }
    }

    @Override
    public void onError(Throwable e) {
			Toast.makeText(MyApplication.getmContext(),e.toString(), Toast.LENGTH_LONG).show();
    }

    @Override
    public void onComplete() {

    }

   public abstract void onHandleSuccess(T t);

    void onHandleError(int code, String message) {
	Toast.makeText(MyApplication.getmContext(),message , Toast.LENGTH_LONG).show();
    }
}
```
Observer观察者，被观察者将数据发给观察者，我们重写这个类，将得到 的数据进行判断
如果value.getCode() == SUCCESS_CODE
代表成功，这里我的SUCCESS_CODE=0 可以根据后台规定自行修改
 T t = value.getData();
  onHandleSuccess(t);
  对外暴露一个接口，把过滤后的数据返回回去，这样就能直接得到我们需要的数据了
  上面还有一个basedata类
  每个公司的后台数据有所不同，可以根据情况修改
```
  public class BaseData<T> {

    /**
     * server_time : 244748
     * time : 1492579031
     * logid : 149257903154183870
     * code : 0
     * err_msg : success
     * data : ob
     */
    private String err_msg;
    private int server_time;
    private int time;
    private long logid;
    private int code;
    private T data;

    public int getServer_time() {
        return server_time;
    }

    public void setServer_time(int server_time) {
        this.server_time = server_time;
    }

    public int getTime() {
        return time;
    }

    public void setTime(int time) {
        this.time = time;
    }

    public long getLogid() {
        return logid;
    }

    public void setLogid(long logid) {
        this.logid = logid;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getErr_msg() {
        return err_msg;
    }

    public void setErr_msg(String err_msg) {
        this.err_msg = err_msg;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}

```
接下来看下retromanager里面做了什么
```
 private RetrofitManager() {
        gson = new GsonBuilder()
                .setLenient()
                .create();

        OkHttpClient client = new OkHttpClient
                .Builder()
                .addInterceptor(addQueryParameterInterceptor())  //添加参数
                .addInterceptor(addHeaderInterceptor()) // 添加头部信息
                .addInterceptor(new LogInterceptor().setLevel(LogInterceptor.Level.BODY))
                .cache(cache)  //添加缓存
                .connectTimeout(60l, TimeUnit.SECONDS)
//                .addNetworkInterceptor(new LogInterceptor())
                .readTimeout(60l, TimeUnit.SECONDS)
                .writeTimeout(60l, TimeUnit.SECONDS)
                .build();

        retrofit = new Retrofit.Builder()
                .baseUrl(baseurl)
                .client(client)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
        mApiService = retrofit.create(ApiService.class);
    }
```
首先看实例化，
我把一些公共的，重复的代码都写在这里面
同时对OkHttpClient进行了处理
一个一个的来
首先看这句话
```
 .addInterceptor(addQueryParameterInterceptor())
```
添加参数,看下addQueryParameterInterceptor方法
```
    /**
     * 设置公共参数
     */
    private static Interceptor addQueryParameterInterceptor() {
        Interceptor addQueryParameterInterceptor = new Interceptor() {
            @Override
            public okhttp3.Response intercept(Chain chain) throws IOException {
                Request originalRequest = chain.request();
                Request request;
                HttpUrl modifiedUrl = originalRequest.url().newBuilder()
                        .addQueryParameter("params1", "")
                        .addQueryParameter("params2", "")
                        .build();
                request = originalRequest.newBuilder().url(modifiedUrl).build();
                return chain.proceed(request);
            }
        };
        return addQueryParameterInterceptor;
    }
```
我们可以把一些需要添加的公共参数写在这里面
比如设置token,设备id等等
```
.addQueryParameter("token", (String) SpUtils.get("token", ""))                    .addQueryParameter("devid",AppUtil.getDeviceId(MyApplication.getmContext()))
```

接下来
```
.addInterceptor(addHeaderInterceptor())

```
看下addHeaderInterceptor
```
    /**
     * 设置头
     */
    private static Interceptor addHeaderInterceptor() {
        Interceptor headerInterceptor = new Interceptor() {
            @Override
            public okhttp3.Response intercept(Chain chain) throws IOException {
                Request originalRequest = chain.request();
                Request.Builder requestBuilder = originalRequest.newBuilder()
                        .header("Accept", "application/json")
                        .method(originalRequest.method(), originalRequest.body());
                Request request = requestBuilder.build();
                return chain.proceed(request);
            }
        };
        return headerInterceptor;
    }
```
这里主要是添加头部信息，比如设置accept为application/json格式
接下来
```
 .addInterceptor(new LogInterceptor().setLevel(LogInterceptor.Level.BODY))
```
这句话的意思是添加打印信息，因为有的时候我们请求了一个url，需要能在log里面把它打印出来一些信息，比如这个url的请求方式，得到的数据，这个URL的地址等等
因为公司后台是php的所以我重写了一下，对最后的数据进行了unicode转换
如果是java后台，直接使用HttpLoggingInterceptor，就会有日志了
首先导入log需要的包
```
compile 'com.squareup.okhttp3:logging-interceptor:3.5.0'
```

然后
```
 .addInterceptor(new HttpLoggingInterceptor().setLevel(HttpLoggingInterceptor.Level.BODY))
```
我只是copy了一份一样的代码，对最后的结果进行了unicode转换，关键代码如下
```
 LogUtils.d("swt",convert(buffer.clone().readString(charset)));
 public String convert(String utfString){
        StringBuilder sb = new StringBuilder();
        int i = -1;
        int pos = 0;

        while((i=utfString.indexOf("\\u", pos)) != -1){
            sb.append(utfString.substring(pos, i));
            if(i+5 < utfString.length()){
                pos = i+6;
                sb.append((char)Integer.parseInt(utfString.substring(i+2, i+6), 16));
            }
        }

        return sb.toString();
    }
```
打印的东西还是很全面的
其他的几个根据字面意思就能明白就不讲了
接下来还有一个ApiService
```
/**
 * 介绍：所有的接口都写到这里
 * 作者：sweet
 * 时间: 2017/4/19
 */
public interface ApiService {


    /**
     *测试类
     */
    @GET("写上你的后缀url")
    Observable<BaseData<CateBean>> getTest();

}
```
这里返回的是Observable，所以在 .addCallAdapterFactory(RxJava2CallAdapterFactory.create())这里别忘记添加rxjava适配，
代码在文末，在用的时候有几个需要自行修改
  1，baseurl 
  2, apiserver的url
  3, data过滤的条件我写的是getcode==0,
  4, log日志如果是java后台，直接使用HttpLoggingInterceptor
	
同时需要注意在BaseData后面接收的是泛型
如果有什么问题 欢迎评论
代码地址http://download.csdn.net/detail/qq_15527709/9825850


