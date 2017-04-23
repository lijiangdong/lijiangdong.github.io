---
title:   Android的消息机制—Handler的工作过程
date: 2016-03-14 20:54
category: [android]
tags: [Handler]
comments: true
---
# **综述**
　　在Android系统中，出于对性能优化的考虑，对于Android的UI操作并不是线程安全的。也就是说若是有多个线程来操作UI组件，就会有可能导致线程安全问题。所以在Android中规定只能在UI线程中对UI进行操作。这个UI线程是在应用第一次启动时开启的，也称之为主线程（Main Thread），该线程专门用来操作UI组件，在这个UI线程中我们不能进行耗时操作，否则就会出现ANR(Application Not Responding)现象。如果我们在子线程中去操作UI，那么程序就回给我们抛出异常。这是因为在ViewRootImpl中对操作UI的线程进行检查。如果操作UI的线程不是主线程则抛出异常（对于在检查线程之前在非UI线程已经操作UI组件的情况除外）。所以这时候我们若是在子线程中更新UI的话可以通过Handler来完成这一操作。<!--more-->
# **Handler用法简介**
　　在开发中，我们对Handler的使用也基本上算是家常便饭了。在这里我们就简单的说一下Handler的几种用法示例，就不在具体给出Demo进行演示。在这里我们只针对后面这一种情形来看一下Handler的使用：在子线程完成任务后通过Handler发送消息，然后在主线程中去操作UI。
　　一般来说我们会在主线程中创建一个Handler的匿名内部类，然后重写它的handleMessage方法来处理我们的UI操作。代码如下所示。
```java
private Handler mHandler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what){
	       //根据msg.what的值来处理不同的UI操作
            case WHAT:
                break;
            default:
                super.handleMessage(msg);
                break;
        }
        
    }
};
```
　　我们还可以不去创建一个Handler的子类对象，直接去实现Handler里的CallBack接口，Handler通过回调CallBack接口里的handleMessage方法从而实现对UI的操作。
```java
private Handler mHandler = new Handler(new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {

        return false;
    }
});
```
　　然后我们就可以在子线程中发送消息了。
```java
new Thread(new Runnable() {
    @Override
    public void run() {
        //子线程任务
        ...
        //发送方式一 直接发送一个空的Message
        mHandler.sendEmptyMessage(WHAT);
        //发送方式二 通过sendToTarget发送
        mHandler.obtainMessage(WHAT,arg1,arg2,obj).sendToTarget();
        //发送方式三 创建一个Message 通过sendMessage发送
        Message message = mHandler.obtainMessage();
        message.what = WHAT;
        mHandler.sendMessage(message);
    }
}).start();
```
　　在上面我们给出了三种不同的发送方式，当然对于我们还可以通过sendMessageDelayed进行延时发送等等。如果我们的Handler只需要处理一条消息的时候，我们可以通过post一系列方法进行处理。
```java
private Handler mHandler = new Handler();

new Thread(new Runnable() {
    @Override
    public void run() {
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                //UI操作
                ...
            }
        });
    }
}).start();
```
　　在Handler中处理UI操作时，上面的Handler对象必须是在主线程创建的。如果我们想在子线程中去new一个Handler对象的话，就需要为Handler指定Looper。
```java
private Handler mHandler；

new Thread(new Runnable() {
    @Override
    public void run() {
        mHandler = new Handler(Looper.getMainLooper()){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                //UI操作
				...
            }
        };

    }
}).start();
```
　　对于这个Looper是什么，下面我们会详细介绍。对于Handler的使用依然存在一个问题，由于我们创建的Handler是一个匿名内部类，他会隐式的持有外部类的一个对象（当然内部类也是一样的），而往往在子线程中是一个耗时的操作，而这个线程也持有Handler的引用，所以这个子线程间接的持有这个外部类的对象。我们假设这个外部类是一个Activity，而有一种情况就是我们的Activity已经销毁，而子线程仍在运行。由于这个线程持有Activity的对象，所以，在Handler中消息处理完之前，这个Activity就一直得不到回收，从而导致了内存泄露。如果内存泄露过多，则会导致OOM（OutOfMemory），也就是内存溢出。那么有没有什么好的解决办法呢？
　　我们可以通过两种方案来解决，第一种方法我们在Activity销毁的同时也杀死这个子线程，并且将相对应的Message从消息队列中移除；第二种方案则是我们创建一个继承自Handler的静态内部类。因为静态内部类不会持有外部类的对象。可是这时候我们无法去访问外部类的非静态的成员变量，也就无法对UI进行操作。这时候我们就需要在这个静态内部类中使用弱引用的方式去指向这个Activity对象。下面我们看一下示例代码。
```java
static class MyHandler extends Handler{
    private final WeakReference<MyActivity> mActivity;

    public MyHandler(MyActivity activity){
        super();
        mActivity = new WeakReference<MyActivity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        MyActivity myActivity = mActivity.get();
        if (myActivity!=null){
            myActivity.textView.setText("123456789");
        }
    }
}
```
# **Handler工作过程**
　　在上面我们简单的说明了Handler是如何使用的。那么现在我们就来看一下这个Handler是如何工作的。在Android的消息机制中主要是由Handler,Looper,MessageQueue,Message等组成。而Handler得运行依赖后三者。那么我们就来看一下它们是如何联系在一起的。
## **Looper**
　　在一个Android应用启动的时候，会创建一个主线程，也就是UI线程。而这个主线程也就是ActivityThread。在ActivityThread中有一个静态的main方法。这个main方法也就是我们应用程序的入口点。我们来看一下这个main方法。
```java
public static void main(String[] args) {
	
	......
	
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);
	
	......
	
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
　　在上面代码中通过prepareMainLooper方法为主线程创建一个Looper,而loop则是开启消息循环。从上面代码我们可以猜想到在loop方法中应该存在一个死循环，否则给我们抛出RuntimeException。也就是说主线程的消息循环是不允许被退出的。下面我们就来看一下这个Looper类。
　　首先我们看一下Looper的构造方法。
```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
　　在这个构造方法中创建了一个消息队列。并且保存当前线程的对象。其中quitAllowed参数表示是否允许退出消息循环。但是我们注意到这个构造方法是private，也就是说我们自己不能手动new一个Looper对象。那么我们就来看一下如何创建一个Looper对象。之后在Looper类中我们找到下面这个方法。
```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
　　在这里新建了一个Looper对象，然后将这个对象保存在ThreadLocal中，当我们下次需要用到Looper的之后直接从这个sThreadLocal中取出即可。在这里简单说明一下ThreadLocal这个类，ThreadLocal它实现了本地变量存储，我们将当前线程的数据存放在ThreadLocal中，若是有多个变量共用一个ThreadLocal对象，这时候在当前线程只能获取该线程所存储的变量，而无法获取其他线程的数据。在Looper这个类中为我们提供了myLooper来获取当前线程的Looper对象。从上面的方法还能够看出，一个线程只能创建一次Looper对象。然后我们在看一下这个prepare在哪里被使用的。
```java
public static void prepare() {
    prepare(true);
}

public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```
　　prepare方法：这个是用于在子线程中创建一个Looper对象，在子线程中是可以退出消息循环的。
　　prepareMainLooper方法：这个方法在上面的ActivityThread中的main方法中我们就已经见到过了。它是为主线程创建一个Looper，在主线程创建Looper对象中，就设置了不允许退出消息循环。并且将主线程的Looper保存在sMainLooper中，我们可以通过getMainLooper方法来获取主线程的Looper。
　　在ActivityThread中的main方法中除了创建一个Looper对象外，还做了另外一件事，那就是通过loop方法开启消息循环。那么我们就来看一下这个loop方法做了什么事情。
```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        msg.recycleUnchecked();
    }
}
```
　　第2~6行：获取当前线程中的Looper，并从Looper中获得消息队列。
　　第10~11行：确保当前线程属于当前进程，并且记录真实的token。clearCallingIdentity的实现是在native层，对于具体是如何实现的就不在进行分析。
　　第14~18行：从消息队列中取出消息，并且只有当取出的消息为空的时候才会跳出循环。
　　第27行：将消息重新交由Handler处理。
　　第35~42行：确保调用过程中线程没有被销毁。
　　第44行：对消息进行回收处理。
　　和我们刚才猜想的一样，在loop中确实存在一个死循环，而唯一退出该循环的方式就是消息队列返回的消息为空。然后我们通过消息队列的next()方法获得消息。msg.target是发送消息的Handler，通过Handler中的dispatchMessage方法又将消息交由Handler处理。消息处理完成之后便对消息进行回收处理。在这里我们也能够通过quit和quitSafely退出消息循环。
```java
public void quit() {
    mQueue.quit(false);
}

public void quitSafely() {
    mQueue.quit(true);
}
```
　　我们可以看出对于消息循环的退出，实际上就是调用消息队列的quit方法。这时候从MessageQueue的next方法中取出的消息也就是null了。下面我们来看一下这个MessageQueue。

## **MessageQueue**
　　MessageQueue翻译为消息队里，在这个消息队列中是采用单链表的方式实现的，提高插入删除的效率。对于MessageQueue在这里我们也只看一下它的入队和出队操作。
　　MessageQueue入队方法。
```java
boolean enqueueMessage(Message msg, long when) {
    
    ......
    
    synchronized (this) {
    
		......
		
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
　　在这里我们简单说一下这个入队的方法。消息的插入过程是在第13~36行完成了。在这里首先判断首先判断消息队列里有没有消息，没有的话则将当前插入的消息作为队头，并且这时消息队列如果处于等待状态的话则将其唤醒。若是在中间插入，则根据Message创建的时间进行插入。
　　MessageQueue出队方法。
```java
Message next() {

    ......

    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            ......
        }

        .....
    }
}
```
　　第11行：nativePollOnce方法在native层，若是nextPollTimeoutMillis为-1，这时候消息队列处于等待状态。
　　第25~42行：按照我们设置的时间取出消息。
　　第43~45行：这时候消息队列中没有消息，将nextPollTimeoutMillis设为-1，下次循环消息队列则处于等待状态。
　　第48~52行：退出消息队列，返回null，这时候Looper中的消息循环也会终止。
　　最后我们在看一下退出消息队列的方法：
```java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```
　　从上面我们可以看到主线程的消息队列是不允许被退出的。并且在这里通过将mQuitting设为true从而退出消息队列。也使得消息循环被退出。到这里我们介绍了Looper和MessageQueue，就来看一下二者在Handler中的作用。
## **Handler**
　　在这里我们首先看一下Handler的构造方法。
```java
public Handler(Callback callback, boolean async) {
	
	......
	
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
　　从这个构造方法中我们可以看出在一个没有创建Looper的线程中是无法创建一个Handler对象的。所以说我们在子线程中创建一个Handler时首先需要创建Looper，并且开启消息循环才能够使用这个Handler。但是在上面的例子中我们确实在子线程中new了一个Handler对象。我们再来看一下上面那个例子的构造方法。
```java
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
　　在这个构造方法中我们为Handler指定了一个Looper对象。也就说在上面的例子中我们在子线程创建的Handler中为其指定了主线程的Looper，也就等价于在主线程中创建Handler对象。下面我们就来看一下Handler是如何发送消息的。
　　对于Handler的发送方式可以分为post和send两种方式。我们先来看一下这个post的发送方式。
```java
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);
}
```
　　在这里很明显可以看出来，将post参数中的Runnable转换成了Message对象，然后还是通过send方式发出消息。我们就来看一下这个getPostMessage方法。
```java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
　　在这里也是将我们实现的Runnable交给了Message对象的callback属性。并返回该Message对象。
　　既然post发送也是由send发送方式进行的，那么我们一路找下去，最终消息的发送交由sendMessageAtTime方法进行处理。我们就来看一下这个sendMessageAtTime方法。
```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
　　然后再来看一下enqueueMessage方法。
```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
　　到这里我们可以看出来了所谓通过Handler发送消息只不过是在Looper创建的消息队列中插入一条消息而已。而在Looper中只不过通过loop取出消息，然后交由Handler中的dispatchMessage方发进行消息分发处理。下面我们来看一下dispatchMessage方法。
```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
　　这里面的逻辑也是非常的简单，msg.callback就是我们通过post里的Runnable对象。而handleCallback也就是去执行Runnable中的run方法。
```java
private static void handleCallback(Message message) {
    message.callback.run();
}
```
　　mCallback就是我们所实现的回调接口。最后才是对我们继承Handler类中重写的handleMessage进行执行。可见其中的优先级顺序为post>CallBack>send;
　　到这里我们对整个Handler的工作过程也就分析完了。现在我们想要通过主线程发送消息给子线程，然后由子线程接收消息并进行处理。这样一种操作也就很容易实现了。我们来看一下怎么实现。
```java
package com.example.ljd.myapplication;

import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.util.Log;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Button;


public class MyActivity extends AppCompatActivity {

    private final String TAG = "MyActivity";
    public Handler mHandler;
    public Button button;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        button = (Button) findViewById(R.id.send_btn);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (mHandler != null){
                    mHandler.obtainMessage(0,"你好，我是从主线程过来的").sendToTarget();
                }
            }
        });
        new Thread(new Runnable() {
            @Override
            public void run() {
                //在子线程中创建一个Looper对象
                Looper.prepare();
                mHandler = new Handler(){
                    @Override
                    public void handleMessage(Message msg) {
                        if (msg.what == 0){
                            Log.d(TAG,(String)msg.obj);
                        }
                    }
                };
                //开启消息循环
                Looper.loop();
            }
        }).start();

    }
}
```
　　点击按钮我们看一下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160316160105126)
　　
# **总结**
　　在这里我们重新整理一下我们的思路，看一下这个Handler的整个工作流程。在主线程创建的时候为主线程创建一个Looper,创建Looper的同时在Looper内部创建一个消息队列。而在创键Handler的时候取出当前线程的Looper，并通过该Looper对象获得消息队列，然后Handler在子线程中发送消息也就是在该消息队列中添加一条Message。最后通过Looper中的消息循环取得这条Message并且交由Handler处理。
