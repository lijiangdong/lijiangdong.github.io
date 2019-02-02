---
title:   AsyncTask使用以及源码分析
date: 2016-03-19 16:38
category: [Android]
tags: [AsyncTask]
comments: true
---
　　在Android中，我们需要进行一些耗时的操作，会将这个操作放在子线程中进行。在子线程操作完成以后我们可以通过Handler进行发送消息，通知UI进行一些更新操作（具体使用及其原理可以查看[Android的消息机制——Handler的工作过程](http://blog.csdn.net/ljd2038/article/details/50889754)这篇文章）。当然为了简化我们的操作，在Android1.5以后为我们提供了AsyncTask类，它能够将子线程处理完成后的结果返回到UI线程中，之后我们便可以根据这些结果进行一列的UI操作了。<!--more-->
# **AsyncTask的使用方法**
　　实际上AsyncTask内部也就是对Handler和线程池进行了一次封装。它是一个轻量级的异步任务类，它的后台任务在线程池中进行。之后我们可以将任务执行的结果传递给主线程，这时候我们就可以在主线程中操作UI了。
　　AsyncTask他是一个抽象的泛型类，所以我们创建一个子类，来实现AsyncTask中的抽象方法。AsyncTask中提供了三个泛型参数，下面我们就来看一下这三个泛型参数.
　　1.  Params:在执行AsyncTask时所传递的参数，该参数在后台线程中使用。
　　2.  Progress:后台任务执行进度的类型
　　3.  Result:后台任务执行完成后返回的结果类型。
　　对于以上三个泛型参数我们不需要使用的时候，可以使用Void来代替。与Activity生命周期类似，在AsyncTask中也为我们提供了一些方法，我们通过重写这几个方法来完成整个异步任务。我们主要使用的方法有一下四个:
　　1.  onPreExecute():该方法在异步任务工作之前执行，主要用于一些参数或者UI的初始化操作。
　　2.  doInBackground(Params... params):该方法在线程池中执行，params参数表示异步任务时输入的参数。在这个方法中我们通过publishProgress来通知任务进度。
　　3.  onProgressUpdate(Progress... values):当后台任务的进度发生改变的时候会调用该方法，我们可以再这个方法中进行UI的进度展示。values参数表示任务进度。
　　4.  postResult(Result result):在异步任务完成之后执行，result参数为异步任务执行完以后所返回的结果。
　　在上面四个方法中只有doInBackground在子线程中运行，其余都三个方法都是在主线程中运行的。其中的...表示参数的数量不定，是一种数组类型的参数。
　　下面我们就来写一个例子来看一下AsyncTask的用法，在这里我们就一个下载的功能，从网络上下载两个文件。我们先来看一下效果演示。
## **效果演示**
![这里写图片描述](http://img.blog.csdn.net/20160323105746897)
## **代码分析**
　　由于我们做的下载任务，首先我们就得添加访问网络权限以及一些sd卡相关的权限。
```xml
<!-- 在SD卡中创建与删除文件权限 -->
<uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/>
<!-- 向SD卡写入数据权限 -->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<!-- 授权访问网络 -->
<uses-permission android:name="android.permission.INTERNET"/>
```
　　下面我们来看一下Activity代码。
```java
package com.example.ljd.asynctask;

import android.app.ProgressDialog;
import android.os.AsyncTask;
import android.os.Environment;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;

import java.io.BufferedInputStream;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;

public class MainActivity extends AppCompatActivity implements View.OnClickListener{

    private DownloadAsyncTask mDownloadAsyncTask;

    private Button mButton;
    private String[] path = {
            "http://msoftdl.360.cn/mobilesafe/shouji360/360safesis/360MobileSafe_6.2.3.1060.apk",
            "http://dlsw.baidu.com/sw-search-sp/soft/7b/33461/freeime.1406862029.exe",
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mButton = (Button)findViewById(R.id.button);
        mButton.setOnClickListener(this);
    }

    @Override
    protected void onDestroy() {
        if (mDownloadAsyncTask != null){
            mDownloadAsyncTask.cancel(true);
        }
        super.onDestroy();
    }

    @Override
    public void onClick(View v) {

        mDownloadAsyncTask = new DownloadAsyncTask();
        mDownloadAsyncTask.execute(path);
    }
    
    class DownloadAsyncTask extends AsyncTask<String,Integer,Boolean>{

        private ProgressDialog mPBar;
        private int fileSize;     //下载的文件大小
        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            mPBar = new ProgressDialog(MainActivity.this);
            mPBar.setProgressNumberFormat("%1d KB/%2d KB");
            mPBar.setTitle("下载");
            mPBar.setMessage("正在下载，请稍后...");
            mPBar.setProgressStyle(ProgressDialog.STYLE_HORIZONTAL);
            mPBar.setCancelable(false);
            mPBar.show();
        }

        @Override
        protected Boolean doInBackground(String... params) {
            //下载图片
            for (int i=0;i<params.length;i++){
                try{
                    if(Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)){
                        URL url = new URL(params[i]);
                        HttpURLConnection conn =  (HttpURLConnection) url.openConnection();
                        //设置超时时间
                        conn.setConnectTimeout(5000);
                        //获取下载文件的大小
                        fileSize = conn.getContentLength();
                        InputStream is = conn.getInputStream();
                        //获取文件名称
                        String fileName = path[i].substring(path[i].lastIndexOf("/") + 1);
                        File file = new File(Environment.getExternalStorageDirectory(), fileName);
                        FileOutputStream fos = new FileOutputStream(file);
                        BufferedInputStream bis = new BufferedInputStream(is);
                        byte[] buffer = new byte[1024];
                        int len ;
                        int total = 0;
                        while((len =bis.read(buffer))!=-1){
                            fos.write(buffer, 0, len);
                            total += len;
                            publishProgress(total);
                            fos.flush();
                        }
                        fos.close();
                        bis.close();
                        is.close();
                    }
                    else{
                        return false;
                    }
                }catch (IOException e){
                    e.printStackTrace();
                    return false;
                }
            }

            return true;
        }

        @Override
        protected void onPostExecute(Boolean aBoolean) {
            super.onPostExecute(aBoolean);
            mPBar.dismiss();
            if (aBoolean){
                Toast.makeText(MainActivity.this,"下载完成",Toast.LENGTH_SHORT).show();
            } else {
                Toast.makeText(MainActivity.this,"下载失败",Toast.LENGTH_SHORT).show();
            }
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
            mPBar.setMax(fileSize / 1024);
            mPBar.setProgress(values[0]/1024);
        }
    }

}
```
　　在以上代码中有几点我们需要注意一下。
　　1.  AsyncTask中的execute方法必须在主线程中执行。
　　2.  每个AsyncTask对象只能执行一次execute方法。
　　3.  当我们的Activity销毁的时候需要进行取消操作，其中boolean型的参数mayInterruptIfRunning表示是否中断后台任务。
　　最后的布局代码就很简单了，只有一个Button而已。
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="@dimen/activity_vertical_margin"
    android:orientation="vertical"
    tools:context="com.example.ljd.asynctask.MainActivity">

    <Button
        android:id="@+id/button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="download"/>
</LinearLayout>
```
　　这里还有一点需要说明一下，由于在不同的Android版本中对AsyncTask进行了多次修改，所以当我们通过多个AsyncTask对象执行多次execute方法的时候，它们执行顺序是串行还是并行根据系统不同的版本而出现差异，这里就不再具体分析。
# **AsyncTask源码分析**
　　在这里我们采用Android6.0中的AsyncTask源码进行分析，对于不同系统的AsyncTask代码会有一定的差异。当创建一个AsyncTask对象以后我们便可以通过execute方法来执行整个任务。那就在这里首先看一下execute方法中的代码。
```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```
　　可以看到这个execute方法中的代码是如此的简单，只是执行了executeOnExecutor方法并返回AsyncTask对象。下面我们就来看一下executeOnExecutor这个方法。
```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute();

    mWorker.mParams = params;
    exec.execute(mFuture);

    return this;
}
```
　　在这个方法内我们首先对AsyncTask所执行的状态进行判断。如果AsyncTask正在执行任务或者任务已经知己完成就会给我们抛出异常，也就解释上面所说的每个AsyncTask对象只能执行一次execute方法。紧接着将当前任务状态修改为正在运行以后便开始执行onPreExecute方法。这也说明了在上面我们重写的四个方法中onPreExecute方法最先指向的。下面我们在看一下mWorker和mFuture这两个全局变量。
　　mFuture是FutureTask对象，而mWorker是WorkerRunnable对象，WorkerRunnable是AsyncTask中的一个实现Callable接口的抽象内部类，在WorkerRunnable中只定义了一个Params[]。
```java
private final WorkerRunnable<Params, Result> mWorker;
private final FutureTask<Result> mFuture;

......

private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}
```
　　mFuture和mWorker是在AsyncTask的构造方法中初始化的。我们看一下AsyncTask的构造方法。
```java
public AsyncTask() {
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);

            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            Result result = doInBackground(mParams);
            Binder.flushPendingCommands();
            return postResult(result);
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```
　　在这里首先创建一个WorkerRunnable对象mWorker，并且实现了Callable接口的call方法，对于这个call方面里面的内容在后面会进行详细说明。然后我们再通过mWorker，创建一个FutureTask对象mFuture，并且重写里面的done方法，当我们调用AsyncTask里面的cancel方法时，在FutureTask中会调用这个done方法。在这里介绍完mWorker和mFuture后我们再回过头来看executeOnExecutor方法。在这里通过我们传入的params参数初始化了mWorker中的mParams。而下面的exec则是在execute里面传入的sDefaultExecutor，并且执行sDefaultExecutor的execute方法。下面我们再来看一下这个sDefaultExecutor对象。
```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE = 1;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};

private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);

public static final Executor THREAD_POOL_EXECUTOR
        = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
                
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

......

private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

......

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```
　　在这里首先整体介绍一下这段代码。这段代码的主要功能就是创建了两个线程池SerialExecutor和THREAD_POOL_EXECUTOR。SerialExecutor线程池用于对任务的排队，而THREAD_POOL_EXECUTOR则是用来执行任务。而sDefaultExecutor就是SerialExecutor线程池。
　　我们进入SerialExecutor代码中看一下里面的内容。SerialExecutor中的execute方法内的参数Runnable就是我们传入的mFuture。在execute方法中创建了一个Runnable对象，并将该队列插入对象尾部。这个时候如果是第一次执行任务。mActive必然为null，这时候便调用scheduleNext方法从队列中取出Runnable对象，并且通过THREAD_POOL_EXECUTOR线程池执行任务。我们可以看到在队列中的Runnable的run方法中首先执行mFuture中的run方法，执行完之后又会调用scheduleNext方法，从队列中取出Runnable执行，直到所有的Runnable执行完为止。下面就来看一下在线城池中是如何执行这个Runnable的。从上面代码可以看出，在线城池中执行Runnable其实最核心的部分还是执行mFuture的run方法。那就来看一下这个mFuture中的run方法。
```java
public void run() {
    if (state != NEW ||
        !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```
　　在这个方法中的callable对象就是我们AsyncTask中的mWorker对象。在这里面也正是执行mWorker中的call方法来完成一些耗时任务，于是我们就能想到我重写的doInBackground应该就在这个call方法中执行了。现在再回到AsyncTask的构造方法中看一下这个mWorker中的call方法。
```java
public Result call() throws Exception {
    mTaskInvoked.set(true);

    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    //noinspection unchecked
    Result result = doInBackground(mParams);
    Binder.flushPendingCommands();
    return postResult(result);
}
```
　　这里我们很清楚的看到它执行我们重写的doInBackground方法，并且将返回的结果传到postResult方法中。下面就在看一下这个postResult方法。
```java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```
　　对与这个postResult方法里面也就是在我们子线程的任务处理完成之后通过一个Handler对象将Message发送到主线程中，并且交由主线程处理。AsyncTaskResult对象中存放的是子线程返回的结果以及当前AsyncTask对象。下面再看一下这和Handler中所处理了哪些事情。
```java
private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
　　可以看到在这个Handler的handleMessage中会接受到两种Message。在MESSAGE_POST_PROGRESS这个消息中主要是通过publishProgress方法将子线程执行的进度发送到主线程中并且通过onProgressUpdate方法来更新进度条的显示。在MESSAGE_POST_RESULT这个消息中通过当前的AsyncTask对象调用了finish方法，那么就来看一下这个finish方法。
```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
　　这时候就可以看出若是我们取消了AsyncTask的话就不在执行onPostExecute方法，而是执行onCancelled方法，所以我们可以通过重写onCancelled方法来执行取消时我们需要处理的一些操作。当然若是AsyncTask没有被取消，这时候就回执行onPostExecute方法。到这里整个AsyncTask任务也就完成了。
# **总结**
　　在上面的SerialExecutor线程池中可以看出，当有多个异步任务同时执行的时候，它们执行的顺序是串行的，会按照任务创建的先后顺序进行一次执行。如果我们希望多个任务并发执行则可以通过AsyncTask中的setDefaultExecutor方法将线程池设为THREAD_POOL_EXECUTOR即可。
　　对于AsyncTask的在不同版本之间的差异不得不提一下。在Android1.6，AsyncTask采用的是串行执行任务，在Android1.6的时候采用线程池处理并行任务，而在3.0以后才通过SerialExecutor线程池串行处理任务。在Android4.1之前AsyncTask类必须在主线程中，但是在之后的版本中就被系统自动完成。而在Android5.0的版本中会在ActivityThread的main方法中执行AsyncTask的init方法，而在Android6.0中又将init方法删除。所以在使用这个AsyncTask的时候若是适配更多的系统的版本的话，使用的时候就要注意了。
# **[源码下载](https://github.com/lijiangdong/asynctask-example)**