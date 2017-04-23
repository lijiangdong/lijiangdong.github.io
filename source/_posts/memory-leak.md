---
title: Android性能优化之内存泄漏
date: 2016-12-11 16:34
category: [Android]
tags: [性能优化]
comments: true
---

　　内存泄漏（memory leak）是指由于疏忽或错误造成程序未能释放已经不再使用的内存。那么在Android中，当一个对象持有Activity的引用，如果该对象不能被系统回收，那么当这个Activity不再使用时，这个Activity也不会被系统回收，那这么以来便出现了内存泄漏的情况。在应用中内出现一次两次的内存泄漏获取不会出现什么影响，但是在应用长时间使用以后，若是存在大量的Activity无法被GC回收的话，最终会导致OOM的出现。那么我们在这就来分析一下导致内存泄漏的常见因素并且如何去检测内存泄漏。<!--more-->
# **导致内存泄漏的常见因素**
## **情景一：静态Activity和View**
　　静态变量Activity和View会导致内存泄漏，在下面这段代码中对Activity的Context和TextView设置为静态对象，从而产生内存泄漏。
```java
import android.content.Context;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    private static Context context;
    private static TextView textView;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        context = this;
        textView = new TextView(this);
    }
}
```
## **情景二：Thread，匿名类，内部类**
　　在下面这段代码中存在一个非静态的匿名类对象Thread，会隐式持有一个外部类的引用LeakActivity，从而导致内存泄漏。同理，若是这个Thread作为LeakActivity的内部类而不是匿名内部类，他同样会持有外部类的引用而导致内存泄漏。在这里只需要将为Thread匿名类定义成静态的内部类即可（静态的内部类不会持有外部类的一个隐式引用）。
```java
public class LeakActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        leakFun();
    }
    
    private void leakFun(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(Integer.MAX_VALUE);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```
## **情景三：动画**
　　在属性动画中有一类无限循环动画，如果在Activity中播放这类动画并且在onDestroy中去停止动画，那么这个动画将会一直播放下去，这时候Activity会被View所持有，从而导致Activity无法被释放。解决此类问题则是需要早Activity中onDestroy去去调用objectAnimator.cancel()来停止动画。
```java
public class LeakActivity extends AppCompatActivity {

    private TextView textView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        textView = (TextView)findViewById(R.id.text_view);
        ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(textView,"rotation",0,360);
        objectAnimator.setRepeatCount(ValueAnimator.INFINITE);
        objectAnimator.start();
    }
}
```
## **情景四：Handler**
　　对于Handler的内存泄漏在(Android的消息机制——Handler的工作过程)[http://blog.csdn.net/ljd2038/article/details/50889754]这篇文章中已经详细介绍，就不在赘述。

## **情景五：第三方库使用不当**
　　对于EventBus，RxJava等一些第三开源框架的使用，若是在Activity销毁之前没有进行解除订阅将会导致内存泄漏。
# **使用MAT检测内存泄漏**
　　对于常见的内存泄露进行介绍完以后，在这里再看一下使用MAT（Memory Analysis Tool）来检测内存泄露。MAT的下载地址为：http://www.eclipse.org/mat/downloads.php。
　　下面来看一段会导致内存泄露的错误代码。
```java
public class LeakActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        EventBus.getDefault().register(this);
    }
    
    @Subscribe
    public void subscriber(String s){

    }
}
```
　　在上面这段代码中有会导致内存泄漏，原因是EventBus没有解除注册。下面就以这段代码为例来看一下如何分析内存泄漏。
　　打开AndroidStudio中的Monitors可以看到如下界面。
![这里写图片描述](http://img.blog.csdn.net/20161211140959892?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　
　　在这里可以看到在应用刚启动的时候，所占用的内存为15M，然后我们现在开始操作APP，反复进入退出LeakActicity。点击上如中的GC按钮。这时候我们在看一下内存使用情况。
![这里写图片描述](http://img.blog.csdn.net/20161211142237659?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　在这里我们可以看到，内存一直在持续增加，已经达到33M，并且无法被GC所回收。所以我们可以判断，这时候必然出现内存泄漏的情形。那么现在再点击Dump Java Heap按钮，在captures窗口看到生成得hprof文件。但这时候所生成的hprof文件不是标准格式的，我们需要通过SDK所提供的工具hprof-conv进行转化，该工具在SDK的platform-tools目录下。执行命令如下：
```java
　　hprof-conv XXX.hprof converted-dump.hprof
```
　　当然在AndroidStudio中可以省去这一步，可以直接导出标准格式的hprof文件。
  ![这里写图片描述](http://img.blog.csdn.net/20161211142936042?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　
　　这时候可以通过MAT工具来打开导出的hprof文件。打开界面如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20161211143501279?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　在MAT中我们最常用的就是Histogram和Dominator Tree，他们分别对应上图中的A和B按钮。Histogram可以看出内存中不同类型的buffer的数量和占用内存的大小，而Dominator Tree则是把内存中的对象按照从大到小的顺序进行排序，并且可以分析对象之间的引用关系。在这里再来介绍一下MAT中两个符号的含义。

- ShallowHeap:对象自身占用的内存大小，不包括他引用的对象
- RetainedHeap:对象自身占用的内存大小并且加上它直接或者间接引用对象的大小
## **Histogram**
　　 由于在Android中一般内存泄漏大多出现在Acivity中，这时候可以点击Histogram按钮，并搜索
Activity。
![这里写图片描述](http://img.blog.csdn.net/20161211150708824?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

　　 在这里可以看出LeakActivity存在69个对象，基本上可以断定存在内存泄漏的情形，这时候便可以通过查看GC对象的引用链来进行分析。点击鼠标右键选择Merge Shortest paths to GC Roots并选择exclude weak/soft references来排除弱引用和软引用。
![这里写图片描述](http://img.blog.csdn.net/20161211151628184?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　 
　　 在排除软引用和弱引用以后如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20161211151955220?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　 在这里可以看出由于EventBus导致的LeakActivity内存泄漏。
　　 在Histogram中还可以查看一个对象包含了那些对象的引用。例如，现在要查看LeakActivity所包含的引用，可以点击鼠标右键，选择list objects中的with incoming reference。而with outcoming reference表示选中对象持有那些对象的引用。
![这里写图片描述](http://img.blog.csdn.net/20161211152654684?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
## **Dominator Tree**
　　 现在我们点击这时候可以点击Dominator Tree按钮，并搜索
Activity。可以看到如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20161211153200851?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　 
　　 在这里可以看到存在大量的LeakActivity。然后点击鼠标右键选择Path To GC Roots->exclude weak/soft references来排除弱引用和软引用。
![这里写图片描述](http://img.blog.csdn.net/20161211153755980?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　 
　　 之后可以看到如下结果，依然是EventBus导致的内存泄漏：
![这里写图片描述](http://img.blog.csdn.net/20161211153927588?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbGpkMjAzOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# **总结**
　　内存泄漏往往被我们所忽略，但是当大量的内存泄漏以后导致OOM。它所造成的影响也是不容小觑的。当然除了上述内存泄漏的分析以为我们还可以通过[LeakCanary](https://github.com/square/leakcanary)来分析内存泄漏。对于LeakCanary的使用在这里就不在进行详细介绍。