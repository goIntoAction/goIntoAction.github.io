---
layout:     post
title:      "android binder机制"
subtitle:   ""
date:       2016-05-01
author:     "goIntoAction"
header-img: "img/post-bg-2015.jpg"
tags:
    - android源码分析
---


> 下滑这里查看更多内容

binder是粘合剂的英文，是Android系统进程间通信（IPC）方式之一。在android系统中，它就是各组件，应用之间的粘合剂。android基本上的跨进程通讯都是使用了binder。

######为什么要使用Binder呢？主要有两方面的原因。

1.进程隔离
对于进程隔离，维基百科上是这样说的（词条：[进程隔离](https://zh.wikipedia.org/zh/%E8%BF%9B%E7%A8%8B%E9%9A%94%E7%A6%BB)）

    进程隔离是为保护操作系统中进程互不干扰而设计的一组不同硬件和软件的技术。这个技术是为了避免进程A写入进程B的情况发生。 进程的隔离实现，使用了虚拟地址空间。进程A的虚拟地址和进程B的虚拟地址不同，这样就防止进程A将数据信息写入进程B。

但是进程不可避免需要进行通信，所以系统需要提供可靠的进程间通信技术。

2.访问内核空间
在Linux中，内核是安全性要求非常，不能让应用随意访问，所以Linux中使用了内核空间与应用层隔离这种机制来保护内核，应用想访问内核空间，需要通过调用系统服务提供的接口来实现。而系统服务与应用肯定不是在同一进程，所以需要进程间通信。

3.android系统的架构与传输效率
android系统中大量使用client-server这种架构，而Linux中原有的进程间通信有管道、system V IPC、socket，原生支持c/s的只有socket。但socket是开销大，传输效率慢，是跨网络的传输方式，不适合用在此处。

4.安全性
传统的跨进程机制安全行不高，android中每个应用都分配了自己的UID，这是重要的识别标识，但传统机制中，UID是可以随便填入数据包的，像socket的ip地址是可以伪造的，管道的名称、system V的键值都是开放给用户的。

所以android选用了开源的OpenBinder作为android的IPC。

Binder的机制因为不符合之前Linux的理念，所以它并没有被加入Linux内核中，但它又需要访问内核，所以是以驱动模块的形式加载到Linux内核空间中的，它提供了“硬件”接口用于进程通信。通信的模型如下：

![binder-model.png](/img/in-post/binder/binder-model.png)

在android系统启动时，ServicesManager会被启动，之后它会启动其他系统服务并注册，而我们自己在manifest文件中声明的服务也会被注册到ServicesManager中，它记录了所有的服务的信息。类比打电话，ServicesManager就像通讯录一样，当我们要打电话时，会先去它那里查询。知道电话号码后就开始打电话，那信号如何发出去？是通过基站，binder在这里就像基站一样，为我们传输数据。

模型知道了，那这个过程具体是怎么实现的。首先我们要知道binder跟其他机制有个明显的不同，它设计的目的就是用于进程之间来传递对象，面向对象是其本质之一，所以在binder中，我们是使用代理对象来通信的。在java层binder机制的现实涉及到三个类。

* IBinder是一个接口，它代表了一种跨进程传输的能力；只要实现了这个接口，就能将这个对象进行跨进程传递；这是驱动底层支持的；在跨进程数据流经驱动的时候，驱动会识别IBinder类型的数据，从而自动完成不同进程Binder本地对象以及Binder代理对象的转换。java层是使用onTransact来读取/写入数据。

* IInterface代表的就是远程server对象具有什么业务能力。具体来说，就是aidl里面的接口。

* Java层的Binder类，代表的其实就是Binder本地对象。BinderProxy类是Binder类的一个内部类，它代表远程进程的Binder对象的本地代理；这两个类都继承自IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder驱动会自动完成这两个对象的转换。

![2016binder-procedure.png](/img/in-post/binder/binder-procedure.png)

这样说可能太抽象了，我们直接看代码。例如我们要创建一个加法的服务，首先是在服务端应用编写aidl文件

    // ICompute.aidl
    package com.example.test.app;
    interface ICompute {
     int add(int a, int b);
    }

然后rebuild，编译工具会为我们生成ICompute类，ICompute继承了IInterface，规范了业务接口。
	package com.example.test.app;

	public interface ICompute extends android.os.IInterface {
        /**
         * Local-side IPC implementation stub class.
         */
        public static abstract class Stub extends android.os.Binder implements com.example.test.app.ICompute {
            private static final java.lang.String DESCRIPTOR = "com.example.test.app.ICompute";

            /**
             * Construct the stub at attach it to the interface.
             */
            public Stub() {
                this.attachInterface(this, DESCRIPTOR);
            }

            /**
             * Cast an IBinder object into an com.example.test.app.ICompute interface,
             * generating a proxy if needed.
             */
            public static com.example.test.app.ICompute asInterface(android.os.IBinder obj) {
                if ((obj == null)) {
                    return null;
                }
                android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
                if (((iin != null) && (iin instanceof com.example.test.app.ICompute))) {
                    return ((com.example.test.app.ICompute) iin);
                }
                return new com.example.test.app.ICompute.Stub.Proxy(obj);
            }

            @Override
            public android.os.IBinder asBinder() {
                return this;
            }

            @Override
            public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
                switch (code) {
                    case INTERFACE_TRANSACTION: {
                        reply.writeString(DESCRIPTOR);
                        return true;
                    }
                    case TRANSACTION_add: {
                        data.enforceInterface(DESCRIPTOR);
                        int _arg0;
                        _arg0 = data.readInt();
                        int _arg1;
                        _arg1 = data.readInt();
                        int _result = this.add(_arg0, _arg1);
                        reply.writeNoException();
                        reply.writeInt(_result);
                        return true;
                    }
                }
                return super.onTransact(code, data, reply, flags);
            }

            private static class Proxy implements com.example.test.app.ICompute {
                private android.os.IBinder mRemote;

                Proxy(android.os.IBinder remote) {
                    mRemote = remote;
                }

                @Override
                public android.os.IBinder asBinder() {
                    return mRemote;
                }

                public java.lang.String getInterfaceDescriptor() {
                    return DESCRIPTOR;
                }

                /**
                 * Demonstrates some basic types that you can use as parameters
                 * and return values in AIDL.
                 */
                @Override
                public int add(int a, int b) throws android.os.RemoteException {
                    android.os.Parcel _data = android.os.Parcel.obtain();
                    android.os.Parcel _reply = android.os.Parcel.obtain();
                    int _result;
                    try {
                        _data.writeInterfaceToken(DESCRIPTOR);
                        _data.writeInt(a);
                        _data.writeInt(b);
                        mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
                        _reply.readException();
                        _result = _reply.readInt();
                    } finally {
                        _reply.recycle();
                        _data.recycle();
                    }
                    return _result;
                }
            }

            static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        }

        /**
         * Demonstrates some basic types that you can use as parameters
         * and return values in AIDL.
         */
        public int add(int a, int b) throws android.os.RemoteException;
	}


接着，我们新建一个ComputeService类，并在onBind里将ICompute.Stub的实现类对象返回。

	private IBinder mBinder = new ComputeBinder();
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }


    class ComputeBinder extends ICompute.Stub {

        @Override
        public int add(int a, int b) throws RemoteException {
            return a + b;
        }
    }

这里讲下Stub的作用，Stub继承了IBinder实现了ICompute，所以具有跨进程通信能力，又有处理业务能力。在跨进程调用中，我们调用的方法是使用编号，所以Stub的onTransact方法中使用了int常量来判断我们调用那个方法，然后读取参数，调用业务方法。add是抽象方法，具体实现stub不管。

	case TRANSACTION_add: {
        data.enforceInterface(DESCRIPTOR);
        int _arg0;
        _arg0 = data.readInt();
        int _arg1;
        _arg1 = data.readInt();
        int _result = this.add(_arg0, _arg1);//调用add
        reply.writeNoException();
        reply.writeInt(_result);
        return true;
    }

将服务端的aidl拷贝到客户端并rebuild生成ICompute类。在Bind Service的时候，我们一般是这样使用

	bindService(serviceIntent,mConn,BIND_AUTO_CREATE);//mConn是ServiceConnection对象
然后在ServiceConnection.onServiceConnected中

	ICompute computeService = ICompute.Stub.asInterface(service);
    computeService.add(1,2);

我们再来分析下整个流程，首先调用bindService的时候，会通过binder，调用到ComputeService的onBind获取一个IBinder对象。然后系统会回调ServiceConnection的onServiceConnected,将此IBinder传给调用方。我们需要在回调方法中调用ICompute.Stub.asInterface，将IBinder转换为服务接口。
这个asInterface方法很重要,我们来看它做了什么处理

	public static com.example.test.app.ICompute asInterface(android.os.IBinder obj) {
                if ((obj == null)) {
                    return null;
                }
                android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
                if (((iin != null) && (iin instanceof com.example.test.app.ICompute))) {
                    return ((com.example.test.app.ICompute) iin);
                }
                return new com.example.test.app.ICompute.Stub.Proxy(obj);
            }

它会调用binder.queryLocalInterface，尝试获取本地接口，就是说这个服务可能是在本进程中，不是跨进程调用，obj为Binder。然后获取不到，那就说明是远程调用，obj为BinderProxy,需要这个BinderProxy来进行跨进程通信，所以使用这个binder去构造一个远程服务的代理类Stub.Proxy。Proxy为ICompute的实现类，能够用来处理具体业务，但其实也不是自己处理，看它的add方法，其实是调用了刚刚传进入的BinderProxy调用远程服务来处理。

	@Override
    public int add(int a, int b) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        int _result;
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeInt(a);
            _data.writeInt(b);
            mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);//mRemote就是刚刚传入的BinderProxy
            _reply.readException();
            _result = _reply.readInt();
        } finally {
            _reply.recycle();
            _data.recycle();
        }
        return _result;
    }

