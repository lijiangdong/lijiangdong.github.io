---
title:  Android的IPC机制(三)—Binder连接池
date: 2016-02-24 20:00
category: [android]
tags: [ipc,aidl]
comments: true
---
# **综述**
　　前两篇说到AIDL的使用方法，但是当我们的项目很大时，很多模块都需要用到Service，我们总不能为每一个模块都创建一个Service吧，这样一来我们的应用就会显得很笨重。那么有没有一种解决方案叫我们只需要创建一个Service，然后去管理AIDL呢？在任玉刚的《Android开发艺术探索》中给出了一个解决方案，那就是Binder连接池。在这里我们看一下他是怎么实现的。<!--more-->
# **Binder连接池的实现**
　　在前面说到AIDL的使用及原理的时候，我们可以看到在服务端只是创建了一个Binder然后返回给客户端使用而已。于是我们可以想到是不是我们可以只有一个Service,对于不同可客户端我们只是去返回一个不同的Binder即可，这样就避免了创建了大量的Service。在任玉刚的《Android开发艺术探索》给出了一个Binder连接池的概念，很巧妙的避免了Service的多次创建。这个Binder连接池类似于设计模式中的工厂方法模式。为每一个客户端创建他们所需要的Binder对象。那么下面我们看一下它是如何实现的。
　　首先我们创建一个名为BinderPool的AIDL接口。

```java
// IBinderPool.aidl
package com.example.ljd_pc.binderpool;

// Declare any non-default types here with import statements

interface IBinderPool {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
     IBinder queryBinder(int binderCode);
}
```
　　在这个AIDL接口中我们根据传入的binderId返回了一个IBinder接口对象。我们再创建两个AIDL接口作为功能显示。
　　创建名为ICalculate的AIDL接口用作计算算术的加减法。
```java
// ICalculate.aidl
package com.example.ljd_pc.binderpool;

// Declare any non-default types here with import statements

interface ICalculate {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
     int add(int first, int second);
     int sub(int first, int second);
}
```
　　再创建一个名为IRect作为求矩形的面积与周长。
```java
// IRect.aidl
package com.example.ljd_pc.binderpool;

// Declare any non-default types here with import statements

interface IRect {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    int area(int length,int width);
    int perimeter(int length,int width);
}
```
　　然后我们继承系统自动生成的Stub类去实现接口中的方法。下面是ICalculate实现。
```java
package com.example.ljd_pc.binderpool;

import android.os.RemoteException;

public class ICalculateImpl extends ICalculate.Stub {
    @Override
    public int add(int first, int second) throws RemoteException {
        return first + second;
    }

    @Override
    public int sub(int first, int second) throws RemoteException {
        return first - second;
    }
}
```
　　IRect的实现。
```java
package com.example.ljd_pc.binderpool;


import android.os.RemoteException;

public class IRectImpl extends IRect.Stub {
    @Override
    public int area(int length, int width) throws RemoteException {
        return length * width;
    }

    @Override
    public int perimeter(int length, int width) throws RemoteException {
        return length*2 + width*2;
    }
}
```

　　然后我们在看一下这个BinderPool是怎么实现的。
```java
package com.example.ljd_pc.binderpool;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.IBinder;
import android.os.RemoteException;


import java.util.concurrent.CountDownLatch;

public class BinderPool {

    public static final int BINDER_NONE = -1;
    public static final int BINDER_CALCULATE = 0;
    public static final int BINDER_RECT = 1;

    private Context mContext;
    private static IBinderPool mBinderPool;
    private static BinderPool mInstance;
    private CountDownLatch mCountDownLatch;

    private BinderPool(Context context) {
        mContext = context.getApplicationContext();
        connectBinderPoolService();
    }

    public static BinderPool getInstance(Context context){
        if (mBinderPool == null){
            synchronized (BinderPool.class){
                if (mBinderPool == null){
                    mInstance = new BinderPool(context);
                }
            }
        }
        return mInstance;
    }

    private ServiceConnection mBinderPoolConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBinderPool = IBinderPool.Stub.asInterface(service);
            try {
                mBinderPool.asBinder().linkToDeath(mBinderPoolDeathRecipient,0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            mCountDownLatch.countDown();
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    private IBinder.DeathRecipient mBinderPoolDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            mBinderPool.asBinder().unlinkToDeath(mBinderPoolDeathRecipient,0);
            mBinderPool = null;
            connectBinderPoolService();
        }
    };

    private synchronized void connectBinderPoolService(){
        mCountDownLatch = new CountDownLatch(1);
        Intent service = new Intent("com.ljd.binder.BINDER_POOL_SERVICE");
        mContext.bindService(service,mBinderPoolConnection,Context.BIND_AUTO_CREATE);
        try {
            mCountDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public IBinder queryBinder(int binderCode){
        IBinder binder = null;
        if (mBinderPool != null){
            try {
                binder = mBinderPool.queryBinder(binderCode);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        return binder;
    }

    public static class BinderPoolImpl extends IBinderPool.Stub {

        public BinderPoolImpl(){
            super();
        }
        @Override
        public IBinder queryBinder(int binderCode) throws RemoteException {
            IBinder binder = null;
            switch (binderCode){
                case BINDER_CALCULATE:
                    binder = new ICalculateImpl();
                    break;
                case BINDER_RECT:
                    binder = new IRectImpl();
                    break;
                default:
                    break;
            }
            return binder;
        }
    }
}

```
　　在这个BinderPool中我们首先为这两个AIDL接口创建一个id。又创建了一个BinderPoolImpl的内部类，这个BinderPoolImpl内部类根据传入的id返回相对应的Binder。在BinderPool中采用单实例模式，而在BinderPool的构造方法中绑定了Service。对客户端提供了一个queryBinder方法来获取所需要的Binder对象。而对于服务端只需要new BinderPool.BinderPoolImpl()即可。从这我们就可以看出这个BinderPool也就是对客户端和服务端的代码进行了一次封装。然后进行对不同的客户端在Service中返回不同的Binder。由于AIDL支持多线程并发访问的，所以在绑定Service中做了采用synchronized和CountDownLatch做了线程同步处理。
　　由于BinderPool对代码进行了封装，所以服务端的代码就很简单了，只需要new BinderPool.BinderPoolImpl()返回即可。

```java
package com.example.ljd_pc.binderpool;

import android.app.Service;
import android.content.Intent;
import android.os.Binder;
import android.os.IBinder;

public class BinderPoolService extends Service {
    private Binder mBinderPool = new BinderPool.BinderPoolImpl();

    public BinderPoolService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinderPool;
    }
}

```
　　下面在看一下客户端代码。
```java
package com.example.ljd_pc.binderpool;

import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.os.RemoteException;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;

import butterknife.Bind;
import butterknife.ButterKnife;

public class BinderPoolActivity extends AppCompatActivity implements View.OnClickListener{

    @Bind(R.id.add_btn)
    Button addBtn;

    @Bind(R.id.sub_btn)
    Button subBtn;

    @Bind(R.id.area_btn)
    Button areaBtn;

    @Bind(R.id.per_btn)
    Button perBtn;
    private final String TAG = "BinderPoolActivity";

    private BinderPool mBinderPool;
    private ICalculate mCalculate;
    private IRect mRect;
    private Handler mHandler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            addBtn.setOnClickListener(BinderPoolActivity.this);
            subBtn.setOnClickListener(BinderPoolActivity.this);
            areaBtn.setOnClickListener(BinderPoolActivity.this);
            perBtn.setOnClickListener(BinderPoolActivity.this);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_binder_pool);
        ButterKnife.bind(this);
        getBinderPool();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        ButterKnife.unbind(this);
    }

    private void getBinderPool(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                mBinderPool = BinderPool.getInstance(BinderPoolActivity.this);
                mHandler.obtainMessage().sendToTarget();
            }
        }).start();
    }

    @Override
    public void onClick(final View v) {
        try {
            switch (v.getId()){
                case R.id.add_btn:
                    mCalculate = ICalculateImpl.asInterface(mBinderPool.queryBinder(BinderPool.BINDER_CALCULATE));
                    Log.e(TAG,String.valueOf(mCalculate.add(3,2)));
                    Toast.makeText(BinderPoolActivity.this,String.valueOf(mCalculate.add(3,2)),Toast.LENGTH_SHORT).show();
                    break;
                case R.id.sub_btn:
                    mCalculate = ICalculateImpl.asInterface(mBinderPool.queryBinder(BinderPool.BINDER_CALCULATE));
                    Log.e(TAG,String.valueOf(mCalculate.sub(3,2)));
                    Toast.makeText(BinderPoolActivity.this,String.valueOf(mCalculate.sub(3,2)),Toast.LENGTH_SHORT).show();
                    break;
                case R.id.area_btn:
                    mRect = IRectImpl.asInterface(mBinderPool.queryBinder(BinderPool.BINDER_RECT));
                    Log.e(TAG,String.valueOf(mRect.area(3,2)));
                    Toast.makeText(BinderPoolActivity.this,String.valueOf(mRect.area(3,2)),Toast.LENGTH_SHORT).show();
                    break;
                case R.id.per_btn:
                    mRect = IRectImpl.asInterface(mBinderPool.queryBinder(BinderPool.BINDER_RECT));
                    Log.e(TAG,String.valueOf(mRect.perimeter(3,2)));
                    Toast.makeText(BinderPoolActivity.this,String.valueOf(mRect.perimeter(3,2)),Toast.LENGTH_SHORT).show();
                    break;
                default:
                    break;
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}


```
　　在这里我们要注意使用CountDownLatch对bindService这一异步操作变为同步，所以在获取BinderPool对象时不能再主线程中操作。并且在这一步中作者建议我们在Application中去初始化这个BinderPool对象。
　　布局代码如下。
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    android:orientation="vertical">

    <Button
        android:id="@+id/add_btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/add"
        />
    <Button
        android:id="@+id/sub_btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/sub"/>
    <Button
        android:id="@+id/area_btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/area"/>
    <Button
        android:id="@+id/per_btn"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@string/per"/>
</LinearLayout>

```

# **演示**
![这里写图片描述](http://img.blog.csdn.net/20160221215211097)
# **总结**
　　这样以来我们就能在我们的应用中只建立一个Service就足够了。当我们添加一个AIDL接口的时候只需要在BinderPool中添加一个id，然后根据这个id，在BinderPoolImpl中创建一个对应的Binder对象即可。这样就很大程度上简化了我们的工作。但是我认为这样做依然存在一个问题。那就是当我们创建一个BinderPool对象时，我们的客户端就已经绑定了Service，之后只是根据不同的id获取不同的Binder。也就是说从我们绑定Service那时起，这个Service进程就一直在后台运行，即使这个应用已经不再前台使用。除非系统将这个Service进程杀死。但是总的来说，在我们的应用中若是用到了很多的AIDL，那么使用Binder连接池还是很好的选择。到这里有关IPC的AIDL内容就说完了。


# [**源码下载**](https://github.com/lijiangdong/BinderPool)