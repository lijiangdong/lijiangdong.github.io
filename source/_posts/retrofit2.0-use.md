---
title:   Retrofit2.0使用详解
date: 2016-04-03 01:09
category: [Android]
tags: [Retrofit]
comments: true
---

　　retrofit是由square公司开发的。square在github上发布了很多优秀的Android开源项目。例如:[otto](https://github.com/square/otto)(事件总线),[leakcanary](https://github.com/square/leakcanary)(排查内存泄露),[android-times-square](https://github.com/square/android-times-square)(日历控件),[dagger](https://github.com/square/dagger)(依赖注入),[picasso](https://github.com/square/picasso)(异步加载图片),[okhttp](https://github.com/square/okhttp)(网络请求),[retrofit](https://github.com/square/retrofit)(网络请求)等等。更多square上的开源项目我们可以去[square的GitHub](https://github.com/square)进行查看。这次就来介绍一下retrofit的一些基本用法。retrofit是REST安卓客户端请求库。使用retrofit可以进行GET，POST，PUT，DELETE等请求方式。下面就来看一下retrofit的基本用法。<!--more-->
# **Retrofit使用方法**
　　由于retrofit2.0与先前版本的差别还是比较大，对于不同版本之间的差异在这里就不在进行详细区别。下面的例子也是针对于retrofit2.0进行介绍的。retrofit2.0它依赖于OkHttp,而且这部分也不再支持替换。在这里我们也不需要显示的导入okHttp,在retrofit中已经导入okhttp3。
```xml
<dependency>
  <groupId>com.squareup.okhttp3</groupId>
  <artifactId>mockwebserver</artifactId>
  <scope>test</scope>
</dependency>
```
　　在下面的例子当中采用与GitHub一些相关api进行演示。在这里首先需要添加访问网络的权限。
```xml
<uses-permission android:name="android.permission.INTERNET"/>
```
## **简单示例**
### **添加Gradle依赖项**
　　在这里我们最好查看一下[retrofit的官网](http://square.github.io/retrofit/)添加最新依赖。
```java
compile 'com.squareup.retrofit2:retrofit:2.0.1'
```
### **创建API接口**
　　在retrofit中通过一个Java接口作为http请求的api接口。
```java
public interface GitHubApi {

    @GET("repos/{owner}/{repo}/contributors")
    Call<ResponseBody> contributorsBySimpleGetCall(@Path("owner") String owner, @Path("repo") String repo);
}
```
### **创建retrofit实例**
　　在这里baseUrl是在创建retrofit实力的时候定义的，我们也可以在API接口中定义完整的url。在这里建议在创建baseUrl中以"/"结尾，在API中不以"/"开头和结尾。
```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.github.com/")
        .build();
```
### **调用API接口**
　　在调用API接口请求后，获得一个json字符串，通过Gson进行解析，获得login以及contributions。
```java
GitHubApi repo = retrofit.create(GitHubApi.class);

    Call<ResponseBody> call = repo.contributorsBySimpleGetCall(mUserName, mRepo);
call.enqueue(new Callback<ResponseBody>() {
    @Override
    public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {
        try {
            Gson gson = new Gson();
            ArrayList<Contributor> contributorsList = gson.fromJson(response.body().string(), new TypeToken<List<Contributor>>(){}.getType());
            for (Contributor contributor : contributorsList){
                Log.d("login",contributor.getLogin());
                Log.d("contributions",contributor.getContributions()+"");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onFailure(Call<ResponseBody> call, Throwable t) {

    }
});
```
### **效果展示**
　　这样就完成了一个http请求，上面请求的完整地址为：https://api.github.com/repos/square/retrofit/contributors
　　然后我们看一下运行结果:
![这里写图片描述](http://img.blog.csdn.net/20160331234336311)
## **取消请求**
　　我们可以终止一个请求。终止操作是对底层的httpclient执行cancel操作。即使是正在执行的请求，也能够立即终止。
```java
call.cancel();
```
## **转换器**
　　在上面的例子中通过获取ResponseBody后，我们自己使用Gson来解析接收到的Json格式数据。在Retrofit中当创建一个Retrofit实例的时候可以为其添加一个Json转换器，这样就会自动将Json格式的响应体转换为所需要的Java对象。那么先来看一下如何根据已有的Json格式数据如何生成Java对象。当然我们可以根据已知的数据手动创建Java对象，也可以通过工具更具Json格式为我们自动生成Java对象。
### **自动生成Java对象**
　　在这里介绍两种根据Json数据自动生成Java对象的工具。
#### **jsonschema2pojo**
　　可以通过访问[jsonschema2pojo](http://www.jsonschema2pojo.org/)网站。先来看一下它的使用方法。
![这里写图片描述](http://img.blog.csdn.net/20160402093530309)
　　上面配置中所选注解若是使用的Gson解析，可以选择Gson，当然没有也是可以的。对于@Generated注解若是需要保留的话添加如下依赖，也可以直接删除@Generated注解，没有任何影响。
```java
compile 'org.glassfish:javax.annotation:10.0-b28'
```
#### **GsonFormat**
　　GsonFormat是AndroidStudio中的一个插件，在AndroidStudio的插件选项中直接搜索安装这个插件即可。在这里看一下是如何使用这个插件的。
![这里写图片描述](http://img.blog.csdn.net/20160403112125709)
### **添加转换器**
　　在这里我们需要为retrofit添加gson转换器的依赖。添加过converter-gson后不用再添加gson库。在converter-gson中已经包含gson。
```java
compile 'com.squareup.retrofit2:converter-gson:2.0.1'
```
　　在这里先创建一个Java类Contributor，用来保存接收到的数据。
```java
public class Contributor {
    private String login;
    private Integer contributions;

    public String getLogin() {
        return login;
    }

    public void setLogin(String login) {
        this.login = login;
    }

    public Integer getContributions() {
        return contributions;
    }

    public void setContributions(Integer contributions) {
        this.contributions = contributions;
    }
}
```
　　这时候修改我们的API接口。
```java
@GET("repos/{owner}/{repo}/contributors")
Call<List<Contributor>> contributorsByAddConverterGetCall(@Path("owner") String owner, @Path("repo") String repo);
```
　　创建retrofit实例，我们通过addConverterFactory指定一个factory来对响应反序列化，在这里converters被添加的顺序将是它们被Retrofit尝试的顺序。
```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.github.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```
　　调用上面所修改的API接口。
```java
GitHubApi repo = retrofit.create(GitHubApi.class);
Call<List<Contributor>> call = repo.contributorsByAddConverterGetCall(mUserName, mRepo);
call.enqueue(new Callback<List<Contributor>>() {
    @Override
    public void onResponse(Call<List<Contributor>> call, Response<List<Contributor>> response) {
        List<Contributor> contributorList = response.body();
        for (Contributor contributor : contributorList){
            Log.d("login", contributor.getLogin());
            Log.d("contributions", contributor.getContributions() + "");
        }
    }
    
    @Override
    public void onFailure(Call<List<Contributor>> call, Throwable t) {

    }
});
```
　　最后在来看一下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160402190927032)
　　retrofit不仅仅只支持gson，还支持其他许多json解析库。以下版本号需要与retrofit版本号保持一致，并且以retrofit官网给出的版本号为准。

<ul>
<li><a href="https://github.com/google/gson">Gson</a>: <code>compile 'com.squareup.retrofit2:converter-gson:2.0.1'</code></li>
<li><a href="http://wiki.fasterxml.com/JacksonHome">Jackson</a>: <code>compile 'com.squareup.retrofit2:converter-jackson:2.0.1'</code></li>
<li><a href="https://github.com/square/moshi/">Moshi</a>: <code>compile 'com.squareup.retrofit2:converter-moshi:2.0.1'</code></li>
<li><a href="https://developers.google.com/protocol-buffers/">Protobuf</a>: <code>compile 'com.squareup.retrofit2:converter-protobuf:2.0.1'</code></li>
<li><a href="https://github.com/square/wire">Wire</a>: <code>compile 'com.squareup.retrofit2:converter-wire:2.0.1'</code></li>
<li><a href="http://simple.sourceforge.net/">Simple XML</a>: <code>compile 'com.squareup.retrofit2:converter-simplexml:2.0.1'</code></li>
</ul>

## **增加日志信息**
　　在retrofit2.0中是没有日志功能的。但是retrofit2.0中依赖OkHttp，所以也就能够通过OkHttp中的interceptor来实现实际的底层的请求和响应日志。在这里我们需要修改上一个retrofit实例，为其自定自定义的OkHttpClient。代码如下：
```java
HttpLoggingInterceptor httpLoggingInterceptor = new HttpLoggingInterceptor();
httpLoggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .addInterceptor(httpLoggingInterceptor)
        .build();

Retrofit retrofit = new Retrofit.Builder().addCallAdapterFactory(RxJavaCallAdapterFactory.create())
        .client(okHttpClient)
        .baseUrl("https://api.github.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```
　　还需要添加如下依赖。
```java
compile 'com.squareup.okhttp3:logging-interceptor:3.1.2'
```
　　其他代码没有任何变化，我们来看一下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160402204610320)
## **添加请求头**
　　我们可以通过@Headers来添加请求头。
```java
@Headers({
        "Accept: application/vnd.github.v3.full+json",
        "User-Agent: RetrofitBean-Sample-App",
        "name:ljd"
})
@GET("repos/{owner}/{repo}/contributors")
Call<List<Contributor>> contributorsAndAddHeader(@Path("owner") String owner,@Path("repo") String repo);
```
　　运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160402214100544)
## **同步请求**
　　在这里我们可以直接通过call.execute()执行一个同步请求，由于不允许在主线程中进行网络请求操作，所以我们需要再子线程中进行执行。
```java
new Thread(new Runnable() {
    @Override
    public void run() {

        try {
            Response<List<Contributor>> response = call.execute();
            List<Contributor> contributorsList = response.body();
            for (Contributor contributor : contributorsList){
                Log.d("login",contributor.getLogin());
                Log.d("contributions",contributor.getContributions()+"");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}).start();
```
　　在这里看一下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160402215006782)
## **clone**
　　在这里无论是同步操作还是异步操作每一个call对象实例只能被执行一次。多次执行抛出如下异常。
![这里写图片描述](http://img.blog.csdn.net/20160402215347799)
　　在这里如果我们的request和respone都是一一对应的。我们通过Clone方法创建一个一模一样的实例，并且它的开销也是很小的。
```java
Call<List<Contributor>> cloneCall = call.clone();
cloneCall.execute();
```
## **get请求**
　　在前面的一些例子当中我们都是采用get请求。当然我们也可以为URL指定查询参数。使用@Query即可。
```java
@GET("search/repositories")
Call<RetrofitBean> queryRetrofitByGetCall(@Query("q")String owner,
                                      @Query("since")String time,
                                      @Query("page")int page,
                                      @Query("per_page")int per_Page);
```
　　当我们的参数过多的时候我们可以通过@QueryMap注解和map对象参数来指定每个表单项的Key，value的值。
```java
@GET("search/repositories")
Call<RetrofitBean> queryRetrofitByGetCallMap(@QueryMap Map<String,String> map);
```
　　下面的call对象实例为上面api中所返回call对象。更具所返回的json数据所创建的实体类在这里就不在贴出代码，下载源码详细查看。
```java
call.enqueue(new Callback<RetrofitBean>() {
    @Override
    public void onResponse(Call<RetrofitBean> call, Response<RetrofitBean> response) {
        RetrofitBean retrofit = response.body();
        List<Item> list = retrofit.getItems();
        if (list == null)
            return;
        Log.d(TAG, "total:" + retrofit.getTotalCount());
        Log.d(TAG, "incompleteResults:" + retrofit.getIncompleteResults());
        Log.d(TAG, "----------------------");
        for (Item item : list) {
            Log.d(TAG, "name:" + item.getName());
            Log.d(TAG, "full_name:" + item.getFull_name());
            Log.d(TAG, "description:" + item.getDescription());
            Owner owner = item.getOwner();
            Log.d(TAG, "login:" + owner.getLogin());
            Log.d(TAG, "type:" + owner.getType());
        }

    }

    @Override
    public void onFailure(Call<RetrofitBean> call, Throwable t) {

    }
});
```
　　上面请求中的完整连接为: https://api.github.com/search/repositories?q=retrofit&since=2016-03-29&page=1&per_page=3
![这里写图片描述](http://img.blog.csdn.net/20160402223902582)

　　在Retrofit 2.0添加了一个新的注解：@Url，它允许我们直接传入一个请求的URL。这样以来我们可以将上一个请求的获得的url直接传入进来。方便了我们的操作。
```java
@GET
Call<List<Contributor>> repoContributorsPaginate(@Url String url);
```
## **Form encoded和Multipart**
### **Form encoded**
　　我们可以使用@FormUrlEncoded注解来发送表单数据。使用 @Field注解和参数来指定每个表单项的Key，value为参数的值。
```java
@FormUrlEncoded
@POST("user/edit")
Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);
```
　　当我们有很多个表单参数时可以通过@FieldMap注解和Map对象参数来指定每个表单项的Key，value的值。
```java
@FormUrlEncoded
@POST("user/edit")
Call<User> updateUser(@FieldMap Map<String,String> fieldMap);
```
### **Multipart**
　　我们还可以通过@Multipart注解来发送Multipart数据。通过@Part注解来定义需要发送的文件。
```java
@Multipart
@PUT("/user/photo")
User updateUser(@Part("photo") TypedFile photo, @Part("description") TypedString description);
```
## **Retrofit与RxJava结合**
　　Retrofit能够与RxJava进行完美结合。下面就来看一下Retrofit与RxJava是如何结合在一起的。对于RxJava在这就不在进行详细介绍，对于RXJava的使用可以参考附录里面给出链接。
　　首先我们需要添加如下依赖。
```java
compile 'com.squareup.retrofit2:adapter-rxjava:2.0.1'
compile 'io.reactivex:rxandroid:1.1.0'
```
　　创建retrofit对象实例时，通过addCallAdapterFactory来添加对RxJava的支持。
```java
HttpLoggingInterceptor httpLoggingInterceptor = new HttpLoggingInterceptor();
httpLoggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .addInterceptor(httpLoggingInterceptor)
        .build();
Retrofit retrofit = new Retrofit.Builder()
        .client(okHttpClient)
        .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
        .addConverterFactory(GsonConverterFactory.create())
        .baseUrl("https://api.github.com/")
        .build();
```
　　使用Observable创建一个API接口。
```java
@GET("repos/{owner}/{repo}/contributors")
Observable<List<Contributor>> contributorsByRxJava(@Path("owner") String owner,@Path("repo") String repo);
```
　　下面来调用这个API接口。
```java
private CompositeSubscription mSubscriptions = new CompositeSubscription();
```
```java
mSubscriptions.add(
        mGitHubService.contributorsByRxJava(mUserName, mRepo)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<List<Contributor>>() {
                    @Override
                    public void onCompleted() {
                    }

                    @Override
                    public void onError(Throwable e) {
                    }

                    @Override
                    public void onNext(List<Contributor> contributors) {
                        for (Contributor c : contributors) {
                            Log.d("TAG", "login:" + c.getLogin() + "  contributions:" + c.getContributions());
                        }
                    }
                }));
```
　　下面来看一下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160403110118763)
　　如果我们想要查看所有contributor的信息，首先我们需要向gitHub请求获取到所有contributor，然后再通过获得contributor进行依次向github请求获取contributor的信息，在这时候我们使用RxJava也就非常方便了。下面看一下如何操作的。
　　首先再添加一个API接口。
```java
@GET("repos/{owner}/{repo}/contributors")
Observable<List<Contributor>> contributorsByRxJava(@Path("owner") String owner,
                                                   @Path("repo") String repo);
```
　　下面在看一下是如何进行根据获得的contributor来查看contributor的信息。
```java
mSubscriptions.add(mGitHubService.contributorsByRxJava(mUserName, mRepo)
        .flatMap(new Func1<List<Contributor>, Observable<Contributor>>() {
            @Override
            public Observable<Contributor> call(List<Contributor> contributors) {
                return Observable.from(contributors);
            }
        })
        .flatMap(new Func1<Contributor, Observable<Pair<User, Contributor>>>() {
            @Override
            public Observable<Pair<User, Contributor>> call(Contributor contributor) {
                Observable<User> userObservable = mGitHubService.userByRxJava(contributor.getLogin())
                        .filter(new Func1<User, Boolean>() {
                            @Override
                            public Boolean call(User user) {
                                return !isEmpty(user.getName()) && !isEmpty(user.getEmail());
                            }
                        });

                return Observable.zip(userObservable,
                        Observable.just(contributor),
                        new Func2<User, Contributor, Pair<User, Contributor>>() {
                            @Override
                            public Pair<User, Contributor> call(User user, Contributor contributor) {
                                return new Pair<>(user, contributor);
                            }
                        });
            }
        })
        .subscribeOn(Schedulers.newThread())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Observer<Pair<User, Contributor>>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Pair<User, Contributor> pair) {
                User user = pair.first;
                Contributor contributor = pair.second;
                Log.d(TAG, "name:" + user.getName());
                Log.d(TAG, "contributions:" + contributor.getContributions());
                Log.d(TAG, "email:" + user.getEmail());

            }
        }));
```
　　最后在来看下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160403111155611)
## **Retrofit设置缓存**
对Retrofit设置缓存，由于Retrofit是对OkHttp的封装，所以我们可以直接通过OkHttpClient着手。也就是为OkHttp设置缓存。设置缓存代码如下所示。
```java
private OkHttpClient getCacheOkHttpClient(Context context){
    final File baseDir = context.getCacheDir();
    final File cacheDir = new File(baseDir, "HttpResponseCache");
    Timber.e(cacheDir.getAbsolutePath());
    Cache cache = new Cache(cacheDir, 10 * 1024 * 1024);   //缓存可用大小为10M

    Interceptor REWRITE_CACHE_CONTROL_INTERCEPTOR = chain -> {
        Request request = chain.request();
        if(!NetWorkUtils.isNetWorkAvailable(context)){
            request = request.newBuilder()
                    .cacheControl(CacheControl.FORCE_CACHE)
                    .build();
        }

        Response originalResponse = chain.proceed(request);
        if (NetWorkUtils.isNetWorkAvailable(context)) {
            int maxAge = 60;                  //在线缓存一分钟
            return originalResponse.newBuilder()
                    .removeHeader("Pragma")
                    .removeHeader("Cache-Control")
                    .header("Cache-Control", "public, max-age=" + maxAge)
                    .build();

        } else {
            int maxStale = 60 * 60 * 24 * 4 * 7;     //离线缓存4周
            return originalResponse.newBuilder()
                    .removeHeader("Pragma")
                    .removeHeader("Cache-Control")
                    .header("Cache-Control", "public, only-if-cached, max-stale=" + maxStale)
                    .build();
        }
    };
    
    return new OkHttpClient.Builder()
            .addNetworkInterceptor(REWRITE_CACHE_CONTROL_INTERCEPTOR)
            .addInterceptor(REWRITE_CACHE_CONTROL_INTERCEPTOR)
            .cache(cache)
            .build();
}
```
# **总结**
　　在retrofit的使用中，对于文件你上传与下载，并没有为我们提供进度更新的接口，在这里就需要我们自己处理了。在下面的例子中给出一个文件下载的例子，并且对下载进度更新通过logcat打印出来。可以下载进行查看。到这里retrofit的基本用法也就介绍完了，对于retrofit更多的好处在使用中我们可以慢慢体会。
# **[源码下载](https://github.com/lijiangdong/retrofit-example)**
# **附录**
 - [Retrofit官方文档](http://square.github.io/retrofit/)
 - [用 Retrofit 2 简化 HTTP 请求](https://realm.io/cn/news/droidcon-jake-wharton-simple-http-retrofit-2/)
 - [给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)
 - [RxJava Essentials 中文翻译版](http://rxjava.yuxingxin.com/)