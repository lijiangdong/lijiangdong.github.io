---
title:  Android的IPC机制(七)—Socket的原理简析与使用
date: 2016-03-06 20:54 
category: [Android]
tags: [ipc,socket]
comments: true
---

　　在前面的几篇文章中，我们介绍了许多在Android中有关进程间通信的方式，但都是在一个设备上进行的进程间通信，而这时候我们两个应用在不同的设备上的时候，在这个时候我们就不能通过前方介绍的那些方法来解决了。但是我们通过网络进行通信来处理这个问题。今天就来介绍一下Android中网络通信的其中一种方式——Socket。Socket翻译为中文为套接字，而现在套接字也成为了操作系统中的一部分。下面我们就来看一下如何使用这个套接字的。<!--more-->
# **TCP/IP介绍**
　　这里有一点需要说明一下，在Internet所使用的各种协议中，最重要的和最著名的就是TCP和IP两个协议。而我们现在经常提到的TCP/IP并不一定单指TCP和IP这两个具体的协议，而往往表示Internet所使用的整个TCP/IP协议族。
## **TCP/IP的体系结构**
　　TCP/IP协议可以为各式各样的应用提供服务，同时TCP/IP协议也允许IP协议在各式各样的网络构成的互联网上运行。TCP/IP协议是一个四层的体系结构，它包含应用层、传输层、网络层、网络接口层。在传输层中主要使用的有两种协议：TCP（Transmission Control Protocol，传输控制协议）和UDP（User Datagram Protocol，用户数据报协议）。下面是TCP/IP体系结构示意图。
![这里写图片描述](http://img.blog.csdn.net/20160308114056753)
## **传输控制协议TCP**
　　TCP是TCP/IP体系中非常复杂的一个协议，在这里我们简单看一下TCP协议的特点，对于TCP协议的详细内容，可以查看一些计算机网络的相关书籍。
1.  TCP是面向连接传输协议，也就是说在我们的应用程序在使用TCP协议之前，必须先建立起TCP连接。在传送数据完毕后，必须释放已经建立的TCP连接。就像我们打电话一样，打电话之前首先需要拨号进行建立连接，等通话结束后再挂断释放连接。
2.  每一条TCP连接只能有两个端点，每一条TCP连接只能是点对点的。
3. TCP提供了一个可靠交付的服务，也就是说通过TCP连接传送的数据，无差错，不丢失，不重复，并且按序到达。
4. TCP提供全双工通信，它允许通信双方的应用进程在任何时候都能够发送数据。
5. TCP通信中是面向字节流的，其中的“流”指的是流入到进程或从进程流出的字节序列。
## **用户数据报协议UDP**
　　用户数据报协议UDP只是在IP的数据服务上增加了很少的一点功能（复用、分用的功能以及差错检测的功能）。在这里简单说一下UDP的特点。
1.  UDP是无连接，也就是在发送数据之前是不需要建立连接的，也就减少了开销和发送数据之间的延时。
2. UDP它只能是尽最大努力地交付，也就是不能够保证可靠交付。
3. UDP它是面向报文的。发送方的UDP对应用程序交下来的报文，再添加首部后就向下交付给IP层。
4. UDP它没有拥塞控制，也就是说在网络出现拥塞的情况下不会使源主机的发送速率降低。
5. UDP支持一对一，一对多，多对一和多对多的交互通信。
## **Socket在TCP/IP中的作用**
　　Socket是工作于TCP/IP协议中应用层和传输层之间的一种抽象（不属于应用层也不属于传输层）。在Android系统中，它可以分为流套接字（streamsocket）和数据报套接字(datagramsocket)。而Socket中的流套接字将TCP协议作为其端对端协议，提供了一个可信赖的字节流服务；数据报套接字使用UDP协议，提供数据打包发送服务。
　　在网络编程的时候，我们经常把Socket作为应用进程和传输层协议之间的接口。在下面图中表示了这样一个概念。在图中我们假定了运输层使用的是TCP协议（如果使用的是UDP协议，情况也是类似的，只是UDP是无连接的通信的两端依然可以用两个套接字来标志）。并且现在套接字已经成为操作系统内核的一部分。
![这里写图片描述](http://img.blog.csdn.net/20160308132117894)
　　不过有一点我们要注意，在套接字以上的进程是受应用程序控制的，而在套接字以下的的传输层协议软件则是由计算机操作系统控制。因此，只要我们的应用程序使用TCP/IP协议进行通信，它就必须通过Socket与操作系统交互并请求服务。从这里可以看出来，我们对Socket以上的应用进程具有完全的控制，但对Socket以下的传输层却只有很少的控制。例如，我们可以选择传输层协议（TCP或UDP）和一些传输层的参数（如最大缓存空间和最大报文长度）。
# **Socket使用案例**
　　在这里我们选择传输层协议为TCP协议，也就是我们将使用流套接字作为例子进行举例说明。现在我们现在做一个聊天室功能。在这里我们创建两个应用程序，分别运行在两个不同的设备上。首先看一下效果图。
## **演示**

<div><img src="http://img.blog.csdn.net/20160308150825062" width = "300" >&nbsp&nbsp&nbsp&nbsp
<img src="http://img.blog.csdn.net/20160308150843437" width = "298" ></div>

## **客户端代码**
```java
package com.example.ljd.socketclient;

import android.annotation.SuppressLint;
import android.os.Handler;
import android.os.Message;
import android.os.SystemClock;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.View;
import android.widget.EditText;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;

import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.Socket;
import java.sql.Date;
import java.text.SimpleDateFormat;

import butterknife.Bind;
import butterknife.ButterKnife;
import butterknife.OnClick;

public class MainActivity extends AppCompatActivity{

    private static final int RECEIVE_NEW_MESSAGE = 1;
    private static final int SOCKET_CONNECT_SUCCESS = 2;
    private static final int SOCKET_CONNECT_FAIL = 3;

    @Bind(R.id.msg_edit_text)
    EditText mMessageEditText;

    @Bind(R.id.show_linear)
    LinearLayout mShowLinear;

    private PrintWriter mPrintWriter;
    private Socket mClientSocket;
    private boolean mIsConnectServer = false;

    @SuppressLint("HandlerLeak")
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {

                case RECEIVE_NEW_MESSAGE:
                    TextView textView = new TextView(MainActivity.this);
                    textView.setText((String)msg.obj);
                    mShowLinear.addView(textView);
                    break;

                case SOCKET_CONNECT_SUCCESS:
                    Toast.makeText(MainActivity.this,"连接服务端成功",Toast.LENGTH_SHORT).show();
                    break;

                case SOCKET_CONNECT_FAIL:
                    Toast.makeText(MainActivity.this,"连接服务端失败，请重新尝试",Toast.LENGTH_SHORT).show();
                    break;
                default:
                    break;

            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);

    }

    @Override
    protected void onDestroy() {
        ButterKnife.unbind(this);
        disConnectServer();
        super.onDestroy();
    }

    @OnClick({R.id.send_btn,R.id.connect_btn,R.id.disconnect_btn})
    public void onClickButton(View v) {

        switch (v.getId()){
            case R.id.send_btn:
                sendMessageToServer();
                break;
            case R.id.connect_btn:
                new Thread() {
                    @Override
                    public void run() {
                        connectServer();
                    }
                }.start();
                break;
            case R.id.disconnect_btn:
                Toast.makeText(MainActivity.this,"已经断开连接",Toast.LENGTH_SHORT).show();
                disConnectServer();
                break;
        }

    }

    private String getTime(long time) {
        return new SimpleDateFormat("(HH:mm:ss)").format(new Date(time));
    }

    private void connectServer() {

        if (mIsConnectServer)
            return;

        int count = 0;
        while (mClientSocket == null) {
            try {
                mClientSocket = new Socket("10.10.14.160", 8088);
                mPrintWriter = new PrintWriter(new BufferedWriter(
                        new OutputStreamWriter(mClientSocket.getOutputStream())), true);
                mIsConnectServer = true;
                mHandler.obtainMessage(SOCKET_CONNECT_SUCCESS).sendToTarget();
            } catch (IOException e) {
                SystemClock.sleep(1000);
                count++;
                if (count == 5){
                    mHandler.obtainMessage(SOCKET_CONNECT_FAIL).sendToTarget();
                    return;
                }
            }
        }

        try {
            // 接收服务器端的消息
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(
                    mClientSocket.getInputStream()));
            while (!MainActivity.this.isFinishing()) {
                String msg = bufferedReader.readLine();
                if (msg != null) {
                    String time = getTime(System.currentTimeMillis());
                    final String showedMsg = "server " + time + ":" + msg;
                    mHandler.obtainMessage(RECEIVE_NEW_MESSAGE, showedMsg)
                            .sendToTarget();
                }
            }
            mPrintWriter.close();
            bufferedReader.close();
            mClientSocket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void disConnectServer(){
        mIsConnectServer = false;
        if (mClientSocket != null) {
            try {
                mClientSocket.shutdownInput();
                mClientSocket.close();
                mClientSocket = null;
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void sendMessageToServer(){
        if (!mIsConnectServer){
            Toast.makeText(this,"没有连接上服务端，请重新连接",Toast.LENGTH_SHORT).show();
            return;
        }
        final String msg = mMessageEditText.getText().toString();
        if (!TextUtils.isEmpty(msg) && mPrintWriter != null) {
            mPrintWriter.println(msg);
            mMessageEditText.setText("");
            String time = getTime(System.currentTimeMillis());
            final String showedMsg = "client " + time + ":" + msg;
            TextView textView = new TextView(this);
            textView.setText(showedMsg);
            mShowLinear.addView(textView);
        }
    }
}
```
## **客户端布局代码**

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="5dp"
    >

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" >

        <LinearLayout
            android:id="@+id/show_linear"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical" >
        </LinearLayout>
    </ScrollView>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <EditText
            android:id="@+id/msg_edit_text"
            android:layout_width="0dp"
            android:layout_gravity="bottom"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            />

        <Button
            android:id="@+id/send_btn"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="发送" />
    </LinearLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">
        <Button
            android:id="@+id/connect_btn"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="连接"/>

        <Button
            android:id="@+id/disconnect_btn"
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="断开连接"/>
    </LinearLayout>
</LinearLayout>
```

## **服务端代码**　

```java
package com.example.ljd.socketserver;

import android.os.Handler;
import android.os.Message;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.text.TextUtils;
import android.widget.EditText;
import android.widget.LinearLayout;
import android.widget.TextView;

import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.sql.Date;
import java.text.SimpleDateFormat;


import butterknife.Bind;
import butterknife.ButterKnife;
import butterknife.OnClick;

public class MainActivity extends AppCompatActivity{

    @Bind(R.id.show_linear)
    LinearLayout mShowLinear;

    @Bind(R.id.msg_edit_text)
    EditText mMessageEditText;

    private ServerSocket mServerSocket;
    private PrintWriter mPrintWriter;
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            if (msg.what == 0){
                TextView textView = new TextView(MainActivity.this);
                textView.setText((String)msg.obj);
                mShowLinear.addView(textView);
            }
            super.handleMessage(msg);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        try {
            mServerSocket = new ServerSocket(8088);
        } catch (IOException e) {
            e.printStackTrace();
        }
        new Thread(new AcceptClient()).start();
    }


    @Override
    public void onDestroy() {
        ButterKnife.unbind(this);
        if (mServerSocket != null){
            try {
                mServerSocket.close();
                mServerSocket = null;
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        super.onDestroy();
    }

    @OnClick(R.id.send_btn)
    public void onClickButton() {
        final String msg = mMessageEditText.getText().toString();
        if (!TextUtils.isEmpty(msg) && mPrintWriter != null) {
            //将消息写入到流中
            mPrintWriter.println(msg);
            mMessageEditText.setText("");
            String time = getTime(System.currentTimeMillis());
            final String showedMsg = "server " + time + ":" + msg;
            TextView textView = new TextView(this);
            textView.setText(showedMsg);
            mShowLinear.addView(textView);
        }
    }

    private String getTime(long time) {
        return new SimpleDateFormat("(HH:mm:ss)").format(new Date(time));
    }

    class AcceptClient implements Runnable{

        @Override
        public void run() {
            try {
                Socket clientSocket = null;
                while (clientSocket == null){
                    clientSocket = mServerSocket.accept();
                    mPrintWriter = new PrintWriter(new BufferedWriter(
                            new OutputStreamWriter(clientSocket.getOutputStream())), true);
                }
                BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(
                        clientSocket.getInputStream()));
                while (!MainActivity.this.isFinishing()) {
                    //读取客户端发来的消息
                    String msg = bufferedReader.readLine();
                    if (msg != null) {
                        String time = getTime(System.currentTimeMillis());
                        final String showedMsg = "client " + time + ":" + msg;
                        mHandler.obtainMessage(0, showedMsg)
                                .sendToTarget();
                    }
                }
                bufferedReader.close();
                clientSocket.close();
                mPrintWriter.close();

            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```
## **服务端布局代码**

```java
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="5dp"
    >

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" >

        <LinearLayout
            android:id="@+id/show_linear"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical" >
        </LinearLayout>
    </ScrollView>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal">

        <EditText
            android:id="@+id/msg_edit_text"
            android:layout_width="0dp"
            android:layout_gravity="bottom"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            />

        <Button
            android:id="@+id/send_btn"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="发送" />
    </LinearLayout>

</LinearLayout>
```
## **添加权限**
　　在客户端与服务端应用中我们还需要添加一些权限
```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
```

# **总结**
　　在网络通信中我们除了socket我们最常用的还有一种方式那就是http通信，而http连接采用的是”请求—响应方式“，也就是说只有当客户端发出请求时，服务端才能够向客户端返回数据。在这里我们就不在详细介绍。对于Android的IPC机制就说到这里了，当然还有其他方式可以进行跨进程通信，我们可以自行研究了。
# **[源码下载](https://github.com/lijiangdong/socket)**