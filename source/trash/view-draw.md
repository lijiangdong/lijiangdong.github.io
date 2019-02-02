---
title:  Android视图的绘制流程(下)——View的Layout与Draw过程
date: 2016-06-11 13:26
category: [Android]
tags: [view]
comments: true
---

　　在上篇文章中[Android视图的绘制流程(上)——View的测量](http://blog.csdn.net/ljd2038/article/details/51629053)对View的Measure过程进行了详细的说明。对于在View的绘制的整个过程中，在对View的大小进行测量以后，便开始确定View的位置并且将其绘制到屏幕上。也就是View的Layout与Draw过程。那么就来看一下是如何实现这两个过程的。<!--more-->
# **View的Layout过程**
　　上文提到View的绘制流程是从ViewRoot的performTraversals方法开始，那么在View完成测量以后，在performTraversals方法中对performLayout进行调用。在performLayout中可以找到下面这行代码。
```java
host.layout(0, 0, host.getMeasuredWidth(),host.getMeasuredHeight())
```
　　上面这行代码中的host指的就是DecorView，对于这个DecorView我们都知道它是一个继承自FrameLayout的ViewGroup。这个layout方法也就是ViewGroup中的layout方法。下面就来看一下ViewGroup中的这个layout方法。
```java
@Override
public final void layout(int l, int t, int r, int b) {
    if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
        if (mTransition != null) {
            mTransition.layoutChange(this);
        }
        super.layout(l, t, r, b);
    } else {
        // record the fact that we noop'd it; request layout when transition finishes
        mLayoutCalledWhileSuppressed = true;
    }
}
```
　　从ViewGroup的layout方法我们可以看出它是一个final类型的，也就是说在ViewGroup中的layout方法是不能被子类重写的。ViewGroup中的layout方法中又调用父类的layout方法，也就是View的layout方法。下面就来看一下View的layout方法。
```java
@SuppressWarnings({"unchecked"})
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }

    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;

    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        ......
    }

    ......
}
```
　　对于setOpticalFrame实质上也是调用setFrame方法，而setFrame的作用是将View的位置分别保存到mLeft，mTop，mBottom，mRight变量当中。之后在判断是否需要重新布局，如果需要重新布局的话，便调用onLayout方法。
　　其实在View的Layout过程当中，在View的layout方法是确定View的自身位置，而在View的onLayout方法中则是确定View子元素的位置。所以在这可以看到对于View的onLayout是一个空方法，没有完成任何事情。
```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```
　　而ViewGroup的onLayout方法则是一个抽象方法，通过具体的ViewGroup实现类来完成对子元素的Layout过程。
```java
@Override
protected abstract void onLayout(boolean changed, int l, int t, int r, int b);
```
　　下面依然通过一个具体的ViewGroup，来看一下FrameLayout的onLayout方法实现过程，对于FrameLayout的onLayout方法的实现是非常简单的，所以就以FrameLayout为例进行说明。下面来看一下FrameLayout的onLayout方法。
```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false /* no force left gravity */);
}
```
　　在FrameLayout的onLayout方法中只是调用了layoutChildren方法，从这个方法名便可以看出它的功能就是为FrameLayout的子元素进行布局。下面就来看一下这个layoutChildren方法。
```java
void layoutChildren(int left, int top, int right, int bottom,
                              boolean forceLeftGravity) {
    final int count = getChildCount();

    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();

    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();

            int childLeft;
            int childTop;

            int gravity = lp.gravity;
            if (gravity == -1) {
                gravity = DEFAULT_CHILD_GRAVITY;
            }

            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                    lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }

            switch (verticalGravity) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                    lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }

            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```
　　对于这段代码的逻辑也很简单。通过遍历FrameLayout内所有的子元素，然后获取到View测量后的宽和高，在根据子View的Gravity属性来决定子View在父控件中四个顶点的位置。最后调用子View的layout方法来完成View的整个测量过程。
# **View的Draw过程**
　　在通过ViewRoot的performTraversals方法完成对View树的整个布局以后，下面便开始将View绘制到手机屏幕上。对于View的Draw过程在ViewRoot的performTraversals方法中通过调用performDraw方法来完成的。在performDraw方法中最终会通过创建一个Canvas对象，并调用View的draw方法，来完成View的绘制。
```java
@CallSuper
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */

    // Step 1, draw the background, if needed
    int saveCount;

    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // we're done...
        return;
    }

   ......
}
```
　　在这里从宏观上来对Draw过程进行一下分析。在注释中可以看出对于View的绘制过程分为六步进行的。其中第二步和第五步一般很少用到，可以忽略。剩下几步则为：

 1. 绘制背景 drawBackground(canvas)
 2. 绘制自身 onDraw(canvas)
 3. 绘制 children dispatchDraw(canvas)
 4. 绘制装饰 onDrawForeground(canvas)

　　对于子View的绘制传递是通过dispatchDraw来进行的，在View中的dispatchDraw方法是由ViewGroup来实现的，并且遍历调用所有子元素的draw方法，完成整个View树的绘制过程。
# **总结**
　　对于View的绘制流程，总共分为三大步。分别是View的测量，布局与绘制。首先通过ViewRoot，对View树根节点进行操作。依次向下遍历，完成它们的Measure，Layout，Draw过程。从而使View展现在手机屏幕上。