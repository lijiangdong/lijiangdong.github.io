---
title:    解决Retrofit文件下载进度显示问题
date: 2016-04-19 12:40
category: [Android]
tags: [Retrofit]
comments: true
---

　　在[Retrofit2.0使用详解](http://blog.csdn.net/ljd2038/article/details/51046512)这篇文章中详细介绍了retrofit的用法。并且在retrofit中我们可以通过ResponseBody进行对文件的下载。但是在retrofit中并没有为我们提供显示下载进度的接口。在项目中，若是用户下载一个文件，无法实时给用户显示下载进度，这样用户的体验也是非常差的。那么下面就介绍一下在retrofit用于文件的下载如何实时跟踪下载进度。<!--more-->
# **演示**
![这里写图片描述](http://img.blog.csdn.net/20160419123959772)
# **Retrofit文件下载进度更新的实现**
　　在retrofit2.0中他依赖于Okhttp，所以如果我们需要解决这个问题还需要从这个OKhttp来入手。在Okhttp中有一个依赖包Okio。Okio也是有square公司所开发，它是java.io和java.nio的补充，使用它更容易访问、存储和处理数据。在这里需要使用Okio中的Source类。在这里Source可以看做InputStream。对于Okio的详细使用在这里就不在介绍。下面来看一下具体实现。
　　在这里我们首先写一个接口，用于监听下载的进度。对于文件的下载，我们需要知道下载的进度，文件的总大小，以及是否操作完成。于是有了下面这样一个接口。
```java
package com.ljd.retrofit.progress;

/**
 * Created by ljd on 3/29/16.
 */
public interface ProgressListener {
    /**
     * @param progress     已经下载或上传字节数
     * @param total        总字节数
     * @param done         是否完成
     */
    void onProgress(long progress, long total, boolean done);
}
```
　　对于文件的下载我们需要重写ResponseBody类中的一些方法。
```java
package com.ljd.retrofit.progress;
import java.io.IOException;

import okhttp3.MediaType;
import okhttp3.ResponseBody;
import okio.Buffer;
import okio.BufferedSource;
import okio.ForwardingSource;
import okio.Okio;
import okio.Source;

/**
 * Created by ljd on 3/29/16.
 */
public class ProgressResponseBody extends ResponseBody {
    private final ResponseBody responseBody;
    private final ProgressListener progressListener;
    private BufferedSource bufferedSource;

    public ProgressResponseBody(ResponseBody responseBody, ProgressListener progressListener) {
        this.responseBody = responseBody;
        this.progressListener = progressListener;
    }

    @Override
    public MediaType contentType() {
        return responseBody.contentType();
    }


    @Override
    public long contentLength() {
        return responseBody.contentLength();
    }

    @Override
    public BufferedSource source() {
        if (bufferedSource == null) {
            bufferedSource = Okio.buffer(source(responseBody.source()));
        }
        return bufferedSource;
    }

    private Source source(Source source) {
        return new ForwardingSource(source) {
            long totalBytesRead = 0L;

            @Override
            public long read(Buffer sink, long byteCount) throws IOException {
                long bytesRead = super.read(sink, byteCount);
                totalBytesRead += bytesRead != -1 ? bytesRead : 0;
                progressListener.onProgress(totalBytesRead, responseBody.contentLength(), bytesRead == -1);
                return bytesRead;
            }
        };
    }
}

```
　　在上面ProgressResponseBody类中，我们计算已经读取文件的字节数，并且调用了ProgressListener接口。所以这个ProgressListener接口是在子线程中运行的。
　　下面就来看一下是如何使用这个ProgressResponseBody。
```java
package com.ljd.retrofit.progress;

import android.util.Log;

import java.io.IOException;

import okhttp3.Interceptor;
import okhttp3.OkHttpClient;

/**
 * Created by ljd on 4/12/16.
 */
public class ProgressHelper {

    private static ProgressBean progressBean = new ProgressBean();
    private static ProgressHandler mProgressHandler;

    public static OkHttpClient.Builder addProgress(OkHttpClient.Builder builder){

        if (builder == null){
            builder = new OkHttpClient.Builder();
        }

        final ProgressListener progressListener = new ProgressListener() {
            //该方法在子线程中运行
            @Override
            public void onProgress(long progress, long total, boolean done) {
                Log.d("progress:",String.format("%d%% done\n",(100 * progress) / total));
                if (mProgressHandler == null){
                    return;
                }

                progressBean.setBytesRead(progress);
                progressBean.setContentLength(total);
                progressBean.setDone(done);
                mProgressHandler.sendMessage(progressBean);

            }
        };

        //添加拦截器，自定义ResponseBody，添加下载进度
        builder.networkInterceptors().add(new Interceptor() {
            @Override
            public okhttp3.Response intercept(Chain chain) throws IOException {
                okhttp3.Response originalResponse = chain.proceed(chain.request());
                return originalResponse.newBuilder().body(
                        new ProgressResponseBody(originalResponse.body(), progressListener))
                        .build();

            }
        });

        return builder;
    }

    public static void setProgressHandler(ProgressHandler progressHandler){
        mProgressHandler = progressHandler;
    }
}
```
　　我们通过为OkhttpClient添加一个拦截器来使用我们自定义的ProgressResponseBody。并且在这里我们可以通过实现ProgressListener接口。来获取下载进度了。但是在这里依然存在一个问题，刚才说到这个ProgressListener接口运行在子线程中。也就是说在ProgressListener这个接口中我们无法进行ui操作。而我们获取文件下载的进度往往则是需要一个进度条进行ui显示。显然这并不是我们想要的结果。
　　在这个时候我们就需要使用Handler了。我们可以通过Handler将子线程中的ProgressListener的数据发送到ui线程中进行处理。也就是说我们在ProgressListener接口中的操作只是将其参数通过Handler发送出去。很显然在上面的代码中我们通过ProgressHandler来发送消息。那么就来看一下具体操作。
　　这里我们创建一个对象，用于存放ProgressListener中的参数。
```java
package com.example.ljd.retrofit.pojo;

import java.util.ArrayList;
import java.util.List;

/**
 * Created by ljd on 3/29/16.
 */
public class RetrofitBean {

    private Integer total_count;
    private Boolean incompleteResults;
    private List<Item> items = new ArrayList<Item>();

    /**
     *
     * @return
     *     The totalCount
     */
    public Integer getTotalCount() {
        return total_count;
    }

    /**
     *
     * @param totalCount
     *     The total_count
     */
    public void setTotalCount(Integer totalCount) {
        this.total_count = totalCount;
    }

    /**
     *
     * @return
     *     The incompleteResults
     */
    public Boolean getIncompleteResults() {
        return incompleteResults;
    }

    /**
     *
     * @param incompleteResults
     *     The incomplete_results
     */
    public void setIncompleteResults(Boolean incompleteResults) {
        this.incompleteResults = incompleteResults;
    }

    /**
     *
     * @return
     *     The items
     */
    public List<Item> getItems() {
        return items;
    }
}
```
　　然后我们在创建一个ProgressHandler类。
```java
package com.ljd.retrofit.progress;

import android.os.Handler;
import android.os.Looper;
import android.os.Message;

/**
 * Created by ljd on 4/12/16.
 */
public abstract class ProgressHandler {

    protected abstract void sendMessage(ProgressBean progressBean);

    protected abstract void handleMessage(Message message);

    protected abstract void onProgress(long progress, long total, boolean done);

    protected static class ResponseHandler extends Handler{

        private ProgressHandler mProgressHandler;
        public ResponseHandler(ProgressHandler mProgressHandler, Looper looper) {
            super(looper);
            this.mProgressHandler = mProgressHandler;
        }

        @Override
        public void handleMessage(Message msg) {
            mProgressHandler.handleMessage(msg);
        }
    }

}
```
　　上面的ProgressHandler他是一个抽象类。在这里我们需要通过Handler对象进行发送和处理消息。于是定义了两个抽象方法sendMessage和handleMessage。之后又定义了一个抽象方法onProgress来处理下载进度的显示，而这个onProgress则是我们需要在ui线程进行调用。最后创建了一个继承自Handler的ResponseHandler内部类。为了避免内存泄露我们使用static关键字。
　　下面来创建一个DownloadProgressHandler类，他继承于ProgressHandler，用来发送和处理消息。
```java
package com.ljd.retrofit.progress;


import android.os.Looper;
import android.os.Message;

/**
 * Created by ljd on 4/12/16.
 */
public abstract class DownloadProgressHandler extends ProgressHandler{

    private static final int DOWNLOAD_PROGRESS = 1;
    protected ResponseHandler mHandler = new ResponseHandler(this, Looper.getMainLooper());

    @Override
    protected void sendMessage(ProgressBean progressBean) {
        mHandler.obtainMessage(DOWNLOAD_PROGRESS,progressBean).sendToTarget();

    }

    @Override
    protected void handleMessage(Message message){
        switch (message.what){
            case DOWNLOAD_PROGRESS:
                ProgressBean progressBean = (ProgressBean)message.obj;
                onProgress(progressBean.getBytesRead(),progressBean.getContentLength(),progressBean.isDone());
        }
    }
}
```
　　在这里我们接收到消息以后调用抽象方法onProgress，这样一来我们只需要创建一个DownloadProgressHandler对象，实现onProgress即可。
　　对于上面的分析，下面我们就来看一下是如何使用的。
```java
package com.example.ljd.retrofit.download;

import android.app.ProgressDialog;
import android.os.Environment;
import android.os.Looper;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;

import com.example.ljd.retrofit.R;
import com.ljd.retrofit.progress.DownloadProgressHandler;
import com.ljd.retrofit.progress.ProgressHelper;

import java.io.BufferedInputStream;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;

import butterknife.ButterKnife;
import butterknife.OnClick;
import okhttp3.OkHttpClient;
import okhttp3.ResponseBody;
import retrofit2.Call;
import retrofit2.Callback;
import retrofit2.Response;
import retrofit2.Retrofit;
import retrofit2.adapter.rxjava.RxJavaCallAdapterFactory;
import retrofit2.converter.gson.GsonConverterFactory;

public class DownloadActivity extends AppCompatActivity {


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_download);
        ButterKnife.bind(this);

    }

    @Override
    protected void onDestroy() {
        ButterKnife.unbind(this);
        super.onDestroy();
    }

    @OnClick(R.id.start_download_btn)
    public void onClickButton(){
        retrofitDownload();
    }

    private void retrofitDownload(){
        //监听下载进度
        final ProgressDialog dialog = new ProgressDialog(this);
        dialog.setProgressNumberFormat("%1d KB/%2d KB");
        dialog.setTitle("下载");
        dialog.setMessage("正在下载，请稍后...");
        dialog.setProgressStyle(ProgressDialog.STYLE_HORIZONTAL);
        dialog.setCancelable(false);
        dialog.show();

        Retrofit.Builder retrofitBuilder = new Retrofit.Builder()
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                .baseUrl("http://msoftdl.360.cn");
        OkHttpClient.Builder builder = ProgressHelper.addProgress(null);
        DownloadApi retrofit = retrofitBuilder
                .client(builder.build())
                .build().create(DownloadApi.class);

        ProgressHelper.setProgressHandler(new DownloadProgressHandler() {
            @Override
            protected void onProgress(long bytesRead, long contentLength, boolean done) {
                Log.e("是否在主线程中运行", String.valueOf(Looper.getMainLooper() == Looper.myLooper()));
                Log.e("onProgress",String.format("%d%% done\n",(100 * bytesRead) / contentLength));
                Log.e("done","--->" + String.valueOf(done));
                dialog.setMax((int) (contentLength/1024));
                dialog.setProgress((int) (bytesRead/1024));

                if(done){
                    dialog.dismiss();
                }
            }
        });

        Call<ResponseBody> call = retrofit.retrofitDownload();
        call.enqueue(new Callback<ResponseBody>() {
            @Override
            public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {
                try {
                    InputStream is = response.body().byteStream();
                    File file = new File(Environment.getExternalStorageDirectory(), "12345.apk");
                    FileOutputStream fos = new FileOutputStream(file);
                    BufferedInputStream bis = new BufferedInputStream(is);
                    byte[] buffer = new byte[1024];
                    int len;
                    while ((len = bis.read(buffer)) != -1) {
                        fos.write(buffer, 0, len);
                        fos.flush();
                    }
                    fos.close();
                    bis.close();
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void onFailure(Call<ResponseBody> call, Throwable t) {

            }
        });

    }
}
```
# **总结**
　　对于上面的实现我们可以看出是通过OkhttpClient实现的。也正是由于在retrofit2.0中它依赖于OkHttp，因此对于OkHttp的功能retrofit也都具备。利用这一特性，我们可以通过定制OkhttpClient来配置我们的retrofit。　　
# **[源码下载](https://github.com/lijiangdong/retrofit-example)**