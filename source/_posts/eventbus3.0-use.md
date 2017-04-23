---
title:  EventBus3.0使用详解
date: 2016-03-12 16:50 
category: [Android]
tags: [EventBus]
comments: true
---
　　这里所介绍的EventBus指的是greenrobot的EventBus，它是一款针对Android的发布/订阅事件总线。它能够让我们很轻松的实现在Android的各个组件以及线程之间进行传递消息。并且将事件的发送者与接收者之间进行解耦。而且他还是轻量级的Android类库。对于EventBus3.0中相对于先前的版本中用法有所改变，那么下面我们就来看一下如何使用这个EventBus;<!--more-->
# **使用方法**
　　对于EventBus的使用也是非常简单的，事件的发送方将事件发出，通过EventBus将事件传递给改事件的订阅者进行使用。

<div align="center">
<img  src="http://img.blog.csdn.net/20160312142714321"  width="75%" />
<div>

## **基本用法**
　　首先我们需要将EventBus添加到我们的项目中。在AndroidStudio中我们可以在gradle里面直接配置即可。
```
compile 'org.greenrobot:eventbus:3.0.0'
```
　　当然我们也可以下载EventBus的jar包导入我们的项目里面。在这里我们写一个小例子来看一下EventBus最基本的用法。在这里为了方便我们的观察，我们就将一个Activity分为左右两个部分，分别在里面添加一个Fragment，左边的Fragment用于事件的发送，右边的Fragment用于接收事件。
　　在这里我们先看一下效果演示。

<img width = "60%" src="http://img.blog.csdn.net/20160312183325201" />

　　我们需要创建一个实体类作为EventBus中的事件
```java
package com.ljd.example.eventbus;

public class MessageEvent {

    public final String message;

    public MessageEvent(String message) {
        this.message = message;
    }
}
```
　　然后我们在创建一个Activity。
```java
package com.ljd.example.eventbus;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```
　　在这里我们可以看到在Activity当中只是加载一下布局文件，之后什么也没有做，其实所有的事情都交给了Fragment来处理。下面我们看一下Activity的布局。

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="horizontal"
    tools:context="com.ljd.example.eventbus.MainActivity">


    <Fragment
            android:id="@+id/left_Fragment"
            android:name="com.ljd.example.eventbus.LeftFragment"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"></Fragment>
    <TextView
        android:layout_width="1dp"
        android:layout_height="match_parent"
        android:background="@color/gray"/>
    <Fragment
            android:id="@+id/right_Fragment"
            android:name="com.ljd.example.eventbus.RightFragment"
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="1"></Fragment>
</LinearLayout>
```
　　在这里我们将Fragment当做一个View，直接写入布局文件中。下面看一下左边的Fragment。
```java
package com.ljd.example.eventbus;


import android.os.Bundle;
import android.support.v4.app.Fragment;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.Button;
import android.widget.LinearLayout;

import org.greenrobot.eventbus.EventBus;


/**
 * A simple {@link Fragment} subclass.
 */
public class LeftFragment extends Fragment {

    private LinearLayout buttonLinear;
    public LeftFragment() {
        // Required empty public constructor
    }


    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.Fragment_left, container, false);
        buttonLinear = (LinearLayout)view.findViewById(R.id.left_Fragment_linear);
        sendEvent();
        return view;
    }

    private void sendEvent(){
        Button button = new Button(getActivity());
        button.setText("SEND");
        buttonLinear.addView(button);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                EventBus.getDefault().post(new MessageEvent("Hello everyone!"));
            }
        });
    }
}
```
　　这里我们注意到我们通过EventBus.getDefault()去获取一个EventBus对象，然后通过post方法进行发送事件。这时候就完成了事件的发布过程。下面我们再看一下右边的Fragment是如何接收事件的。

```java
package com.ljd.example.eventbus;


import android.os.Bundle;
import android.support.v4.app.Fragment;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.LinearLayout;
import android.widget.TextView;

import org.greenrobot.eventbus.EventBus;
import org.greenrobot.eventbus.Subscribe;


/**
 * A simple {@link Fragment} subclass.
 */
public class RightFragment extends Fragment {

    private LinearLayout mTextViewLinear;

    public RightFragment() {
        // Required empty public constructor
    }

    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.Fragment_right, container, false);
        mTextViewLinear = (LinearLayout)view.findViewById(R.id.right_Fragment_linear);
        return view;
    }

    @Override
    public void onStop() {
        EventBus.getDefault().unregister(this);
        super.onStop();
    }

    @Subscribe
    public void onMessage(MessageEvent event) {
        TextView textView = new TextView(getActivity());
        textView.setText(event.message);
        mTextViewLinear.addView(textView);
    }
}
```
　　对于这两个Fragment的布局非常简单，里面只包含一个垂直方向的线性布局。在这里就不在贴出。下面我们就来看一下EventBus是如何来接收事件的。
　　如果我们需要接收某个事件并进行处理的话，首先我们需要通过EventBus.getDefault().register(this)将订阅者的对象注册到EventBus中，当我们不在使用这个事件时还需要通过EventBus.getDefault().unregister(this)解除注册。这时候我们就可以再写一个方法，参数为所要接收事件的对象，并且在该方法上需要添加@Subscribe来标示这个方法为接收事件的方法。这时候我们就可以接收事件并且对事件进行处理。
　　在这里有一点需要注意，在EventBus3.0中接收事件的方法是通过@Subscribe来标识的，方法名可以随意写。而在EventBus先前的版本中接收事件的方法必须是以onEvent开头的四个方法。它们分别是onEvent，onEventMainThread，onEventBackground，onEventAsync。

## **EventBus中的线程模式**
　　在上面的例子中我们可以看到，我们是在主线程（也称为UI线程）中去提交事件，然后订阅者也是在主线程中对事件进行的处理。但是在Android开发中我们都知道无法在主线程中去执行一个耗时的任务，并且子线程中我们也无法进行更新UI的操作。那么这时候问题就来了，这里再上一个例子的基础上假设一种情形，我们在左边的Fragment中从网络中获取到数据，然后将从网络获取到的数据在右边的Fragment中显示。可是在Android4.0以后我们无法在主线程中去请求网络，也就是说我们只能在子线程中请求数据，然后在子线程中去提交事件。那么这时候订阅者的方法是在主线程中执行还是在子线程中执行的呢？这里就要来看一下EventBus中的ThreadMode了。EventBus的ThreadMode总共有四种，并且都是在订阅者中的@Subscribe里进行制定的。下面我们就来看一下这四种ThreadMode。
　　1.  ThreadMode: POSTING
　　这时候订阅者执行的线程与事件的发布者所在的线程为同一个线程。也就是说事件由哪个线程发布的，订阅者就在哪个线程中执行。这个也是EventBus默认的线程模式，也就是说在上面的例子中用的就是这种ThreadMode。由于没有线程的切换，也就意味消耗的资源也是最小的。如果一个任务不需要多线程的，也是推荐使用这种ThreadMode的。在EventBus以前的版本中对应onEvent方法。使用例子：
```java
//与事件的提交者运行在同一个线程（默认的ThreadMode）
@Subscribe(threadMode = ThreadMode.POSTING) // 这里的threadMode可以省略不写
public void onMessage(MessageEvent event) {
    log(event.message);
}
```
　　2.  ThreadMode: MAIN
　　从它的名字就很容易可以看出，他是在Android的主线程中运行的。如果提交的线程也是主线程，那么他就和ThreadMode.POSTING一样了。当然在这里由于是在主线程中运行的，所以在这里就不能执行一些耗时的任务。在EventBus以前的版本中对应onEventMainThread方法。使用例子：
```java
// 在Android的主线程中运行
@Subscribe(threadMode = ThreadMode.MAIN)
public void onMessage(MessageEvent event) {
    textField.setText(event.message);
}
```
　　3.  ThreadMode: BACKGROUND
　　这种模式下，我们的订阅者将会在后台线程中执行。如果发布者是在主线程中进行的事件发布，那么订阅者将会重新开启一个子线程运行，若是发布者在不是在主线程中进行的事件发布，那么这时候订阅者就在发布者所在的线程中执行任务。在EventBus以前的版本中对应onEventBackground方法。使用例子：
```java
// 在后台线程中执行
@Subscribe(threadMode = ThreadMode.BACKGROUND)
public void onMessage(MessageEvent event){
    saveToDisk(event.message);
}
```
　　4.  ThreadMode: ASYNC
　　在这种模式下，订阅者将会独立运行在一个线程中。不管发布者是在主线程还是在子线程中进行事件的发布，订阅者都是在重新开启一个线程来执行任务。在EventBus以前的版本中对应onEventAsync方法。使用例子：
```java
// 在独立的线程中执行
@Subscribe(threadMode = ThreadMode.ASYNC)
public void onMessage(MessageEvent event){
    backend.send(event.message);
}
```
　　对于EventBus的线程模式讲了这么多，下面我们来写一个例子验证一下上面对应的四种ThreadMode。我们就在上面的例子中添加一些内容。首先在左边的Fragment中添加一个发送按钮，用来提交事件。并且这个发布者发送事件是在子线程中运行的。我们看一下关键代码。
```java
private void testThreadMode(){
    Button button = new Button(getActivity());
    button.setText("TEST THREAD MODE");
    mButtonLinear.addView(button);
    button.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Log.d(TAG,"订阅者线程ID:"+Thread.currentThread().getId());
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    EventBus.getDefault().post(new MessageEvent("Hello everyone!"));
                }
            }).start();
        }
    });
}
```
　　在上面发布事件过程中我们使线程休眠5秒钟时间，用来模拟耗时操作。下面我们在右边的Fragment中添加如下代码进行测试。

```java
@Subscribe(threadMode = ThreadMode.POSTING)
public void onPostingModeMessage(MessageEvent event){

    Log.d(TAG, getResultString("ThreadMode:POSTING",event.message));
}

@Subscribe(threadMode = ThreadMode.MAIN)
public void onMainModeMessage(MessageEvent event){

    Log.d(TAG, getResultString("ThreadMode:MAIN",event.message));
}

@Subscribe(threadMode = ThreadMode.BACKGROUND)
public void onBackgroundModeMessage(MessageEvent event){

    Log.d(TAG, getResultString("ThreadMode:BACKGROUND",event.message));
}

@Subscribe(threadMode = ThreadMode.ASYNC)
public void onAsyncModeMessage(MessageEvent event){

    Log.d(TAG, getResultString("ThreadMode:ASYNC",event.message));
}

private String getResultString(String threadMode,String msg){
    StringBuilder sb = new StringBuilder("");
    sb.append(threadMode)
            .append("\n接收到的消息：")
            .append(msg)
            .append("\n线程id:")
            .append(Thread.currentThread().getId())
            .append("\n是否是主线程：")
            .append(Looper.getMainLooper() == Looper.myLooper())
            .append("\n");
    return sb.toString();
}
```
　　在这里只贴出了关键代码，具体源码可以通过下面链接进行下载。在这里我们点击这个Button发送一个事件，然后我们从logcat中看一下输出的信息。
![这里写图片描述](http://img.blog.csdn.net/20160312233431993)
　　从上面的结果我们可以看出来由于事件的发布者是在子线程中，所以BACKGROUND与POSTING模式下订阅者与事件的发布者运行在同一个线层。而ASYNC模式下又重新开起一个线程来执行任务。Main模式则是在主线程中运行。
## **优先级与事件的取消**
　　在订阅者中我们也可以为其设置优先级，优先级高的将会首先接收到发布者所发布的事件。并且我们还能在高优先中取消事件，这时候的优先级的订阅者将接收不到事件。这类似于BroadcastReceiver中的取消广播。不过这里有一点我们要注意，对于订阅者的优先级只是针对于相同的ThreadMode中。默认的优先级为0。下面我们来做一个测试。
　　在上面的例子中我们添加一下代码。
```java
@Subscribe(priority = 1)
public void onPriority1Message(MessageEvent event){

    Log.d(TAG, "priority = 1:" + event.message);
}

@Subscribe(priority = 2)
public void onPriority2Message(MessageEvent event){

    Log.d(TAG, "priority = 2:" + event.message);
    EventBus.getDefault().cancelEventDelivery(event) ;
}

@Subscribe(priority = 4)
public void onPriority4Message(MessageEvent event){

    Log.d(TAG, "priority = 4:" + event.message);
}

@Subscribe(priority = 3)
public void onPriority3Message(MessageEvent event){

    Log.d(TAG, "priority = 3:" + event.message);

}
```
　　在这里我们将优先级为2的接收事件的方法中取消事件，这么一来优先级为1的将接收不到该事件。下面我们来看一下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160313201015366)
　　在这里我们可以清楚的看到优先级高的首先接收到事件，并且成功取消该事件。

## **订阅者索引**
　　对于上面所描述的EventBus的功能，是通过Java反射来获取订阅方法，这样以来大大降低了EventBus的效率，同时也影响了我们应用程序的效率。其实对于反射的处理解析不仅仅只能够通过Java反射的方式来进行，还能够通过apt(Annotation Processing Tool)来处理。为了提高效率，EventBus提供这中方式来完成EventBus的执行过程。下面就来看一下对于EventBus的另一种使用方式。
　　在Project的build.gradle中添加如下代码：
```java
buildscript {
    dependencies {
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}
```
　　然后在app的build.gradle中添加如下代码。
```java
apply plugin: 'com.neenbedankt.android-apt'

dependencies {
    compile 'org.greenrobot:eventbus:3.0.0'
    apt 'org.greenrobot:eventbus-annotation-processor:3.0.1'
}
apt {
    arguments {
        eventBusIndex "com.example.myapp.MyEventBusIndex"
    }
}
```
　　在当我们使用EventBus以后，在我们的项目没有错误的情况下重新rebuild之后会在build目录下面生成MyEventBusIndex文件，文件名可以自定义。下面就来看一下如何使用这个MyEventBusIndex。
　　我们可以自定义设置自己的EventBus来为其添加MyEventBusIndex对象。代码如下所示：
```java
EventBus eventBus = EventBus.builder().addIndex(new MyEventBusIndex()).build();
```
　　我们也能够将MyEventBusIndex对象安装在默认的EventBus对象当中。代码如下所示：
```java
EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
// Now the default instance uses the given index. Use it like this:
EventBus eventBus = EventBus.getDefault();
```
　　剩下对于EventBus的用法则是一模一样。当然也建议通过添加订阅者索引这种方式来使用EventBus，这样会比通过反射的方式来解析注解效率更高。

## **ProGuard**
　　当我们使用ProGuard混淆时，我们还需要在我们的ProGuard的配置文件中添加如下代码。
```bash
-keepattributes *Annotation*
-keepclassmembers class ** {
    @org.greenrobot.eventbus.Subscribe <methods>;
}
-keep enum org.greenrobot.eventbus.ThreadMode { *; }

# Only required if you use AsyncExecutor
-keepclassmembers class * extends org.greenrobot.eventbus.util.ThrowableFailureEvent {
    <init>(java.lang.Throwable);
}
```
# **总结**
　　对于EventBus的使用我们就说到这里，对于EventBus想要有更深入的了解，我们可以去github上下载[EventBus源码](https://github.com/greenrobot/EventBus)进行研究。
# **[源码下载](https://github.com/lijiangdong/eventbus-example)**