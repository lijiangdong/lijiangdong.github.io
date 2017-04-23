---
title:   Java设计模式之代理模式
date: 2016-05-17 23:56
category: [Java]
tags: [设计模式]
comments: true
---

　　代理模式(Proxy Pattern)也称作为委托模式。在生活中我们也是处处可见，例如通过设置代理进行上网，委托律师来打官司，又或者是代理某个品牌来创业等等。而在开发中也是被经常被使用到的一种设计模式。对于代理模式的定义为:**为其他对象提供一种代理以控制对这个对象的访问**。对于代理模式可以分为两大类，分别是静态代理和动态代理。下面就开始对这两种类型的代理模式进行详细介绍。<!--more-->
# **静态代理**
　　在这里通过一个场景来展开描述代理模式。现在我想按照自己想要的配置来组装一台电脑。于是我将主板，内存条，显卡，机箱，显示器等所有的配件都买回来了。可是这个时候遇到了一个难题，我不会组装呀。于是我带着这些配件去找专业人士为我组装电脑。对于电脑的组装内部细节部分步骤比较多。在这里作为举例说明，将组装电脑的过程主要分为三个步骤。这三个步骤分别是安装主机，设置显示器，安装操作系统。
## **简单实现**
　　那么就针对上述场景，来实现代理模式。首先来看一下它的UML类图。
![这里写图片描述](http://img.blog.csdn.net/20160517085234261)
　　对于这个类图可以看出，对于上述场景代理模式的实现也是非常的简单。在这里定义了一个组装电脑的接口。在这个接口里面是一些电脑组装的实现步骤。而后又定义了一个客户类和一个专业人士的代理类，分别实现这个接口。之后组装电脑的这些事情全部都交由这个专业人士代理类来实现。下面就来看一下具体的代码实现。
　　这里定义一个组装电脑的接口,在接口里面分别定义了对主机的安装，显示屏的设置和操作系统的安装。
```java
public interface IHomebuiltComputer {
	void installHost();
	void setDisplay();
	void installOS();
}
```
　　下面这个是客户类，实现了IHomebuiltComputer接口。是一些对电脑安装的具体操作步骤。
```java
public class Customer implements IHomebuiltComputer{

	@Override
	public void installHost() {
		System.out.println("安装主机");
	}

	@Override
	public void setDisplay() {
		System.out.println("安装显示器");
	}

	@Override
	public void installOS() {
		System.out.println("安装操作系统");
	}

}
```
　　最后看一下这个代理类。这个代理类同样也继承自IHomebuiltComputer接口。在这个代理类中还持有被代理的引用，也就是Customer类。通过调用被代理类Customer的方法，来实现代理功能。
```java
public class ProfessionalProxy implements IHomebuiltComputer{

	private IHomebuiltComputer customer;
	
	public ProfessionalProxy(IHomebuiltComputer customer) {
		this.customer = customer;
	}
	
	@Override
	public void installHost() {
		this.customer.installHost();
	}

	@Override
	public void setDisplay() {
		this.customer.setDisplay();
	}

	@Override
	public void installOS() {
		this.customer.installOS();
	}

}
```
　　下面就通过一个Client类，来看一下具体的调用执行关系。
```java
public class Client {

	public static void main(String[] args) {
		IHomebuiltComputer customer = new Customer();
		IHomebuiltComputer proxy = new ProfessionalProxy(customer);
		proxy.installHost();
		proxy.setDisplay();
		proxy.installOS();
	}

}
```
　　在这个Client类中，可以看到，所有的事情都交由代理类处理了。然后在看一下运行结果。
![这里写图片描述](http://img.blog.csdn.net/20160517092028661)
　　对于这个代理类中，也能够实现不同的接口来完成不同功能的整合。在这里就不在详细赘述。下面就来看一下代理模式的通用代码。
## **静态代理通用模板**
　　在这里首先来看一下代理模式的通用类图。
　<div align="center">
　　<img  width=70% src="http://img.blog.csdn.net/20160517140153611"/>
　　<div>

　　ISubject（抽象主题类）:在这个类中，主要声明真实主题类和代理类的共同方法。它既可以是一个接口也可以是一个抽象类。
　　RealSubject（真实主题类）:这是被代理类，具体的业务实现都在这个类中。
　　Proxy（代理类）:这是一个代理类，在这个代理类中它持有真实主题类的对象。通过调用真实主题类的方法来实现代理。
　　下面来看一下通用代码：
　　**抽象主题类**
```java
public interface ISubject {
	public void request();
}
```
　　**实现抽象主题的真实主题类**
```java
public class RealSubject implements ISubject{

	@Override
	public void request() {
		System.out.println("我是真实对象");
	}
}
```
　　**代理类**
```java
public class Proxy implements ISubject{

	private ISubject subject;
	
	public Proxy(ISubject subject) {
		this.subject = subject;
	}
	
	@Override
	public void request() {
		this.subject.request();
	}

}
```
　　**客户类**
```java
public class Client {
	public static void main(String[] args) {
		ISubject subject = new RealSubject();
		Proxy proxy = new Proxy(subject);
		proxy.request();
	}
}
```
　　以上就是静态代理类的实现。在前面的一系列有关Android的IPC机制文章中，在分析AIDL的实现原理的时，系统根据用户写的AIDL接口自动创建了Java接口，而在这个Java接口中通过Binder与Service建立连接的过程中使用的就是静态代理模式。详细内容见[Android的IPC机制（二）——AIDL实现原理简析](http://blog.csdn.net/ljd2038/article/details/50705935)这篇文章。
　　从上面的分析出可以看出，对静态代理模式，代理者的代码都是通过程序员或者是通过一些自动化的工具生成的固定代码(例如刚才所提到的AIDL的实现)然后再对他们进行编译。这样也就意味着在我们的代码运行之前代理类的Class文件就已经存在了。
　　
# **动态代理**
　　在这里再来看一下动态代理。对于动态代理，在程序运行时，动态的创建一个代理类以及代理类的对象，当然也能够创建一个实现多个接口的代理类。在动态代理类中使用了Java中的反射机制，来生成了代理类的对象。并且Java也为我们提供了一个动态代理接口InvocationHandler，对被代理类的方法进行代理。在使用动态代理类时，必须实现InvocationHandler接口。
　　对于InvocationHandler接口，我们必须手动实现的它的invoke方法，正是在InvocationHandler中的invoke方法中完成了对真实方法的调用。也就是说所有被代理的方法都是交由这个InvocationHandler进行处理。在这里首先看一下这个InvocationHandler接口。

```java
package java.lang.reflect;
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```
　　从上面接口中我们可以看出在InvocationHandler接口中只有一个invoke方法。在invoke方法中，其中proxy参数表示代理类的实例；method参数表示被调用的方法对象;args参数表示被调用方法对象的参数。
　　下面就以上面的情形通过动态代理来实现。在这里需要添加一个InvocationHandler接口，以及它的实现类。于是在这里对上面的类图进行一下修改。
　　
 　<div align="center">
　　<img  width=70% src="http://img.blog.csdn.net/20160517225613675"/>
　　<div>

　　下面就来看一下具体的代码实现。对于上面的IHomebuiltComputer接口和Customer不需要改变。所以在这里只需要一个实现InvocationHandler接口的CustomerHandler类。然后在对Client做一次修改即可。在这里看一下CustomerHandler类。
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class CustomerHandler implements InvocationHandler{
	
	private Object obj = null;
	
	public CustomerHandler(Object object) {
		this.obj = object;
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		// TODO Auto-generated method stub
		return method.invoke(this.obj, args);
	}

}
```
　　再看一下Client类。
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

public class Client {

	public static void main(String[] args) {

		IHomebuiltComputer homebuiltComputer = new Customer();
		InvocationHandler handler = new CustomerHandler(homebuiltComputer);
		IHomebuiltComputer proxy = (IHomebuiltComputer)Proxy.newProxyInstance(homebuiltComputer.getClass().getClassLoader(), 
				new Class[]{IHomebuiltComputer.class}, handler);
		proxy.installHost();
		proxy.setDisplay();
		proxy.installOS();
	}

}
```
　　这样就通过动态代理模式实现了上述的情形。对于运行结果与上面一样，在这里就不在贴出。下面来看一下动态代理的通用模板。

　　通过DynamicProxy，RealSubject可以在运行时动态改变，需要控制的接口Subject也可以在运行时改变，控制的方式DynamicSubject类也可以动态改变，从而实现了非常灵活的动态代理关系。
## **动态代理通用模板**
　　在这里再看一下动态代理的通用类图。
<div align="center">
　　<img  width=70% src="http://img.blog.csdn.net/20160517232856479"/>
　　<div>

　　在这里通过DynamicProxy来创建一个Proxy对象。并且它依赖于InvocationHandler。并且是InvocationHandler来进行实际的业务处理。下面就来看一下动态代理的通用代码。
　　**抽象主题类**
```java
public interface ISubject {
	void request();
}
```
　　**真实主题类**
```java
public class RealSubject implements ISubject{

	@Override
	public void request() {
		//业务处理
	}
}
```
　　**InvocationHandler的实现类**
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class MyInvocationHandler implements InvocationHandler{

	private Object target = null;
	
	public MyInvocationHandler(Object object) {
		this.target = object;
	}
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		
		return method.invoke(this.target, args);
	}

}
```
　　**DynamicProxy类**
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

public class DynamicProxy<T> {
	
	@SuppressWarnings("unchecked")
	public static <T> T newProxyInstance(ClassLoader classLoader, Class<?>[] interfaces,InvocationHandler handler){
		
		return (T)Proxy.newProxyInstance(classLoader, interfaces, handler);
	}
}
```
　　**Client类**
```java
public class Client {

	public static void main(String[] args) {
		ISubject subject = new RealSubject();
		
		InvocationHandler handler = new MyInvocationHandler(subject);
		ISubject proxy = DynamicProxy.newProxyInstance(subject.getClass().getClassLoader(), subject.getClass().getInterfaces(), handler);
		proxy.request();
	}
}
```
# **总结**
　　在这里对代理模式做一下总结。对于代理模式来说，代理的存在对于调用者来说是透明的，而调用者看到的只是接口。对于代理对象来说，可以封装一些内部的逻辑处理，如访问控制、远程通信、日志、缓存等等。