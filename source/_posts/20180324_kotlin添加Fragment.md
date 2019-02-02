---
title:  聊一聊Kotlin扩展函数run,with,let,also和apply的使用和区别
date: 2018-03-24 18：47
category: [Android]
tags: [Kotlin]
comments: true
---

# **综述**

在上面文章**聊一聊Kotlin扩展函数run,with,let,also和apply的使用和区别**中讲解Kotlin的几个扩展函数的使用和区别。那么在这篇文章中去自己定义一些扩展函数来更加优雅的去将添加Fragment到Activity中。<!--more-->

# **回顾Fragment使用**

在使用Kotlin之前，首先回顾一下在Java中是如何添加一个Fragment到Activity当中的。
```
FragmentManager manager = getSupportFragmentManager();
FragmentTransaction transaction = manager.beginTransaction();
transaction.add(frameId, fragment);
transaction.commit();
```
上面这段代码是在熟悉不过了，在项目中为了避免大量的重复书写上面那段代码，通常会创建一个ActivityUtil的类，在这个类中写一个static方法来封装上面那段代码。
```
public static void addFragmentToActivity(FragmentManager manager, Fragment fragment, int frameId) {
    
    FragmentTransaction transaction = manager.beginTransaction();
    transaction.add(frameId, fragment);
    transaction.commit();
    
}
```

这时候在Activity当中便能够直接使用上面的方法：

```
ActivityUtil.addFragmentToActivity(
        getSupportFragmentManager(), fragment, R.id.frag_container);
```

在介绍完使用Java来添加Fragment后，下面就来看一下如何使用Kotlin更优雅的来添加Fragment。

# **Kotlin添加Fragment**

使用Kotlin完成Fragment的天假只需要两步即可：

## 消除beginTransaction() and commit()方法

对于Fragment的操作每次都需要beginTransaction()和commit()，如果在开发中少写了一个commit()方法，或许会花费大量时间进行调试找到问题所在。所以省去beginTransaction()和commit()代码的书写还是很有必要的。

在Kotlin中，可以为FragmentManager类写一个扩展函数，将一个带有接收者函数的lambda作为扩展函数的参数。下面简单介绍一下相关知识点：

>1. 扩展函数([Extension functions](https://kotlinlang.org/docs/reference/extensions.html))
扩展函数能够向已经存在的类中添加新的函数或属性，也包含第三方库或者SDK中的类。在函数内部，可以不使用任何限定符来访问类的公共函数和属性，就像这个函数在类的内部一样。

 >2. 高阶函数([Higher-Order Functions](https://kotlinlang.org/docs/reference/lambdas.html))
高阶函数是将函数作为参数，或返回一个函数的函数。可以传递函数或者从函数中返回一个函数。

 >3. 带接收者的Lambda函数（[Function Literals with Receiver](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver)）
可以说是以上两个的组合，一个高阶函数将一个扩展函数作为它的参数。将lambda表达式中作为参数传递，并且我们可以访问接收者的函数和属性，就好像lambda函数在接收者对象的内部一样。

现在书写一个FragmentManager的扩展函数，它将一个带有接受者的Lambda函数作为传入的参数，而这个FragmentTransaction就是接收者对象。

```
inline fun FragmentManager.inTransaction(func: FragmentTransaction.() -> Unit) {
    val fragmentTransaction = beginTransaction()
    fragmentTransaction.func()
    fragmentTransaction.commit()
}
```

上面这段代码是一个扩展函数，他接收的参数是一个带有接收者的Lambda函数。这个Lambda函数没有任何参数并且返回的是Unit。在这个inTransaction扩展函数体内首先调用beginTransaction()获取FragmentTransaction对象，然后在执行Lambda函数之后执行commit()方法。

那么就可以通过一下代码将我们的Fragment添加到Activity当中：

```
supportFragmentManager.inTransaction {
    add(R.id.frameLayoutContent, fragment)
}
```

在这里可以看出来在Lambda函数当中可以直接操作FragmentTransaction中的方法，并且没有任何的使用限制。因为这个Lambda函数本身就是FragmentTransaction的一个扩展函数。

在使用上面的扩展函数当中不需要每次再执行beginTransaction()和commit()方法。甚至我们还能够在inTransaction方法中执行多次操作。

```
supportFragmentManager.inTransaction {
    remove(fragmentA)    
    add(R.id.frameLayoutContent, fragmentB)
}
```

现在再次对inTransaction函数进行一次升级，可以将传入的Lambda函数添加一个返回值，返回FragmentTransaction对象。这样会使得我们的代码更加简洁。

```
inline fun FragmentManager.inTransaction(func: FragmentTransaction.() -> FragmentTransaction) {
    beginTransaction().func().commit()
}
```

## 使用扩展函数来替代ActivityUtil

现在就可以使用上面的inTransaction方法来写一些FragmentActivity的扩展函数来取代Java中的ActivityUtil类。那么就以对Fragment的add和replace操作为例写两个FragmentActivity的扩展函数addFragment和replaceFragment。

```
fun FragmentActivity.addFragment(fragment: Fragment, frameId: Int){
    supportFragmentManager.inTransaction { add(frameId, fragment) }
}


fun FragmentActivity.replaceFragment(fragment: Fragment, frameId: Int) {
    supportFragmentManager.inTransaction{replace(frameId, fragment)}
}
```

由于这些扩展函数是FragmentActivity的扩展函数。所以在这些扩展函数内部就能够直接访问到FragmentActivity中的supportFragmentManager。

使用上面的扩展函数，就可以通过一行代码在Activity中添加Fragment。
```
addFragment(fragment, R.id.fragment_container)

replaceFragment(fragment, R.id.fragment_container)
```

# **总结**

通过上篇文章了解了Kotlin一些扩展函数。而在这篇文章中去通过对Fragment添加去了解扩展函数的意义以及对高阶函数的使用。当然通过扩展函数也能够更佳的完善一些SDK中一些不合理的API。还能够写出更漂亮简洁的代码。