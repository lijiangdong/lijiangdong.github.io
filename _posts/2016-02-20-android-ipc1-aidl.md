---
layout: post
title: Sample Post
description: "Just about everything you'll need to style in the theme: headings, paragraphs, blockquotes, tables, code blocks, and more."
modified: 2014-12-24
tags: [sample post]
categories: [intro]
---
# Android中的事件分发机制--ViewGroup的事件分发
#**综述**
　　Android中的事件分发机制也就是View与ViewGroup的对事件的分发与处理。在ViewGroup的内部包含了许多View，而ViewGroup继承自View，所以ViewGroup本身也是一个View。对于事件可以通过ViewGroup下发到它的子View并交由子View进行处理，而ViewGroup本身也能够对事件做出处理。下面就来详细分析一下ViewGroup对时间的分发处理。
#**MotionEvent**
　　当手指接触到屏幕以后，所产生的一系列的事件中，都是由以下三种事件类型组成。
　　1. **ACTION_DOWN:** 手指按下屏幕
　　2. **ACTION_MOVE:** 手指在屏幕上移动
　　3. **ACTION_UP:** 手指从屏幕上抬起
　　例如一个简单的屏幕触摸动作触发了一系列Touch事件:ACTION_DOWN->ACTION_MOVE->...->ACTION_MOVE->ACTION_UP
　　对于Android中的这个事件分发机制，其中的这个事件指的就是MotionEvent。而View的对事件的分发也是对MotionEvent的分发操作。可以通过getRawX和getRawY来获取事件相对于屏幕左上角的横纵坐标。通过getX()和getY()来获取事件相对于当前View左上角的横纵坐标。
#**三个重要方法**
**public boolean dispatchTouchEvent(MotionEvent ev)**

　　这是一个对事件分发的方法。如果一个事件传递给了当前的View，那么当前View一定会调用该方法。对于dispatchTouchEvent的返回类型是boolean类型的，返回结果表示是否消耗了这个事件，如果返回的是true，就表明了这个View已经被消耗，不会再继续向下传递。　　
　　
**public boolean onInterceptTouchEvent(MotionEvent ev)**

　　该方法存在于ViewGroup类中，对于View类并无此方法。表示是否拦截某个事件，ViewGroup如果成功拦截某个事件，那么这个事件就不在向下进行传递。对于同一个事件序列当中，当前View若是成功拦截该事件，那么对于后面的一系列事件不会再次调用该方法。返回的结果表示是否拦截当前事件，默认返回false。由于一个View它已经处于最底层，它不会存在子控件，所以无该方法。
　　
**public boolean onTouchEvent(MotionEvent event)**

　　这个方法被dispatchTouchEvent调用，用来处理事件，对于返回的结果用来表示是否消耗掉当前事件。如果不消耗当前事件的话，那么对于在同一个事件序列当中，当前View就不会再次接收到事件。
　　
#**View事件分发流程图**
　　对于事件的分发，在这里先通过一个流程图来看一下整个分发过程。
![这里写图片描述](http://img.blog.csdn.net/20160511091631513)
#**ViewGroup事件分发源码分析**
　　根据上面的流程图现在就详细的来分析一下ViewGroup事件分发的整个过程。
　　手指在触摸屏上滑动所产生的一系列事件，当Activity接收到这些事件通过调用Activity的dispatchTouchEvent方法来进行对事件的分发操作。下面就来看一下Activity的dispatchTouchEvent方法。
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
　　通过getWindow().superDispatchTouchEvent(ev)这个方法可以看出来，这个时候Activity又会将事件交由Window处理。Window它是一个抽象类，它的具体实现只有一个PhoneWindow，也就是说这个时候，Activity将事件交由PhoneWindow中的superDispatchTouchEvent方法。现在跟踪进去看一下这个superDispatchTouchEvent代码。
```
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```
　　这里面的mDecor它是一个DecorView，DecorView它是一个Activity的顶级View。它是PhoneWindow的一个内部类，继承自FrameLayout。于是在这个时候事件又交由DecorView的superDispatchTouchEvent方法来处理。下面就来看一下这个superDispatchTouchEvent方法。
```
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```
　　在这个时候就能够很清晰的看到DecorView它调用了父类的dispatchTouchEvent方法。在上面说到DecorView它继承了FrameLayout，而这个FrameLayout又继承自ViewGroup。所以在这个时候事件就开始交给了ViewGroup进行处理了。下面就开始详细看下这个ViewGroup的dispatchTouchEvent方法。由于dispatchTouchEvent代码比较长，在这里就摘取部分代码进行说明。
```
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
```
　　从上面代码可以看出，在dispatchTouchEvent中，会对接收的事件进行判断，当接收到的是ACTION_DOWN事件时，便会清空事件分发的目标和状态。然后执行resetTouchState方法重置了触摸状态。下面就来看一下这两个方法。
　　**1.  cancelAndClearTouchTargets(ev)**
```
private TouchTarget mFirstTouchTarget;

......

private void cancelAndClearTouchTargets(MotionEvent event) {
    if (mFirstTouchTarget != null) {
        boolean syntheticEvent = false;
        if (event == null) {
            final long now = SystemClock.uptimeMillis();
            event = MotionEvent.obtain(now, now,
                    MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
            event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
            syntheticEvent = true;
        }

        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            resetCancelNextUpFlag(target.child);
            dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
        }
        clearTouchTargets();

        if (syntheticEvent) {
            event.recycle();
        }
    }
}
```
　　在这里先介绍一下mFirstTouchTarget，它是TouchTarget对象，TouchTarget是ViewGroup的一个内部类，TouchTarget采用链表数据结构进行存储View。而在这个方法中主要的作用就是清空mFirstTouchTarget链表并将mFirstTouchTarget设为空。
　　**2.  resetTouchState()**
```
private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}
```
　　在这里介绍一下FLAG_DISALLOW_INTERCEPT标记，这是禁止ViewGroup拦截事件的标记，可以通过requestDisallowInterceptTouchEvent方法来设置这个标记，当设置了这个标记以后，ViewGroup便无法拦截除了ACTION_DOWN以外的其它事件。因为在上面代码中可以看出，当事件为ACTION_DOWN时，会重置FLAG_DISALLOW_INTERCEPT标记。
　　那么下面就再次回到dispatchTouchEvent方法中继续看它的源代码。
```
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
        || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
    } else {
        intercepted = false;
    }
} else {
    // There are no touch targets and this action is not an initial down
    // so this view group continues to intercept touches.
    intercepted = true;
}
```
　　这段代码主要就是ViewGroup对事件是否需要拦截进行的判断。下面先对mFirstTouchTarget是否为null这两种情况进行说明。当事件没有被拦截时，ViewGroup的子元素成功处理事件后，mFirstTouchTarget会被赋值并且指向其子元素。也就是说这个时候mFirstTouchTarget!=null。可是一旦事件被拦截，mFirstTouchTarget不会被赋值，mFirstTouchTarget也就为null。
　　在上面代码中可以看到根据actionMasked==MotionEvent.ACTION_DOWN||mFirstTouchTarget!=null这两个情况进行判断事件是否需要拦截。对于actionMasked==MotionEvent.ACTION_DOWN这个条件很好理解，对于mFirstTouchTarget!=null的两种情况上面已经说明。那么对于一个事件序列，当事件为MotionEvent.ACTION_DOWN时，会重置FLAG_DISALLOW_INTERCEPT，也就是说!disallowIntercept一定为true，必然会执行onInterceptTouchEvent方法，对于onInterceptTouchEvent方法默认返回为false，所以需要ViewGroup拦截事件时，必须重写onInterceptTouchEvent方法，并返回true。这里有一点需要注意，对于一个事件序列，一旦序列中的某一个事件被成功拦截，执行了onInterceptTouchEvent方法，也就是说onInterceptTouchEvent返回值为true，那么该事件之后一系列事件对于条件actionMasked==MotionEvent.ACTION_DOWN||mFirstTouchTarget!=null必然为false，那么这个时候该事件序列剩下的一系列事件将会被拦截，并且不会执行onInterceptTouchEvent方法。于是在这里得出一个结论：**对于一个事件序列，当其中某一个事件成功拦截时，那么对于剩下的一些列事件也会被拦截，并且不会再次执行onInterceptTouchEvent方法**
　　下面再来看一下对于ViewGroup并没有拦截事件是如何进行处理的。
```
final int childrenCount = mChildrenCount;
if (newTouchTarget == null && childrenCount != 0) {
    final float x = ev.getX(actionIndex);
    final float y = ev.getY(actionIndex);
    // Find a child that can receive the event.
    // Scan children from front to back.
    final ArrayList<View> preorderedList = buildOrderedChildList();
    final boolean customOrder = preorderedList == null
            && isChildrenDrawingOrderEnabled();
    final View[] children = mChildren;
    for (int i = childrenCount - 1; i >= 0; i--) {
        final int childIndex = customOrder
                ? getChildDrawingOrder(childrenCount, i) : i;
        final View child = (preorderedList == null)
                ? children[childIndex] : preorderedList.get(childIndex);

        // If there is a view that has accessibility focus we want it
        // to get the event first and if not handled we will perform a
        // normal dispatch. We may do a double iteration but this is
        // safer given the timeframe.
        if (childWithAccessibilityFocus != null) {
            if (childWithAccessibilityFocus != child) {
                continue;
            }
            childWithAccessibilityFocus = null;
            i = childrenCount - 1;
        }

        if (!canViewReceivePointerEvents(child)
                || !isTransformedTouchPointInView(x, y, child, null)) {
            ev.setTargetAccessibilityFocus(false);
            continue;
        }

        newTouchTarget = getTouchTarget(child);
        if (newTouchTarget != null) {
            // Child is already receiving touch within its bounds.
            // Give it the new pointer in addition to the ones it is handling.
            newTouchTarget.pointerIdBits |= idBitsToAssign;
            break;
        }

        resetCancelNextUpFlag(child);
        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
            // Child wants to receive touch within its bounds.
            mLastTouchDownTime = ev.getDownTime();
            if (preorderedList != null) {
                // childIndex points into presorted list, find original index
                for (int j = 0; j < childrenCount; j++) {
                    if (children[childIndex] == mChildren[j]) {
                        mLastTouchDownIndex = j;
                        break;
                    }
                }
            } else {
                mLastTouchDownIndex = childIndex;
            }
            mLastTouchDownX = ev.getX();
            mLastTouchDownY = ev.getY();
            newTouchTarget = addTouchTarget(child, idBitsToAssign);
            alreadyDispatchedToNewTouchTarget = true;
            break;
        }

        // The accessibility focus didn't handle the event, so clear
        // the flag and do a normal dispatch to all children.
        ev.setTargetAccessibilityFocus(false);
    }
    if (preorderedList != null) preorderedList.clear();
}

```
　　对于这段代码虽然说比较长，但是在这里面的逻辑去不是很复杂。首先获取当前ViewGroup中的子View和ViewGroup的数量。然后对该ViewGroup中的元素进行逐步遍历。在获取到ViewGroup中的子元素后，判断该元素是否能够接收触摸事件。子元素若是能够接收触摸事件，并且该触摸坐标在子元素的可视范围内的话，便继续向下执行。否则就continue。对于衡量子元素能否接收到触摸事件的标准有两个：子元素是否在播放动画和点击事件的坐标是否在子元素的区域内。
　　一旦子View接收到了触摸事件，然后便开始调用dispatchTransformedTouchEvent方法对事件进行分发处理。对于dispatchTransformedTouchEvent方法代码比较多，现在只关注下面这五行代码。从下面5行代码中可以看出，这时候会调用子View的dispatchTouchEvent，也就是在这个时候ViewGroup已经完成了事件分发的整个过程。
```
if (child == null) {
    handled = super.dispatchTouchEvent(event);
} else {
    handled = child.dispatchTouchEvent(event);
}
```
　　当子元素的dispatchTouchEvent返回为true的时候，也就是子View对事件处理成功。这时候便会通过addTouchTarget方法对mFirstTouchTarget进行赋值。
　　如果dispatchTouchEvent返回了false，或者说当前的ViewGroup没有子元素的话，那么这个时候便会调用如下代码。
```
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
}
```
　　在这里调用dispatchTransformedTouchEvent方法，并将child参数设为null。也就是执行了super.dispatchTouchEvent(event)方法。由于ViewGroup继承自View，所以这个时候又将事件交由自己处理。
　　到这里对于ViewGroup的事件分发已经讲完了，在这一路下来，不难发现对于dispatchTouchEvent有一个boolean类型返回值。对于这个返回值，当返回true的时候表示当前事件处理成功，若是返回false，一般来说是因为在事件处理onTouchEvent返回了false，这时候变会交由它的父控件进行处理，最终会交由Activity的onTouchEvent方法进行处理。
#**总结**
　　在这里从宏观上再看一下这个ViewGroup对事件的分发，当ViewGroup接收一个事件序列以后，首先会判断是否拦截该事件，若是拦截该事件，则将这个事件交由自己处理。若是不去拦截这一事件，便将该事件下发到子View当中。若果说ViewGroup没有子View，或者说子View对事件处理失败，则将该事件有交由该ViewGroup处理，若是该ViewGroup对事件依然处理失败，最终则会将事件交由Activity进行处理。
