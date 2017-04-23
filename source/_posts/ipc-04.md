---
title:  Android的IPC机制(四)— Messenger的使用及源码分析
date: 2016-02-25 18:22
category: [Android]
tags: [ipc,aidl]
comments: true
---

　　在前面几篇中我们详细的介绍了AIDL的使用及原理。在这里我们感觉到AIDL的在使用过程中还是比较复杂的，那么有没有一种简单的方法来实现进程间的通信呢？当然是有的，那就是利用Messenger。Messenger翻译为信使，从他的名字就可以看出这个Messenger就是作为传递消息用的。那么我们就来看一下这个Messenger到底是如何使用的，以及在它内部是如何实现的。<!--more-->
# **Messenger的使用**
## **使用步骤**
　　其实对于Messenger用起来是非常简单的，那么我们首先来看一下这个Messenger的使用步骤：
　　1.  在服务端我们实现一个 Handler，接收来自客户端的每个调用的回调
　　2.  这个Handler 用于创建 Messenger 对象（也就是对 Handler 的引用）
　　3.  用Messenger 创建一个 IBinder，服务端通过 onBind() 使其返回客户端
　　4.  客户端使用 IBinder 将 Messenger（引用服务的 Handler）实例化，然后使用后者将 Message 对象发送给服务端
　　5.  服务端在其 Handler 中（具体地讲，是在 handleMessage() 方法中）接收每个 Message
　　在这里我们写一个例子来更详细的说明这个Messenger的使用。　　
## **演示**
　　在这里我们写一个demo，我们通过客户端随机产生一个由小写字母组成的10位字符串发送给服务端，由服务端转换为大写后再发送给客户端。
![这里写图片描述](http://img.blog.csdn.net/20160225212718165)
## **源代码**
　　下面是服务端代码。
```java
package com.ljd.messenger;

import android.app.Service;
import android.content.Intent;
import android.os.Bundle;
import android.os.Handler;
import android.os.IBinder;
import android.os.Message;
import android.os.Messenger;
import android.os.RemoteException;

public class MessengerService extends Service {

    private final Messenger mMessenger = new Messenger(new ServiceHandler());
    private class ServiceHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 0:
                    Messenger clientMessenger = msg.replyTo;
                    Message replyMessage = Message.obtain();
                    replyMessage.what = 1;
                    Bundle bundle = new Bundle();
                    //将接收到的字符串转换为大写后发送给客户端
                    bundle.putString("service", msg.getData().getString("client").toUpperCase());
                    replyMessage.setData(bundle);
                    try {
                        clientMessenger.send(replyMessage);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}

```
　　下面我们使服务端运行在独立的进程中。

```xml
        <service
            android:name=".MessengerService"
            android:process=":remote">
        </service>
```
　　下面是客户端代码。
```java
package com.ljd.messenger;

import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Handler;
import android.os.IBinder;
import android.os.Message;
import android.os.Messenger;
import android.os.RemoteException;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.LinearLayout;
import android.widget.TextView;

import java.util.Random;

import butterknife.Bind;
import butterknife.ButterKnife;
import butterknife.OnClick;

public class MainActivity extends AppCompatActivity {

    @Bind(R.id.messenger_linear)
    LinearLayout mShowLinear;

    private Messenger mMessenger;
    private final String LETTER_CHAR = "abcdefghijkllmnopqrstuvwxyz";




    private class ClientHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case 1:
                    TextView textView = new TextView(MainActivity.this);
                    textView.setText("convert ==>:"
                            + (msg.getData().containsKey("service")?msg.getData().getString("service"):""));
                    mShowLinear.addView(textView);
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mMessenger = new Messenger(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            bindService();
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        bindService();
    }

    @OnClick({R.id.test_button,R.id.clear_button})
    public void onClickButton(View v){
        switch (v.getId()){
            case R.id.test_button:
                sendToService();
                break;
            case R.id.clear_button:
                mShowLinear.removeAllViews();
                break;
        }
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
        ButterKnife.unbind(this);
    }

    private void bindService(){
        Intent intent = new Intent(MainActivity.this,MessengerService.class);
        bindService(intent,mConnection, Context.BIND_AUTO_CREATE);
    }

    private void sendToService(){
        Message messageClient = Message.obtain(null,0);
        //用于传递给服务端回复的Messenger
        Messenger replyMessenger = new Messenger(new ClientHandler());
        Bundle bundle = new Bundle();
        bundle.putString("client",generateMixString());
        messageClient.setData(bundle);
        //通过Message的replyTo属性将Messenger对象传递到服务端
        messageClient.replyTo = replyMessenger;
        TextView textView = new TextView(MainActivity.this);
        textView.setText("send:" + (bundle.getString("client")));
        mShowLinear.addView(textView);
        try {
            mMessenger.send(messageClient);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
    /**
     * 随机生成10位小写字母的字符串
     * @return
     */
    public String generateMixString() {
        StringBuffer sb = new StringBuffer();
        Random random = new Random();
        for (int i = 0; i < 10; i++) {
            sb.append(LETTER_CHAR.charAt(random.nextInt(LETTER_CHAR.length())));
        }
        return sb.toString();
    }
}

```
　　注意：我们可以看到在Messenger中通过Message进行传递数据的。在Message中的字段obj是不能用于进程间通信的。
　　
# **Messenger原理分析**
　　首先我们进入Messenger这个类里面看一下这个Messenger到底是什么东西。
```java
package android.os;

public final class Messenger implements Parcelable {
    private final IMessenger mTarget;

    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
    
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }
    
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
    
    public boolean equals(Object otherObj) {
        if (otherObj == null) {
            return false;
        }
        try {
            return mTarget.asBinder().equals(((Messenger)otherObj)
                    .mTarget.asBinder());
        } catch (ClassCastException e) {
        }
        return false;
    }

    public int hashCode() {
        return mTarget.asBinder().hashCode();
    }
    
    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel out, int flags) {
        out.writeStrongBinder(mTarget.asBinder());
    }

    public static final Parcelable.Creator<Messenger> CREATOR
            = new Parcelable.Creator<Messenger>() {
        public Messenger createFromParcel(Parcel in) {
            IBinder target = in.readStrongBinder();
            return target != null ? new Messenger(target) : null;
        }

        public Messenger[] newArray(int size) {
            return new Messenger[size];
        }
    };
    
    public static void writeMessengerOrNullToParcel(Messenger messenger,
            Parcel out) {
        out.writeStrongBinder(messenger != null ? messenger.mTarget.asBinder()
                : null);
    }
    
    public static Messenger readMessengerOrNullFromParcel(Parcel in) {
        IBinder b = in.readStrongBinder();
        return b != null ? new Messenger(b) : null;
    }
    
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
}

```
　　我们可以看到这个Messenger实现Parcelable接口。然后声明了一个IMessenger 对象mTarget。并且在Messenger中的两个构造方法中都对这个mTarget进行了初始化。那么这个IMessenger 是什么东西呢？我们首先看一下Messenger中的第一个构造方法。	 
```java
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}
```
　　首先我们使用这个构造方法在服务端创建一个Messenger对象，在onBind中通过Messenger中的getBinder方法向客户端返回一个Binder对象。而在这个构造方法里面通过Handler中的getIMessenger方法初始化mTarget。那么我们进入Handler中看一下这个getIMessenger方法。　
```java
public class Handler {
	...
	
	final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }

    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            Handler.this.sendMessage(msg);
        }
    }
    
    ...
}
```
　　这回我们看到了这个getIMessenger方法中返回了一个MessengerImpl对象，而这个MessengerImpl中对send方法实现，也只不过是通过Handler发送了一个message。并且在Handler中的handleMessage方法中进行处理。
　　而在我们的客户端当中，我们通过客户端的onServiceConnected中拿到服务端返回的Binder对象，并且在onServiceConnected方法中new Messenger(service)去获取这个IMessenger对象，下面我们就看一下这个Messenger的第二个构造方法。　
```java
public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```
　　这个构造方法一般用于客户端中。并且到现在我们一路走下来似乎看到了许多熟悉的身影IMessenger.Stub，Stub.asInterface等等。这些不正是我们在AIDL中使用到的吗？于是我们猜想这个IMessenger应该是一个AIDL接口。现在就去找找这个IMessenger。这个IMessenger位于[frameworks](https://github.com/android/platform_frameworks_base.git)的base中， 完整路径为frameworks/base/core/java/android/os/。我们就看一下这个AIDL接口，
```java
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```
　　在这个IMessenger接口中只有一个send方法。而在这里关键字oneway表示当服务用户请求相应功能的时后不需要等待应答就可以直接调用返回，这个关键字可以用于接口声明或者方法声明语句中，如果接口声明语句中使用了oneway关键字，则这个接口中声明的所有方法都采用了oneway方式。这回我们也就明白这个Messenger了，其实这个Messenger就是进一步对AIDL进行了一次封装。
　　在这里客户端通过服务端返回的Binder创建了Messenger对象。然后我们只需要创建一个Message对象，在这个Message对象中携带我们所需要传递给服务端的数据，这时候就可以在服务端中的handleMessage进行处理。若是我们需要服务端给我们返回数据，只需要在客户端创建一个Handler，并且使用这个Handler创建一个Messenger对象，然后将这个Messenger对象通过Message中的replyTo字段传递到服务端，在服务端获取到客户端的Messenger对象后，便可以通过这个Messenger发送给客户端，然后在客户端中的handlerMessage处理即可。
# **AIDL与Messenger的区别**
　　我们可以发现在IPC通信中使用Messenger要比使用AIDL简单很多。因为在Messenger中是通过Handler来对AIDL进行的封装，也就是说Messenger是通过队列来调用服务的，而单纯的AIDL会同时像服务端发出请求，这时候我们就必须对多线程进行处理。
　　那么对于一个应用来说。它的Service不需要执行多线程我们应该去使用这个Messenger，返过来若是我们Service处理是多线程的我们就应该使用AIDL去定义接口。
# **总结**
　　Messenger它是一种轻量级的IPC 方法，我们使用起来也是非常的简单。他是使用Handler对AIDL进行了一次封装，一次只能处理一个请求。并且Messenger在发送Message的时候不能使用他的obj字段，我们可以用bundle来代替。最后还有一点就是Messenger只是在客户端与服务端跨进程的传递数据，而不能够去访问服务端的方法。
# **[源码下载](https://github.com/lijiangdong/messenger)**    