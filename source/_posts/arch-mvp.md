---
title:  Android官方MVP架构解读
date: 2016-05-22 22:24
category: [Android]
tags: [架构]
comments: true
---

　　对于MVP (Model View Presenter)架构是从著名的MVC(Model View Controller)架构演变而来的。而对于Android应用的开发中本身可视为一种MVC架构。通常在开发中将XML文件视为MVC中的View角色，而将Activity则视为MVC中的Controller角色。不过更多情况下在实际应用开发中Activity不能够完全充当Controller，而是Controller和View的合体。于是Activity既要负责视图的显示，又要负责对业务逻辑的处理。这样在Activity中代码达到上千行，甚至几千行都不足为其，同时这样的Activity也显得臃肿不堪。所以对于MVC架构并不很合适运用于Android的开发中。下面就来介绍一下MVP架构以及看一下google官方给出的MVP架构示例。<!--more-->
# **MVP架构简介**
　　对于一个应用而言我们需要对它抽象出各个层面，而在MVP架构中它将UI界面和数据进行隔离，所以我们的应用也就分为三个层次。

-  View: 对于View层也是视图层，在View层中只负责对数据的展示，提供友好的界面与用户进行交互。在Android开发中通常将Activity或者Fragment作为View层。
-  Model: 对于Model层也是数据层。它区别于MVC架构中的Model，在这里不仅仅只是数据模型。在MVP架构中Model它负责对数据的存取操作，例如对数据库的读写，网络的数据的请求等。
-  Presenter:对于Presenter层他是连接View层与Model层的桥梁并对业务逻辑进行处理。在MVP架构中Model与View无法直接进行交互。所以在Presenter层它会从Model层获得所需要的数据，进行一些适当的处理后交由View层进行显示。这样通过Presenter将View与Model进行隔离，使得View和Model之间不存在耦合，同时也将业务逻辑从View中抽离。

　　下面通过MVP结构图来看一下MVP中各个层次之间的关系。
![这里写图片描述](http://img.blog.csdn.net/20160522153041711)
　　在MVP架构中将这三层分别抽象到各自的接口当中。通过接口将层次之间进行隔离，而Presenter对View和Model的相互依赖也是依赖于各自的接口。这点符合了接口隔离原则，也正是面向接口编程。在Presenter层中包含了一个View接口，并且依赖于Model接口，从而将Model层与View层联系在一起。而对于View层会持有一个Presenter成员变量并且只保留对Presenter接口的调用，具体业务逻辑全部交由Presenter接口实现类中处理。
# **官方MVP架构分析**
## **项目介绍**
　　对于MVP架构有了一些的了解，而在前端时间Google给出了一些App开发架构的实现。
　　项目地址为：https://github.com/googlesamples/android-architecture.
　　在这里进入README看一下这次Google给出那些Android开发架构的实现。

![这里写图片描述](http://img.blog.csdn.net/20160522160230609)
　　对于上面五个开发架构的实现表示到目前为止是已经完成的项目，而下面两个则表示正在进行的中的项目。现在首先来介绍一下这几个架构。

- todo-mvp: 基础的MVP架构。
- todo-mvp-loaders:基于MVP架构的实现，在获取数据的部分采用了loaders架构。
- todo-mvp-databinding: 基于MVP架构的实现，采用了数据绑定组件。
- todo-mvp-clean: 基于MVP架构的clean架构的实现。
- todo-mvp-dagger2: 基于MVP架构，采用了依赖注入dagger2。
- dev-todo-mvp-contentproviders: 基于mvp-loaders架构，使用了ContenPproviders。
- dev-todo-mvp-rxjava: 基于MVP架构，对于程序的并发处理和数据层（MVP中的Model）的抽象。

　　从上述的介绍中可以看出，对于官方给出所有的架构的实现最终都是基于MVP架构。所以在这里就对上面的基础的MVP架构todo-mvp进行分析。
## **项目结构的分析**
　　对于这个项目，它实现的是一个备忘录的功能。对于工作中未完成的任务添加到待办任务列表中。我们能够在列表中可以对已完成的任务做出标记，能够进入任务详细页面修改任务内容，也能够对已完成的任务和未完成的任务数量做出统计。
　　首先在这里来看一下todo-mvp整体的项目结构
<img src="http://img.blog.csdn.net/20160522165007677" width=70%>

　　从上图中可以看出，项目整体包含了一个app src目录，四个测试目录。在src目录下面对代码的组织方式是按照功能进行划分。在这个项目中包含了四个功能，它们分别是：任务的添加编辑(addedittask),任务完成情况的统计(statistics),任务的详情(taskdetail),任务列表的显示(tasks)。对于data包它是项目中的数据源，执行数据库的读写，网络的请求操作都存放在该包内，也是MVP架构中的Model层。而util包下面则是存放一些项目中使用到的工具类。在最外层存放了两接口BasePresenter和BaseView。它们是Presenter层接口和View层接口的基类，项目中所有的Presenter接口和View层接口都继承自这两个接口。
　　现在进入功能模块内看下在模块内部对类是如何划分的。在每个功能模块下面将类分作xxActivity,xxFragment,xxPresenter,xxContract。也正是这些类构成了项目中的Presenter层与View层。下面就来分析在这个项目中是如何实现MVP架构。
## **MVP架构的实现**
　　在这里只从宏观上关注MVP架构的实现,对于代码的内部细节在就不在具体分析。那么就以任务的添加和编辑这个功能来看一下Android官方是如何实现MVP架构。
### **Model层的实现**
　　首先我们从MVP架构的最内层开始分析，也就是对应的Model层。在这个项目中对应的data包下的内容。在data下对数据库等一些数据源的封装。对于Presenter层提供了TasksDataSource接口。在这里看一下这个TasksDataSource接口。
```java
public interface TasksDataSource {

    interface LoadTasksCallback {

        void onTasksLoaded(List<Task> tasks);

        void onDataNotAvailable();
    }

    interface GetTaskCallback {

        void onTaskLoaded(Task task);

        void onDataNotAvailable();
    }

    void getTasks(@NonNull LoadTasksCallback callback);

    void getTask(@NonNull String taskId, @NonNull GetTaskCallback callback);

    void saveTask(@NonNull Task task);

    ......
}
```
　　TasksDataSource接口的实现是TasksLocalDataSource，在TasksDataSource中的方法也就是一些对数据库的增删改查的操作。而在TasksDataSource的两个内部接口LoadTasksCallback和GetTaskCallback是Model层的回调接口。它们的真正实现是在Presenter层。对于成功获取到数据后变或通过这个回调接口将数据传递Presenter层。同样，若是获取失败同样也会通过回调接口来通知Presenter层。下面来看一下TasksDataSource的实现类。
```java
public class TasksLocalDataSource implements TasksDataSource {

    ......

    @Override
    public void getTask(@NonNull String taskId, @NonNull GetTaskCallback callback) {

        //根据taskId查训出相对应的task
        ......

        if (task != null) {
            callback.onTaskLoaded(task);
        } else {
            callback.onDataNotAvailable();
        }
    }

    @Override
    public void saveTask(@NonNull Task task) {
        checkNotNull(task);
        SQLiteDatabase db = mDbHelper.getWritableDatabase();

        ContentValues values = new ContentValues();
        values.put(TaskEntry.COLUMN_NAME_ENTRY_ID, task.getId());
        values.put(TaskEntry.COLUMN_NAME_TITLE, task.getTitle());
        values.put(TaskEntry.COLUMN_NAME_DESCRIPTION, task.getDescription());
        values.put(TaskEntry.COLUMN_NAME_COMPLETED, task.isCompleted());

        db.insert(TaskEntry.TABLE_NAME, null, values);

        db.close();
    }

    ......
}
```
　　在这里我们针对任务的添加和编辑功能，所以省略很多代码。可以看出在TasksLocalDataSource中实现的getTask方法，在这个方法中传入TasksDataSource内的GetTaskCallback回调接口。在getTask方法的实现可以看出在查询到Task以后调用回调方法，若是在Presenter层中实现了这两个回调方法，便将数据传递到Presenter层。而对于查询到的Task为空的时候也是通过回调方法执行对应的操作。
　　同样对于通过网络请求获取到数据也是一样，对于成功请求到的数据可以通过回调方法将数据传递到Presenter层，对于网络请求失败也能够通过回调方法来执行相对应的操作。
### **Presenter与View层提供的接口**
　　由于在Presenter和View层所提供的接口在一个类中，在这里就先来查看他们对外所提供了哪些接口。首先观察一下两个基类接口BasePresenter和BaseView。

**BasePresenter**
```java
package com.example.android.architecture.blueprints.todoapp;

public interface BasePresenter {

    void start();

}
```
　　在BasePresenter中只存在一个start方法。这个方法一般所执行的任务是在Presenter中从Model层获取数据，并调用View接口显示。这个方法一般是在Fragment中的onResume方法中调用。

**BaseView**
```java
package com.example.android.architecture.blueprints.todoapp;

public interface BaseView<T> {

    void setPresenter(T presenter);

}
```
　　在BaseView中只有一个setPresenter方法，对于View层会存在一个Presenter对象。而setPresenter正是对View中的Presenter进行初始化。

**AddEditTaskContract**

　　在Android官方给出的MVP架构当中对于Presenter接口和View接口提供的形式与我们平时在网上所见的有所不同。在这里将Presenter中的接口和View的接口都放在了AddEditTaskContract类里面。这样一来我们能够更清晰的看到在Presenter层和View层中有哪些功能，方便我们以后的维护。下面就来看一下这个AddEditTaskContract类。
```java
public interface AddEditTaskContract {

    interface View extends BaseView<Presenter> {

        void showEmptyTaskError();

        void showTasksList();

        void setTitle(String title);

        void setDescription(String description);

        boolean isActive();
    }

    interface Presenter extends BasePresenter {

        void createTask(String title, String description);

        void updateTask( String title, String description);

        void populateTask();
    }
}
```
　　在这里很清晰的可以看出在View层中处理了一些数据显示的操作，而在Presenter层中则是对Task保存，更新等操作。
### **Presenter层的实现**
　　下面就来看一下在Presenter是如何实现的。
```java
public class AddEditTaskPresenter implements AddEditTaskContract.Presenter,
        TasksDataSource.GetTaskCallback {
        
    ......
    
    public AddEditTaskPresenter(@Nullable String taskId, @NonNull TasksDataSource tasksRepository,
            @NonNull AddEditTaskContract.View addTaskView) {
        mTaskId = taskId;
        mTasksRepository = checkNotNull(tasksRepository);
        mAddTaskView = checkNotNull(addTaskView);

        mAddTaskView.setPresenter(this);
    }

    @Override
    public void start() {
        if (mTaskId != null) {
            populateTask();
        }
    }

    ......

    @Override
    public void populateTask() {
        if (mTaskId == null) {
            throw new RuntimeException("populateTask() was called but task is new.");
        }
        mTasksRepository.getTask(mTaskId, this);
    }

    @Override
    public void onTaskLoaded(Task task) {
        // The view may not be able to handle UI updates anymore
        if (mAddTaskView.isActive()) {
            mAddTaskView.setTitle(task.getTitle());
            mAddTaskView.setDescription(task.getDescription());
        }
    }
    ......
}
```
　　在这里可以看到在AddEditTaskPresenter中它不仅实现了自己的Presenter接口，也实现了GetTaskCallback的回调接口。并且在Presenter中包含了Model层TasksDataSource的对象mTasksRepository和View层AddEditTaskContract.View的对象mAddTaskView。于是整个业务逻辑的处理就担负在Presenter的身上。
　　从Presenter的业务处理中可以看出，首先调用Model层的接口getTask方法，通过TaskId来查询Task。在查询到Task以后，由于在Presenter层中实现了Model层的回调接口GetTaskCallback。这时候在Presenter层中就通过onTaskLoaded方法获取到Task对象，最后通过调用View层接口实现了数据的展示。
### **View层的实现**
　　对于View的实现是在Fragment中，而在Activity中则是完成对Fragment的添加，Presenter的创建操作。下面首先来看一下AddEditTaskActivity类。
```java
public class AddEditTaskActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.addtask_act);

        ......
        
        if (addEditTaskFragment == null) {
            addEditTaskFragment = AddEditTaskFragment.newInstance();
            ......
            ActivityUtils.addFragmentToActivity(getSupportFragmentManager(),
                    addEditTaskFragment, R.id.contentFrame);
        }

        // Create the presenter
        new AddEditTaskPresenter(
                taskId,
                Injection.provideTasksRepository(getApplicationContext()),
                addEditTaskFragment);
    }

    ......
}
```
　　对于Activity的提供的功能也是非常的简单，首先创建Fragment对象并将其添加到Activity当中。之后创建Presenter对象，并将Fragment也就是View传递到Presenter中。
　　下面再来看一下View的实现，也就是Fragment。
```java
public class AddEditTaskFragment extends Fragment implements AddEditTaskContract.View {

    ......
 
    @Override
    public void onResume() {
        super.onResume();
        mPresenter.start();
    }

    @Override
    public void setPresenter(@NonNull AddEditTaskContract.Presenter presenter) {
        mPresenter = checkNotNull(presenter);
    }

    ......

    @Override
    public void setTitle(String title) {
        mTitle.setText(title);
    }

    ......
}

```
　　在这对于源码就不在过多贴出。在Fragment中，通过setPresenter获取到Presenter对象。并通过调用Presenter中的方法来实现业务的处理。而在Fragment中则只是对UI的一些操作。这样一来对于Fragment类型的代码减少了很多，并且逻辑更加清晰。
　　我们注意到View层的实现是通过Fragment来完成的。对于View的实现为什么要采用Fragment而不是Activity。来看一下官方是如何解释的。
```text
 The separation between Activity and Fragment fits nicely with this implementation of MVP: the Activity is the overall controller that creates and connects views and presenters.
Tablet layout or screens with multiple views take advantage of the Fragments framework.
```
　　在这里官方对于采用Fragment的原因给出了两种解释。

 - 通过Activity和Fragment分离非常适合对于MVP架构的实现。在这里将Activity作为全局的控制者将Presenter于View联系在一起。
 - 采用Fragment更有利于平板电脑的布局或者是多视图屏幕。

# **总结**
　　通过MVP架构的使用可以看出对于各个层次之间的职责更加单一清晰，同时也很大程度上降低了代码的耦合度。对于官方MVP架构示例，google也明确表明对于他们所给出的这些架构示例只是作为参考，而不是一个标准。所以对于基础的MVP架构有更大的扩展空间。例如综合google给出的示例。我们可以通过在MVP架构的基础上使用dagger2，rxJava等来构建一个Clean架构。也是一个很好的选择。