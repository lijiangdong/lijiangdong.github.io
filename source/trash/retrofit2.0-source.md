---
title:  Retrofit2源码解读
date: 2016-06-18 15:20
category: [Android]
tags: [retrofit]
comments: true
---

　　Retrofit2的用法在[Retrofit2.0使用详解](http://blog.csdn.net/ljd2038/article/details/51046512)这篇文章中已经详细介绍过了。那么在这就来看一下Retrofit2它是如何实现的。Retrofit2中它的内部网络请求是依赖于OKHttp，所以Retrofit2可以看做是对OKHttp的一次封装，那么下面就开看下Retrofit2是如何对OKHttp进行封装的。<!--more-->
# **回顾Retrofit2的使用**
　　在这里首先来回顾一下Retrofit2的使用。对于Retrofit2的使用可以分为三步。
　　首先，我们创建一个Java接口GitHubService来作为Http请求接口。
```java
public interface GitHubService {

    @GET("repos/{owner}/{repo}/contributors")
    Call<ResponseBody> contributorsBySimpleGetCall(@Path("owner") String owner,
@Path("repo") String repo);
}
```
　　然后我们需要创建一个Retrofit实例，并通过Retrofit对象创建一个GitHubService接口的实现。
```java
Retrofit retrofit = new Retrofit.Builder()
        //添加对RxJava的支持
        .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
        //添加对Json转换器的支持
        .addConverterFactory(GsonConverterFactory.create())
        .baseUrl("https://api.github.com/")
        .build();
GitHubService service = retrofit.create(GitHubService.class);
```
　　最后通过调用我们创建的接口便能够向后台发起请求。
```
Call<ResponseBody> call = service.contributorsBySimpleGetCall("square", "retrofit");
call.enqueue(new Callback<ResponseBody>() {
    @Override
    public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {
        ......
    }

    @Override
    public void onFailure(Call<ResponseBody> call, Throwable t) {
        ......
    }
});
```
# **Retrofit2源码分析**
　　首先通过@GET来标识这个接口是一个GET请求。那么看一下这个GET注解的定义。
```java
package retrofit2.http;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;
import okhttp3.HttpUrl;

import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

@Documented
@Target(METHOD)
@Retention(RUNTIME)
public @interface GET {
  String value() default "";
}
```
　　从注解中可以看出这个注解是对方法的声明，并且在运行时VM依然保留注解。Java自定义注解在这里就不在过多说明，可以参考[ Java注解在Android中使用](http://blog.csdn.net/ljd2038/article/details/51586257)这篇文章。
　　下面就再来看一下是如何创建Retrofit对象的。对于Retrofit对象的创建采用的是Builder模式。那么在这里我们就来看一下这个Builder类。
```java
public static final class Builder {

  //对平台的支持
  private Platform platform;
  //发起请求OKHttp3的Client工厂
  private okhttp3.Call.Factory callFactory;
  //OKHttp2的HttpUrl对象，也就是将我们传入的baseUrl字符串包装成HttpUrl
  private HttpUrl baseUrl;
  //转换器工厂集合，retrofit可以插入多个转化器，例如：Gson，Jackson等等
  private List<Converter.Factory> converterFactories = new ArrayList<>();
  //用于发起请求和接收响应的Call适配器工厂集合，
  //Retrofit对RxJava的支持就是通过在该集合中添加RxJavaCallAdapterFactory,
  //而RxJavaCallAdapterFactory正是继承自CallAdapter.Factory
  private List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
  //Executor并发框架，用于对请求后响应结果的回调执行
  private Executor callbackExecutor;
  //是否需要立即生效
  private boolean validateEagerly;

  Builder(Platform platform) {
    this.platform = platform;
    // Add the built-in converter factory first. This prevents overriding its behavior but also
    // ensures correct behavior when using converters that consume all types.
    converterFactories.add(new BuiltInConverters());
  }

  public Builder() {
    this(Platform.get());
  }

  //对属性的配置
  ......

  public Retrofit build() {
    if (baseUrl == null) {
      throw new IllegalStateException("Base URL required.");
    }

    okhttp3.Call.Factory callFactory = this.callFactory;
    if (callFactory == null) {
      callFactory = new OkHttpClient();
    }

    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
      callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
    adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

    // Make a defensive copy of the converters.
    List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

    return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
        callbackExecutor, validateEagerly);
  }
}
```
　　对于Retrofit中的属性配置已经在注释中进行说明。在这里在对Platform进行一下介绍。我们可以看出通过Platform.get()来获取到当前运行的平台。下面就进出Platform.get()方法中来查看一下Retrofit到底支持哪些平台。
```java
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("org.robovm.apple.foundation.NSObject");
      return new IOS();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }
```
　　在Platform的get方法中实际上还是调用了findPlatform方法，通过findPlatform方法我们可以看出Retrofit支持三个平台，它们分别是Android，Java8，IOS。这里的IOS指的是RoboVM。RoboVM它是一种可以在iOS设备上运行Java应用程序的技术，这种技术主要还是用于在游戏开发中。在这里我们只讨论在Android平台中的使用。在获取到当前平台为Android平台之后返回一个Android对象，这个Android类是Platform中的一个静态内部类，并且它继承自Platform。下面就在来看一下这个Android类。
```java
  static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
```
　　在这个Android类中重写了父类的defaultCallbackExecutor和defaultCallAdapterFactory方法。
对于defaultCallbackExecutor方法它所返回的一个Executor，从它的实现类MainThreadExecutor可以看出，实际上就是在主线中创建一个Handler对象，然后通过Handler的post方法将请求的结果回调到主线程中运行。而defaultCallAdapterFactory它是默认的Call适配器，它是一个ExecutorCallAdapterFactory对象，对于ExecutorCallAdapterFactory在后面会进行说明。
在这里完成了对于Retrofit对象的创建以后，便通过Retrofit中的create方法创建一个我们请求接口的实现。下面就来看一下这个create方法。
```java
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }

  private void eagerlyValidateMethods(Class<?> service) {
    Platform platform = Platform.get();
    for (Method method : service.getDeclaredMethods()) {
      if (!platform.isDefaultMethod(method)) {
        loadServiceMethod(method);
      }
    }
  }

  ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
　　在create方法中首先会进行判断所传入进来的Service是否是一个接口，当我们将validateEagerly属性设为true的时候，在我们调用create方法创建一个Service，就直接调用eagerlyValidateMethods方法。而eagerlyValidateMethods的作用是通过反射获取我们创建service接口中所有的接口方法，然后根据接口方法和当前的retrofit对象来获得ServiceMethod并且以接口方法作为Key，ServiceMethod作为值添加到serviceMethodCache缓存中。下次便可以通过接口方法直接获取ServiceMethod。
　　现在在往下看我们就明白为什么通过接口来作为一个Http请求，以及为什么调用create方法时需要验证我们的service是否为一个接口。因为在这里是通过Java的动态代理来实现Http请求，并返回一个代理类对象。对于Java的动态代理它是需要委托类与代理类实现同一个接口，在这里也就是我们创建的service请求接口。对于Java的动态代理可参考[Java设计模式之代理模式](http://blog.csdn.net/ljd2038/article/details/51440489)这篇文章。当我们通过代理类（也就是我们调用create方法后返回的service）调用我们所创建的接口方法时。InvocationHandler中的invoke方法将会被调用。在invoke方法中由于method.getDeclaringClass()获取到的是一个接口，并不是Object类，所以第一个条件不成立。而在Android平台下platform.isDefaultMethod(method)返回的为false，所以这个条件也不成立。之后通过loadServiceMethod方法来获取ServiceMethod。最后调用ServiceMethod中callAdapter的adapt方法。而ServiceMethod中的callAdapter属性是通过ServiceMethod中createCallAdapter方法所创建。在createCallAdapter中实际上是调用了Retrofit中的callAdapter方法来对ServiceMethod中的callAdapter进行初始化。下面再看一下Retrofit中的callAdapter方法。
```java
  public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }
  
  public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");

    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    
    ......
  }
```

　　在这段代码中，通过遍历adapterFactories并根据我们的接口方法所中返回值类型来获取响应的适配器callAdapter。例如上面的例子中返回值是一个Call对象，将会采用默认的适配器，如果我们返回的是RxJava中Observable对象，如果我们添加了RxJavaCallAdapterFactory，那么返回的就是RxJavaCallAdapter。如果没有添加那么此处的adapter为null，便会抛出异常。在正是通过这种适配器模式完成了对RxJava的完美结合。下面就以Retrofit默认的callAdapter为例来看一下是如何调用OKHttp来完成对网络的请求的。对于此时的callAdapter就是通过platform.defaultCallAdapterFactory(callbackExecutor)所创建的适配器。从刚才观察Platform内部类Android中可以看出它返回的是一个ExecutorCallAdapterFactory对象。下面就来看一下这个ExecutorCallAdapterFactory类。
```java
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter<Call<?>> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public <R> Call<R> adapt(Call<R> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }

  static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(final Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(call, new IOException("Canceled"));
              } else {
                callback.onResponse(call, response);
              }
            }
          });
        }

        @Override public void onFailure(final Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(call, t);
            }
          });
        }
      });
    }
    ......
}
```
　　从ExecutorCallAdapterFactory类中可看出通过get方法返回一个CallAdapter对象，从对CallAdapter的实现中可以看出在CallAdapter中的adapt方法返回的是ExecutorCallbackCall对象。它实现了Retrofit中的Call接口。到这里我们也就明白了，当通过代理调用我们创建的接口方法中所返回的Call对象就是这个ExecutorCallbackCall。当我们通过call.enqueue来完成网络请求操作实际上就是调用ExecutorCallbackCall中的enqueue方法。在ExecutorCallbackCall中enqueue又将网络请求委托给OkHttpCall去执行。而这个OkHttpCall正是我们在Retrofit的create方法中所创建的OkHttpCall。由于OKHttp的CallBack接口中的onResponse和onFailure是在子线程中执行的，所以在这时候又通过callbackExecutor将CallBack的onResponse和onFailure切换到主线程中执行。

# **总结**
　　在这里对于Retrofit2的源码分析就结束了,在这里我们只分析了通过GET方式进行异步请求这一种情况，对于同步，以及POST请求等原理类似，在这就不在进行详细说明。在Retrofit2中我们可以发现它的代码虽然不多，但是却大量用到了Java中的设计模式。很是值得我们学习。