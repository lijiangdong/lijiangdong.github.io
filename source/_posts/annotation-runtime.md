---
title:  Java注解在Android中使用
date: 2016-06-04 21:40
category: [Java]
tags: [Annotation]
comments: true
visible: hide
---

　　注解（Annotation）也被称为元数据(Metadata)，是在Java SE 5中提供的一个新特性。Annotation可以用来修饰类，属性，方法。在Java中，除了使用系统内置的Annotation以外，用户还能够通过自定义Annotation来实现所需要的功能。下面就来看一下如何使用自定义的Annotation。<!--more-->
# **元注解**
　　在Java中目前包含及四种元注解。元注解是专门用来负责注解其他的注解。所以自定义注解也是通过元注解来完成的。首先在这里来看一下Java中的四种元注解。

 1. @Target
 @Target表示Annotation可用在什么地方。其中的ElementType取值如下：
    TYPE：类，接口或者是enum声明
    FIELD：域声明（包括enum实例）
    METHOD：方法声明
    PARAMETER：参数声明
    CONSTRUCTOR：构造器声明
    LOCAL_VARIABLE：局部变量声明
    ANNOTATION_TYPE：注解类型声明
    PACKAGE：包声明
 2. @Retention
 @Retention表示在什么级别保存该注解信息。其中RetentionPolicy取值如下：
    SOURCE：只在源码中保留，该注解将会被编译器丢掉
    CLASS：注解在class文件中可用，但是会被VM丢弃
    RUNTIME：VM会在运行时保留注解，这时可以通过反射读取注解信息。
 3. @Documented
 @Documented表示在Javadocs中包含这个注解。
 4. @Inherited
 @Inherited表示允许子类继承父类中的注解。

# **自定义注解实现View注入**
　　下面我们就通过一个例子来看一下如何自定义注解以及对自定义注解的使用。在Android开发中通常会在Activity中通过setContentView来为Activity指定Layout，也会经常通过findViewById来回去View控件，并且更麻烦的就是通过setOnClickListener来为其设置点击事件。我们就通过注解的方式来实现这三个功能从而简化我们的代码，增加代码的可读性。下面就来看一下具体的代码实现。
　　定义一个ContentView注解来为Activity指定其Layout。
```java
package com.ljd.annotation.inject;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ContentView {
    int value();
}
```
　　定义一个InjectView注解实现Activity中View的注入。
```java
package com.ljd.annotation.inject;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Inject {
    int value();
}
```
　　定义一个OnClick注解实现View的事件注入。
```java
package com.ljd.annotation.inject;

import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OnClick {
    int[] value();
}
```
　　在这里我们采用int类型和int数组作为注解中的值。需要注意是并不是所有类型都适用于注解元素。注解元素的可用类型如下所示：

 - 所有基本类型（int,float,boolean等）
 - String
 - Class
 - enum
 - Annotation
 - 以上类型的数组

　　在上面这三个自定义注解中它们分别修饰了类、属性和方法。并且在运行时保留这些注解，所以我们便可以通过反射对这些注解进行解析。下面就开看一下是如何通过反射来解析这些注解。
```java
package com.ljd.annotation.inject;

import android.app.Activity;
import android.view.View;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class ViewInject {

    private static Class<?> clazz;

    public static void inject(Activity activity){

        //获取activity的Class类
        clazz = activity.getClass();
        injectContent(activity);
        injectView(activity);
        injectEvent(activity);
    }

    public static void unInject(){
        clazz = null;
    }

    /**
     * 对ContentView注解进行解析
     * @param activity
     */
    private static void injectContent(Activity activity){

        //取的Activity中的ContentView注解
        ContentView contentView = clazz.getAnnotation(ContentView.class);
        if (contentView != null){

            //取出ContentView注解中的值
            int id = contentView.value();
            try {

                //获取Activity中setContentView方法,执行setContentView方法为Activity设置ContentView
                //在这一步中我们也可以直接使用 activity.setContentView(id) 来设置ContentView
                clazz.getMethod("setContentView",Integer.TYPE).invoke(activity,id);
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 对InjectView注解进行解析
     * @param activity
     */
    private static void injectView(Activity activity){

        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields){
            Inject inject = field.getAnnotation(Inject.class);
            if (inject != null){
                int id = inject.value();
                try {
                    //这一步中同样也能够使用 Object view = activity.findViewById(id) 来获取View
                    Object view = clazz.getMethod("findViewById",Integer.TYPE).invoke(activity,id);
                    field.set(activity,view);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }

        }
    }

    /**
     * 对OnClick注解进行解析
     * @param activity
     */
    private static void injectEvent(Activity activity){

        Method[] methods = clazz.getMethods();

        for (Method method : methods) {
            OnClick onClick = method.getAnnotation(OnClick.class);
            if (onClick != null){
                int[] ids = onClick.value();
                MyInvocationHandler handler = new MyInvocationHandler(activity,method);
                
                //通过Java中的动态代理来执行View.OnClickListener
                Object listenerProxy = Proxy.newProxyInstance(
                        View.OnClickListener.class.getClassLoader(),
                        new Class<?>[] { View.OnClickListener.class }, handler);
                for (int id : ids) {

                    try {
                        Object view = clazz.getMethod("findViewById",Integer.TYPE).invoke(activity,id);
                        Method listenerMethod = view.getClass()
                                .getMethod("setOnClickListener", View.OnClickListener.class);
                        listenerMethod.invoke(view, listenerProxy);
                    } catch (NoSuchMethodException e) {
                        e.printStackTrace();
                    } catch (InvocationTargetException e) {
                        e.printStackTrace();
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }

                }
            }
        }
    }

    static class MyInvocationHandler implements InvocationHandler {

        private Object target = null;
        private Method method = null;

        public MyInvocationHandler(Object target,Method method) {
            super();
            this.target = target;
            this.method = method;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

            return this.method.invoke(target,args);
        }
    }
}
```
　　在这里执行View的点击事件时是通过Java中的动态代理来执行的。对于Java动态代理的使用参见[Java设计模式之代理模式](http://blog.csdn.net/ljd2038/article/details/51440489)这篇文章。下面就来看一下如何使用我们自定义的这些注解。
```java
package com.ljd.annotation;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

import com.ljd.annotation.inject.ContentView;
import com.ljd.annotation.inject.Inject;
import com.ljd.annotation.inject.OnClick;
import com.ljd.annotation.inject.ViewInject;


@ContentView(R.layout.activity_main)
public class MainActivity extends AppCompatActivity {

    @Inject(R.id.test_text)
    TextView textView;

    @Inject(R.id.test_btn)
    Button button;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ViewInject.inject(this);
        textView.setText("hello word");
        button.setText("test");
    }

    @OnClick({R.id.test_btn,R.id.test_text})
    public void onClick(View view) {
        switch (view.getId()){
            case R.id.test_btn:
                Toast.makeText(this,"test onClick",Toast.LENGTH_SHORT).show();
                break;
            case R.id.test_text:
                Toast.makeText(this,"hello word",Toast.LENGTH_SHORT).show();
                break;
            
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        ViewInject.unInject();
    }
}
```
　　下面是布局文件
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="@dimen/activity_vertical_margin"
    tools:context="com.ljd.annotation.MainActivity">

    <TextView
        android:id="@+id/test_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <Button
        android:id="@+id/test_btn"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />
</LinearLayout>
```
# **总结**
　　对于这个例子目的主要是为了对自定义注解以及Java反射的一个学习，对于实际应用并没有多大的意义，并且在我们的应用中应当避免对反射的使用，因为使用反射会大大降低我们应用的执行效率。对于View的注入我们可以采用[JakeWharton](https://github.com/JakeWharton)的[butterknife](https://github.com/JakeWharton/butterknife)框架。
# [**源码下载**](https://github.com/lijiangdong/annotation-example)