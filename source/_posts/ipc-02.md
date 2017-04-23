---
title: Android的IPC机制(二)—AIDL实现原理简析
date: 2016-02-20 20:10
category: [Android]
tags: [ipc,aidl]
comments: true
---
　　上篇说到AIDL的使用方法，我们不能仅仅只是满足对AIDL的使用，那么对于AIDL到底是如何实现的呢？为什么我们只是创建一个AIDL文件，系统就会为我们自动生成一个Java文件，那么这个Java文件里面到底包含了哪些内容呢？我们今天就来研究一下。<!--more-->
# **AIDL实现原理**
　　在这里我们首先看一下AIDL是怎么实现的。当我们创建一个Service和一个AIDL接口的时候，然后创建一个Binder对象并在Service中的onBind方法中去返回这个Binder对象到客户端，客户端得到这个对象后就可以绑定服务端的Service，并且与服务端建立连接后就可以访问服务端的方法了。所以在整个AIDL的实现过程中这个Binder是关键。那么这个Binder究竟是什么呢？在这里简要说明一下。
　　Binder是一种进程间的通信方式。Binder在Linux 内核中是一个驱动程序(/dev/binder)，ServiceManager通过这个Binder驱动连接各种Manager(AvtivityManager,WindowManager等)。在Android的应用层中通过Binder实现Android的RPC（Remote Procedure Call 远程进程调用）过程。
　　以上篇文章中的加法运算为例分析来一下Binder在应用层的实现机制，我们可以看到在服务端通过new ICalculate.Stub()来创建一个Binder对象，那么我们找到ICalculate的Java代码。 也就是系统根据我们的AIDL接口自动生成Java文件。
```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: G:\\Android\\androidstudioProject\\aidl\\AIDLClient\\app\\src\\main\\aidl\\com\\ljd\\aidl\\ICalculate.aidl
 */
package com.ljd.aidl;
// Declare any non-default types here with import statements

public interface ICalculate extends android.os.IInterface {
/** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements com.ljd.aidl.ICalculate {
        private static final java.lang.String DESCRIPTOR = "com.ljd.aidl.ICalculate";
         /** Construct the stub at attach it to the interface. */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        /**
         * Cast an IBinder object into an com.ljd.aidl.ICalculate interface,
         * generating a proxy if needed.
         */
        public static com.ljd.aidl.ICalculate asInterface(android.os.IBinder obj) {
            if ((obj==null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin!=null)&&(iin instanceof com.ljd.aidl.ICalculate))) {
                return ((com.ljd.aidl.ICalculate)iin);
            }
            return new com.ljd.aidl.ICalculate.Stub.Proxy(obj);
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

        private static class Proxy implements com.ljd.aidl.ICalculate {
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
            public int add(int first, int second) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(first);
                    _data.writeInt(second);
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
    public int add(int first, int second) throws android.os.RemoteException;
}
```
　　其实这段代码很有规律可寻的。能够看出ICalculate 是一个继承IInterface的接口，IInterface中只声明了一个asBinder接口，用于返回Binder对象。所有在Binder传输的接口都必须继承这个IInterface接口。在ICalculate的java接口中再次声明了AIDL中的方法并且创建了一个继承Binder的内部抽象类Stub。也就是说这个Stub也就是一个Binder。
　　在服务端我们通过new ICalculate.Stub()来创建一个Binder对象，我们来看一下这个Stub里面的内容。Stub内定义了一个DESCRIPTOR作为Binder的唯一标识，定义了一个TRANSACTION_add 作为接口方法的id。下面看一下asInterface()方法。

```java
public static com.ljd.binder.ICalculate asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof com.ljd.binder.ICalculate))) {
        return ((com.ljd.binder.ICalculate) iin);
    }
    return new com.ljd.binder.ICalculate.Stub.Proxy(obj);
}
```
　　这个方法是在客户端被调用，运行在主线程当中，是将服务端返回来的Binder对象转为客户端所需要的接口对象。通过Binder对象的queryLocalInterface方法去查找客户端进程中是否存在所需要的接口对象。存在的话就直接返回这个接口对象。也就是说这时候客户端和服务端在同一个进程内。若是不存在，就创建一个Proxy对象。Proxy是Stub的一个内部代理类。我们看一下Proxy里面到底做了什么。

```java
private static class Proxy implements com.ljd.aidl.ICalculate {
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
    public int add(int first, int second) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        int _result;
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeInt(first);
            _data.writeInt(second);
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
```
　　在跨进程通信中，由于是在asInterface方法中创建了一个Proxy对象，所以，Proxy也是运行在客户端的主线程中。在Proxy类中实现了ICalculate接口，定义了IBinder接口对象。通过实现asBinder接口返回这个IBinder接口对象。而对于我们自己定义的add接口方法中创建两个Parcel对象_data和_reply （Parcel可以说是一个存放读取数据的容器，在Parcel内部包装了可序列化的数据，并且可以在Binder中自由的传输)。我们可以很清晰看出_data对象写入了我们传入的参数，而_reply 则是用来存放方法的返回值。紧接着调用了Binder的transact方法。在transact方法方法中将方法id,_data,_reply作为参数传入进去。下面看一下Binder的transact方法。

```java
public final boolean transact(int code, Parcel data, Parcel reply,
        int flags) throws RemoteException {
    if (false) Log.v("Binder", "Transact: " + code + " to " + this);
    if (data != null) {
        data.setDataPosition(0);
    }
    boolean r = onTransact(code, data, reply, flags);
    if (reply != null) {
        reply.setDataPosition(0);
    }
    return r;
}
```
　　我们可以很明显的看到在transact方法中调用了onTransact方法并返回了一个boolean类型。当返回false的时候客户端请求就会失败。在Stub中我们重写onTransact方法。下面我们看一下onTransact方法。
　　
```java
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
```
　　在onTransact方法中我们会根据code来判断所需要执行的方法，通过data取出所需要的参数，然后执行该方法，若是有返回值则将将返回值写入reply中。在这里Parcel对象data和reply读写顺序要严格保持一致。
　　可以看出Binder运行的整个流程也就是：当客户端绑定服务端发起远程请求，客户端进程处于休眠，当前线程处于挂起状态。然后服务端开始执行，执行完毕后将结果返回给客户端。然后客户端继续执行。
　　这也说明了当远程方法是一个很耗时的操作时，我们不应该在主线程中发起请求。而服务端的Binder方法在Binder线程池中运行，所以Binder不论是否耗时都不应该重新为他在开启一个线程。
　　到这里AIDL中Binder的工作机制已经分析完了。现在我们可以发现完全不用AIDL文件也能够实现跨进程的方法调用。那么我们自己来写一个Binder去实现服务端与客户端的跨进程的方法调用。
## **自定义Binder的实现**##
### **实现过程**###
　　在这里我们创建一个继承IInterfaceJava接口，在接口里面我们去声明算术的加减法运算。
```java
package com.ljd.binder.custombinder;

import android.os.IInterface;

public interface ICalculate extends IInterface {

    static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_sub = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);

    public int add(int first, int second) throws android.os.RemoteException;

    public int sub(int first, int second) throws android.os.RemoteException;
}
```
　　然后我们再去实现这个接口。
```java
package com.ljd.binder.custombinder;

import android.os.Binder;
import android.os.IBinder;
import android.os.IInterface;
import android.os.Parcel;
import android.os.RemoteException;
import android.util.Log;

/**
 * Created by ljd-PC on 2016/2/21.
 */
public class Calculate extends Binder implements ICalculate{

    private final String TAG = "Calculate";
    private static final String DESCRIPTOR = "com.ljd.binder.ICalculate";

    @Override
    public void attachInterface(IInterface owner, String descriptor) {
        this.attachInterface(this, DESCRIPTOR);
    }
    public static ICalculate asInterface(IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof ICalculate))) {
            return (ICalculate) iin;
        }
        return new Proxy(obj);
    }

    @Override
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
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
            case TRANSACTION_sub: {
                data.enforceInterface(DESCRIPTOR);
                int _arg0;
                _arg0 = data.readInt();
                int _arg1;
                _arg1 = data.readInt();
                int _result = this.sub(_arg0, _arg1);
                reply.writeNoException();
                reply.writeInt(_result);
                return true;
            }
        }
        return super.onTransact(code, data, reply, flags);
    }

    @Override
    public int add(int first, int second) throws RemoteException {
        Log.e(TAG,String.valueOf(first+second));
        return first + second;
    }

    @Override
    public int sub(int first, int second) throws RemoteException {
        return first - second;
    }

    @Override
    public IBinder asBinder() {
        return null;
    }

    private static class Proxy implements ICalculate {
        private IBinder mRemote;

        Proxy(IBinder remote) {
            mRemote = remote;
        }

        @Override
        public IBinder asBinder() {
            return mRemote;
        }

        public String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }

        @Override
        public int add(int first, int second) throws RemoteException {
            Parcel _data = Parcel.obtain();
            Parcel _reply = Parcel.obtain();
            int _result;
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                _data.writeInt(first);
                _data.writeInt(second);
                mRemote.transact(TRANSACTION_add, _data, _reply, 0);
                _reply.readException();
                _result = _reply.readInt();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
            return _result;
        }

        @Override
        public int sub(int first, int second) throws RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            int _result;
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                _data.writeInt(first);
                _data.writeInt(second);
                mRemote.transact(TRANSACTION_sub, _data, _reply, 0);
                _reply.readException();
                _result = _reply.readInt();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
            return _result;
        }
    }
}

```
　　在这里创建一个Service。

```java
package com.ljd.binder.custombinder;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;


public class CalculateService extends Service {
    private Calculate mCalculate;
    public CalculateService() {
        mCalculate = new Calculate();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mCalculate;
    }
}
```
　　在上篇文章中说过我们可以通过android:process	属性为服务端开启一个进程。

```java
        <service
            android:name=".custombinder.CalculateService"
            android:process=":remote">
            <intent-filter>
                <action android:name="com.ljd.binder.CUSTOM_BINDER"></action>
                <category android:name="android.intent.category.DEFAULT"></category>
            </intent-filter>
        </service>
```
　　其中:remote是一种简写法，全称为com.ljd.binder.custombinder:remote。以:开头的进程是属于当前应用的私有进程，其他应用的组件不能和他运行在同一个进程中。
　　下面是客户端代码。

```java
package com.ljd.binder.custombinder;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.IBinder;
import android.os.RemoteException;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Toast;

import com.ljd.binder.*;

import butterknife.ButterKnife;
import butterknife.OnClick;

public class CustomBinderActivity extends AppCompatActivity {


    private final String TAG = "CustomBinderActivity";
    private ICalculate mCalculate;
    private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            if(mCalculate != null){
                mCalculate.asBinder().unlinkToDeath(mDeathRecipient,0);
                mCalculate = null;
                bindService();
            }
        }
    };
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.e(TAG,"Bind Success");
            mCalculate = Calculate.asInterface(service);
            try {
                mCalculate.asBinder().linkToDeath(mDeathRecipient,0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mCalculate = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_custom_binder);
        ButterKnife.bind(this);
        bindService();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        ButterKnife.unbind(this);
        unbindService(mConnection);
    }

    @OnClick({R.id.custom_add_btn,R.id.custom_sub_btn})
    public void onClickButton(View v){
        if (mCalculate == null){
            return;
        }
        switch (v.getId()){
            case R.id.custom_add_btn:
                try {
                    Toast.makeText(this,String.valueOf(mCalculate.add(6,2)),Toast.LENGTH_SHORT).show();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
            case R.id.custom_sub_btn:
                try {
                    Toast.makeText(this,String.valueOf(mCalculate.sub(6,2)),Toast.LENGTH_SHORT).show();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
            default:
                break;
        }
    }

    private void bindService(){
        Intent intent = new Intent("com.ljd.binder.CUSTOM_BINDER");
        bindService(intent,mConnection, Context.BIND_AUTO_CREATE);
    }

}

```
　　
### **效果演示**
![这里写图片描述](http://img.blog.csdn.net/20160221215127939)
# **总结**
　　我们现在自己已经实现了跨进程间的调用。而且我们构建的这个Binder几乎和系统生成的一模一样。所以AIDL就是对Binder进行了一次封装，并且能够支持多线程并发访问。通过AIDL的使用能够大大简化了我们开发过程，节约了我们的开发时间。

# **[源码下载](https://github.com/lijiangdong/binder-example)**