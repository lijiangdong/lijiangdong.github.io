---
title:  Android的IPC机制(六)—BroadcastReceiver的使用
date: 2016-02-29 18:00
category: [android]
tags: [ipc]
comments: true
---
# **综述**
　　在Android的四大组件中除了ContentProvider能够用于进程间的通信外，还有一个也能够用于进程间的通信，那就是BroadcastReceiver。BroadcastReceiver翻译成中文为广播接收器，既然作为广播接收器，那么必然就有Broadcast。在Android中，Broadcast是一种广泛运用的在应用程序之间传输信息的机制。而BroadcastReceiver则是对发送出来的 Broadcast进行过滤接受并响应的一类组件。在 Android 里面有各种各样的广播，比如电池的使用状态，电话的接收和短信的接收都会产生一个广播，应用程序开发者也可以监听这些广播并做出程序逻辑的处理。<!--more-->
# **生命周期**
　　对于BroadcastReceiver的生命周期也是非常的简单。它的生命周期只存在于onReceive方法中。对于这个onReceive方法它也是运行在主线程中。所以在onReceive方法中不能进行的耗时的操作。否则就会出现ANR(Application Not Responding)。由于这个ANR的限制对于onReceive方法最多可执行的时间为10秒左右。并且由于BroadcastReceiver的生命周期随着onReceive方法的结束而结束。所以我们不能再onReceive方法中去创建一个线程来执行任务。因为onReceive方法执行完毕，这时候这个BroadcastReceiver 也就结束了。也就无法处理异步的结果。如果这个BroadcastReceiver在独立的进程中，它所在进程也很容易在系统需要内存时被优先杀死 , 因为它属于空进程 ( 没有任何活动组件的进程 ). 那么正在工作的子线程也会被杀死 。若是我们需要完成一个比较耗时任务的话，我们可以通过发送Intent给Service，并且由这个Service来完成这项任务。
# **使用方法**
## **注册方式**
　　对于BroadcastReceiver有两种注册方式，一种是动态注册，一种是静态注册。静态注册就是在AndroidManifest文件中进行注册。而动态注册则是在程序中通过registerReceiver方法注册。对于这两种注册方法的优先级来说，动态注册的优先级要高于静态注册的优先级。那么下面我们来看一下这两种注册方式。
　　1.  静态注册
```xml
<receiver
    android:name=".MyReceiver">
    <intent-filter>
        <action android:name="com.example.ljd.BROADCASTRECEIVER" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</receiver>
```
　　2.  动态注册
```java
IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction("com.example.ljd.BROADCASTRECEIVER");
registerReceiver(myReceiver,intentFilter);
```
　　对于动态注册，当我们不在使用的这个broadcastreceiver的时候我们还需要对它解除注册。
```java
unregisterReceiver(myReceiver);
```
## **广播类型**
　　在这里我们介绍一下其中最常用的两种广播：普通广播和有序广播。它们分别是通过sendBroadcast和sendOrderedBroadcast进行发送。那么现在我们就来看一下这两种方式的用法与区别。
### **普通广播（Normal Broadcast）**
　　我们可以通过sendBroadcast方法发送一个普通广播。对于普通广播来说，BroadcastReceiver的优先级一样的话它们接收广播先后顺序是随机的，并且在广播的传输过程中我们无法截断广播。
　　下面我们做一个实验。我们创建三个BroadcastReceiver，使其运行在三个不同的进程内。并且我们采用静态注册方式。
　　第一个BroadcastReceiver
```java
package com.example.ljd.broadcastreceiver;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class MyReceiver1 extends BroadcastReceiver {

    private final String TAG = "MyReceiver1";
    public MyReceiver1() {
    }
    @Override
    public void onReceive(Context context, Intent intent) {

        if (Constant.BROADCAST_ACTION.equals(intent.getAction())){
            Log.d(TAG,intent.getStringExtra(Constant.CONFERENCE_KEY));
        }
    }
}
```
　　第二个BroadcastReceiver
```java
package com.example.ljd.broadcastreceiver;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class MyReceiver2 extends BroadcastReceiver {
    private final String TAG = "MyReceiver2";
    public MyReceiver2() {
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        if (Constant.BROADCAST_ACTION.equals(intent.getAction())){
            Log.d(TAG, intent.getStringExtra(Constant.CONFERENCE_KEY));
        }
    }
}
```
　　第三个BroadcastReceiver
```java
package com.example.ljd.broadcastreceiver;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class MyReceiver3 extends BroadcastReceiver {
    private final String TAG = "MyReceiver3";
    public MyReceiver3() {
    }

    @Override
    public void onReceive(Context context, Intent intent) {
        if (Constant.BROADCAST_ACTION.equals(intent.getAction())){
            Log.d(TAG,intent.getStringExtra(Constant.CONFERENCE_KEY));
        }
    }
}
```
　　对三个BroadcastReceiver进行静态注册。
```xml
<receiver
    android:name=".MyReceiver1"
    android:process=":receiver1">
    <intent-filter>
        <action android:name="com.example.ljd.BROADCASTRECEIVER" />

        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</receiver>
<receiver
    android:name=".MyReceiver3"
    android:process=":receiver3">
    <intent-filter>
        <action android:name="com.example.ljd.BROADCASTRECEIVER" />

        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</receiver>
<receiver
    android:name=".MyReceiver2"
    android:process=":receiver2">
    <intent-filter>
        <action android:name="com.example.ljd.BROADCASTRECEIVER" />

        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</receiver>
```
　　我们在创建一个Activity，在这个Activity里面我们创建一个BroadcastReceiver并且对其进行动态注册。然后我们采用sendBroadcast进行发送广播。下面看一下Activity代码。

```java
package com.example.ljd.broadcastreceiver;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;

import butterknife.ButterKnife;
import butterknife.OnClick;

public class MainActivity extends AppCompatActivity {

    private final String TAG = "MainActivity";

    private MyReceiver myReceiver = new MyReceiver();

    class MyReceiver extends BroadcastReceiver{

        @Override
        public void onReceive(Context context, Intent intent) {
            if (Constant.BROADCAST_ACTION.equals(intent.getAction())){
                Log.d(TAG,intent.getStringExtra(Constant.CONFERENCE_KEY));
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);
        ButterKnife.bind(this);
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Constant.BROADCAST_ACTION);
        registerReceiver(myReceiver,intentFilter);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        ButterKnife.bind(this);
        unregisterReceiver(myReceiver);
    }

    @OnClick(R.id.send_button)
    public void onClickButton(){
        Intent intent = new Intent(Constant.BROADCAST_ACTION);
        intent.putExtra(Constant.CONFERENCE_KEY,"你好，明天九点半101会议室开会。");
        sendBroadcast(intent);
    }
}

```
　　我们发送一条通知开会的广播，并且接收到广播后打印出来。由于代码非常简单这里就不在进行说明，从下面的结果我们可以看出，无论是采用静态注册还是动态注册，它们接收到的先后顺序是随机的。我们发送了三次广播。这时候可以通过DDMS查看一下日志信息。　　
结果1：
![这里写图片描述](http://img.blog.csdn.net/20160303212651706)
结果2：
![这里写图片描述](http://img.blog.csdn.net/20160303212711256)
结果3：
![这里写图片描述](http://img.blog.csdn.net/20160303212752226)
　　现在我们对MyReceiver2中的OnReceive添加一个abortBroadcast方法，这个方法是用来截断广播的。然后我们在看一下运行结果
![这里写图片描述](http://img.blog.csdn.net/20160303220813113)
　　我们可以看到虽然系统给我们抛出了了运行时异常，但是我们还是准确无误的接收到了广播。而这个异常是在abortBroadcast里面的checkSynchronousHint方法中抛出的，这个checkSynchronousHint方法是用来检测广播接收者是否同步接收广播。由于我们接收广播是随机的，所以抛出运行时异常。
　　现在又有一种情形，如果我们只是希望动态创建的BroadcastReceiver进行接收广播，而静态创建的BroadcastReceiver不进行接收。我们可以在发送广播的Intent中设置一个flag：FLAG_RECEIVER_REGISTERED_ONLY。

```java
intent.setFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
```
　　这时候我们可以看到只有动态注册的广播才接收到消息。
![这里写图片描述](http://img.blog.csdn.net/20160303222321947)
### **有序广播（Ordered Broadcast）**
　　对于有序广播而言，BroadcastReceiver是按照接收者的优先级接收广播 , 优先级在 intent-filter 中的 priority 中声明 。priority的值从-1000 到1000 之间 , 值越大 , 优先级越高。如果我们没有设置priority的话，那么对于BroadcastReceiver的优先级则是先注册的要大于后注册的，动态注册优先级大于静态注册的优先级。对于刚才的例子来说在有序广播中它们的优先级就是MyReceiver>MReceiver1>MReceiver3>MReceiver2。并且在在有序广播中我们可以终止广播 ，接收者也能够篡改内容。
　　我们可以通过sendOrderedBroadcast进行发送一个有序广播，他有两个重载方法。
```java
public void sendOrderedBroadcast(Intent intent,
            String receiverPermission)
            
public void sendOrderedBroadcast(
        Intent intent, String receiverPermission, BroadcastReceiver resultReceiver,
        Handler scheduler, int initialCode, String initialData,
        Bundle initialExtras)
```
这里我们对里面的参数进行一下说明：　
1.  intent：所有匹配的这个intent的BroadcastReceiver将接收广播。
2.  receiverPermission：这个是权限，一个接收器必须持有接收我们的广播的权限。如果BroadcastReceiver没有设置任何权限，这里为null即可。
3.  resultReceiver ：设置一个BroadcastReceiver 来当作最后的广播接收器。 
4.  initialCode：结果代码的初始值。通常为 Activity.RESULT_OK ，这个值是 -1 。也可以为其他 int 型，如 0,1,2 ； 
5.  initialData： 一种结果数据的初始值。为String 类型 ;
6.  initialExtras：一种结果额外的初始值。为Bundle类型。
　　
　　对于我们没有设置priority属性的BroadcastReceiver的优先级情况很简单这里也就不在进行验证。现在我们来修改上面代码来看一下是如何拦截广播并且修改广播中数据的。我们首先对动态注册的优先级设为0。对上面的MyReceiver1,MyReceiver2,MyReceiver3的优先级分别设为3，2，1。然后我们通过sendOrderedBroadcast发送一个有序广播。
```java
Intent intent = new Intent(Constant.BROADCAST_ACTION);
intent.putExtra(Constant.CONFERENCE_KEY, "你好，明天九点半101会议室开会。");

Bundle bundle = new Bundle();
bundle.putString(Constant.DINE_KEY, "今天晚上聚餐");
intent.putExtras(bundle);
sendOrderedBroadcast(intent,null,null,null, Activity.RESULT_OK ,null,bundle);
```
　　在MyReceiver1的onReceive方法中修改如下：
```java
if (Constant.BROADCAST_ACTION.equals(intent.getAction())){
     Log.d(TAG, intent.getStringExtra(Constant.CONFERENCE_KEY));
     Log.d(TAG, getResultExtras(true).getString(Constant.DINE_KEY));
     Bundle bundle = getResultExtras(true);
     bundle.putString(Constant.DINE_KEY, "后天中午一起聚餐");
     setResultExtras(bundle);
 }
```
　　在MyReceiver2的onReceive方法中修改如下：
```java
if (Constant.BROADCAST_ACTION.equals(intent.getAction())){
    Log.d(TAG, intent.getStringExtra(Constant.CONFERENCE_KEY));
    Log.d(TAG, getResultExtras(true).getString(Constant.DINE_KEY));
    abortBroadcast();
}
```
　　getResultExtras：优先级低的BroadcastReceiver可以通过这个方法获取到最新的经过处理的信息集合。
 　　下面我们来看一下运行结果：
 ![这里写图片描述](http://img.blog.csdn.net/20160303235146895)
 　　在这里我们可以看到由于在MyReceiver2中的onReceive方法中设置了abortBroadcast，比MyReceiver2优先级低的BroadcastReceiver接收不到广播。并且在MyReceiver1中将“今天晚上聚餐”成功修改为“后天中午聚餐”。
# **总结**
　　BroadcastReceiver除了可以接收我们自己发送的广播以外，还能够接收一些系统的广播。例如对网络环境的监测，手机电量的监测，设备重启的监测等等。也就是说BroadcastReceiver可以方便应用程序和系统，应用程序之间，应用程序内的通信。所以对于我们单个应用程序来说它是存在安全性的问题。不过我们我们可以在发送广播时指定接收者必须具备的permission。当然我们也可以通过使用LocalBroadcastManager来解决安全性问题。对于LocalBroadcastManager使用也很简单。它是一个单例模式，可以通过LocalBroadcastManager.getInstance(context)获取LocalBroadcastManager对象。然后通过LocalBroadcastManager里面的registerReceiver方法进行BroadcastReceiver进行注册。也就是说使用LocalBroadcastManager必须进行动态注册。
# **[源码下载](https://github.com/lijiangdong/BroadcastReceiver)**