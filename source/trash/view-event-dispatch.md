---
title:  Android事件分发机制(下)—View的事件处理
date: 2016-05-15 19:23
category: [Android]
tags: [view]
comments: true
---
　　在上篇文章[Android中的事件分发机制(上)——ViewGroup的事件分发](http://blog.csdn.net/ljd2038/article/details/51394605)中，对ViewGroup的事件分发进行了详细的分析。在文章的最后ViewGroup的dispatchTouchEvent方法调用dispatchTransformedTouchEvent方法成功将事件传递给ViewGroup的子View。并交由子View进行处理。那么现在就来分析一下子View接收到事件以后是如何处理的。<!--more-->
# **View的事件处理**
　　对于这里描述的View，它是ViewGroup的父类，并不包含任何的子元素。这也就意味着View无法再次向下对事件进行分发操作，因此在View中并不存在onInterceptTouchEvent方法，也不会对事件做出拦截操作。它所做的事情就是对所接收的事件进行处理。下面就开看一下View如何对事件进行处理的。
　　既然ViewGroup将事件交由View的dispatchTouchEvent方。那么首先在这里就来看一下dispatchTouchEvent里面做了什么事情。
```java
public boolean dispatchTouchEvent(MotionEvent event) {

    ......

    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    
    ......

    return result;
}
```
　　在View的dispatchTouchEvent方法中对事件处理的核心部分体现在上述代码中。onFilterTouchEventForSecurity方法表示当前接收事件的view是否处于被遮盖状态，View处于被遮盖状态表示当前View不位于顶部，该view被其它View所覆盖。如果当前View被遮盖，那么该View不会对事件进行处理。
```java
public interface OnTouchListener {
    boolean onTouch(View v, MotionEvent event);
}

public void setOnTouchListener(OnTouchListener l) {
    getListenerInfo().mOnTouchListener = l;
}

ListenerInfo getListenerInfo() {
    if (mListenerInfo != null) {
        return mListenerInfo;
    }
    mListenerInfo = new ListenerInfo();
    return mListenerInfo;
}
```
　　在结合上述一段代码可以看到，通过setOnTouchListener方法设置OnTouchListener以后，若是当前View处于可用状态，那么条件li != null && li.mOnTouchListener !=null && (mViewFlags & ENABLED_MASK) == ENABLED必然为true。这时候程序便会回调OnTouchListener中的onTouch方法，若是在onTouch方法中返回true，便不会在执行View的onTouchEvent方法。从这里我们能够看到，一旦设置了OnTouchListener，那么OnTouchListener的优先级要高于onTouchEvent。
　　有一点需要注意，在程序中设置了OnTouchListener以后，对于OnTouchListener中的onTouch的返回值并不代表View中的dispatchTouchEvent方法所返回的值。在onTouch方法返回true的时候，表示事件成功被当前View所消耗，这时候result被置为true并且不再执行onTouchEvent，所以dispatchTouchEvent也就返回true。可是一旦在onTouch方法中返回false。这时候便会调用onTouchEvent方法，如果事件被onTouchEvent成功处理，并返回true，result依然会被置为true，dispatchTouchEvent自然而然的也就返回true。
　　下面在进入View的onTouchEvent方法一探究竟。对于onTouchEvent方法里的内容比较多，在这里分段查看。
```java
if ((viewFlags & ENABLED_MASK) == DISABLED) {
    if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        setPressed(false);
    }
    // A disabled view that is clickable still consumes the touch
    // events, it just doesn't respond to them.
    return (((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
}
```
　　在这里可以看出对于不可用的View，如果他们的一些点击事件可用的话，依然能够成功的消费事件，只是它不会为该事件做出响应。对于View的这些点击之间默认为不可用，但是对于不同的的View他们的默认值不一样。例如在ImageView中点击事件依然为不可用，但是在Button中点击事件为可用。当然如果手动为它们设置监听事件，那么这些监听事件都将会自动被设为可用状态。从如下源码中可以看出。
```java
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}

public void setOnLongClickListener(@Nullable OnLongClickListener l) {
    if (!isLongClickable()) {
        setLongClickable(true);
    }
    getListenerInfo().mOnLongClickListener = l;
}

public void setOnContextClickListener(@Nullable OnContextClickListener l) {
    if (!isContextClickable()) {
        setContextClickable(true);
    }
    getListenerInfo().mOnContextClickListener = l;
}
```
　　下面接着看OnTouchEvent里面代码。
```java
if (mTouchDelegate != null) {
    if (mTouchDelegate.onTouchEvent(event)) {
        return true;
    }
}
```
　　这里首先判断是否对事件设置了代理，如果对事件设置了代理，便会执行TouchDelegate的onTouchEvent方法。mTouchDelegate默认值为null，可以通过View的setTouchDelegate方法来设置代理。对于TouchDelegate在后续的文章中在进行详细分析，在这里就不在过多描述。
　　最后看一下View是如何处理事件的，对于接收的事件整个处理过程比较复杂，在这里就从宏观上来整体看一下它的处理机制。
```java
if (((viewFlags & CLICKABLE) == CLICKABLE ||
        (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
        (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
    switch (action) {
        case MotionEvent.ACTION_UP:
            
            ......
            
                if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                    // This is a tap, so remove the longpress check
                    removeLongPressCallback();

                    // Only perform take click actions if we were in the pressed state
                    if (!focusTaken) {
                        // Use a Runnable and post this rather than calling
                        // performClick directly. This lets other visual state
                        // of the view update before click actions start.
                        if (mPerformClick == null) {
                            mPerformClick = new PerformClick();
                        }
                        if (!post(mPerformClick)) {
                            performClick();
                        }
                    }
                }

            ......

            break;

        ......

    }

    return true;
}
```
　　如果View的点击事件处于可用状态的话，便会对于这些事件进行处理，并且返回true。当一个事件序列完成以后调用performClick方法,下面看下这个performClick方法。
```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```
　　从上面代码中可以看出，如果我们设置了OnClickListener，便会调用它的onClick方法。从这一路下来我们可以看出对于View的onClick事件，在最后才会被调用，可见onClick的优先级是最低的。
# **总结**
　　在这里对View的事件处理做一下总结。在ViewGroup将事件分发到View以后。在View里面通过OnTouchListener的onTouch和View中的onTouchEvent这两个方法对事件进行处理。对于onTouch方法是View提供给用户的，方便用户自己处理触摸事件，而onTouchEvent是Android系统自己实现的接口。若是用户设置了OnTouchListener，Android系统会首先调用OnTouchListener的onTouch方法。若是在onTouch方法中返回true，就不在执行View的onTouchEvent方法。只有在onTouch中返回了false才会执行onTouchEvent。
