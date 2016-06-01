---
layout:     post
title:      "RxJava与Retrofit"
subtitle:   ""
date:       2016-01-09
author:     "goIntoAction"
header-img: "img/post-bg.jpg"
tags:
    - android
---
2015年至今，Android界什么开源库最火，在我看来是RxJava和Retrofit。
RxJava 是一个响应式编程框架，采用观察者设计模式，其实这模式之前已经被微软在.net上实现了，NextFlix按照这种思想将其移植到基于jvm的语言上，在RxJava中有四个重要的概念，分别是Observable (可观察者)、 Observer (观察者)、 subscribe (订阅)、事件。Observer通过subscribe()订阅Observable，Observable会发出事件触发Observer。其中Observer有三个方法提供给Observable触发：
+ onNext() 每发出一个事件的时候触发
+ onCompleted() 事件队列完成时触发。
+ onError() 发生错误的时候触发

Retrofit是一套RESTful网络框架，是由Android界鼎鼎有名的Jake Wharton主导开发的，使用它访问网络非常方便。在Retrofit中你只要定义一个普通的接口，加上相应的注解，就能被转换为一个网络接口。

这两个库非常好用，而且有点感觉他们是天生的一对，将Retrofit的网络请求结果封装成一个Observable，然后等它访问服务器后触发我们的Observer，想起来就很酷，下面就来看应该怎么做吧。

首先要使用他们，我们应该在build.gradle的dependencies加上这些依赖

    compile 'io.reactivex:rxjava:1.1.0'
    compile 'io.reactivex:rxandroid:1.1.0'
    compile 'com.squareup.okhttp3:okhttp:3.2.0'
    compile 'com.squareup.retrofit2:retrofit:2.0.0'
    compile 'com.squareup.retrofit2:converter-gson:2.0.0'
    compile 'com.squareup.retrofit2:adapter-rxjava:2.0.0'

接着根据接口返回值定义两个用来存放请求结果的类。用SerializedName来指定属性与json的匹配。

	public class GankBeautyResult {
	    public boolean error;
	    public @SerializedName("results") List<GankBeauty> beauties;
	}
	public class GankBeauty {
	    public String createdAt;
	    public String url;
	}
	
然后创建一个接口，作为Retrofit的网络接口

	public interface GankApi {
	    @GET("data/福利/{number}/{page}")
	    Observable<GankBeautyResult> getBeauties(@Path("number") int number, @Path("page") int page);
	}
	
这里GET和PATH是Retrofit的注解，GET表示使用get请求，括号里表示接口的路径，服务器的地址之后会设置。PATH表示这两个参数会被替换到路径中。
接着，我们要在Retrofit的帮助下生成一个接口实例对象
	
	CallAdapter.Factory rxJavaCallAdapterFactory = RxJavaCallAdapterFactory.create();
	Converter.Factory gsonConverterFactory = GsonConverterFactory.create();
	Retrofit retrofit = new Retrofit.Builder()
	                    .client(okHttpClient)
	                    .baseUrl("http://gank.io/api/")
	                    .addConverterFactory(gsonConverterFactory)
	                    .addCallAdapterFactory(rxJavaCallAdapterFactory)
	                    .build();
	GankApi gankApi = retrofit.create(GankApi.class);
	
因为我们要与RxJava一起使用，所以在前面需要引入Retrofit提供的adapter-rxjava，并指定适配器模式为RxJavaCallAdapterFactory。client指定okhttp作为访问网络的库，baseUrl就是指定我们的服务器地址，addConverterFactory是指定用gson库转换结果。最后使用Retrofit创建出接口对象GankApi，我们就可以使用此对象来获取网络数据并观察结果。

	Observer<GankBeautyResult> gankObserver = new Observer<GankBeautyResult>() {
	            @Override
	            public void onCompleted() {
	
	            }
	
	            @Override
	            public void onError(Throwable e) {
	                Toast.makeText(getActivity(), R.string.loading_failed, Toast.LENGTH_SHORT).show();
	            }
	
	            @Override
	            public void onNext(GankBeautyResult gankBeautyResult) {
	                List<GankBeauty> gankBeautyList = gankBeautyResult.beauties;
	                GankListAdapter adapter = new GankListAdapter();
	                adapter.setData(gankBeautyList);
	
	            }
	        };
	        
	        Network.getGankApi().getBeauties(1,1)
	                .subscribeOn(Schedulers.io())
	                .observeOn(AndroidSchedulers.mainThread())
	                .subscribe(gankObserver);
在使用的时候，我们先创建一个Observer对象，泛型写GankBeautyResult，表示要在触发onNext方法时候传入网络获取封装成GankBeautyResult类型数据，然后把数据设置给RecycleView的Adapter。
而下面subscribeOn(Schedulers.io())表示访问网络比较耗时，不能在主线程执行，要使用io调度器。observeOn(AndroidSchedulers.mainThread())表示onNext涉及UI操作，需要在Android主线程执行。最后就是使用gankObserver观察网络请求结果。

这是最简单的一个RxJava与Retrofit合用的例子。