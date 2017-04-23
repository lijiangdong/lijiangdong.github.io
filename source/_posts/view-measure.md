---
title:   Android视图的绘制流程(上)——View的测量
date: 2016-06-10 22:34
category: [Android]
tags: [view]
comments: true
---

　　View的绘制流程可以分为三大步，它们分别是measure，layout和draw过程。measure表示View的测量过程，用于测量View的宽度和高度；layout用于确定View在父容器的位置；draw则是负责将View绘制到屏幕中。下面主要来看一下View的Measure过程。<!--more-->
# **测量过程**
　　View的绘制流程是从ViewRoot的performTraversals方法开始的，ViewRoot对应ViewRootImpl类。ViewRoot在performTraversals中会调用performMeasure方法来进行对根View的测量过程。而在performMeasure方法中又会调用View的measure方法。对于View的measure方法它是一个final类型，也就是说这个measure方法不能被子类重写。但是在measure方法中调用了onMeasure方法。所以View的子类可以重写onMeasure方法来实现各自的Measure过程。在这里也就是主要对onMeasure方法进行分析。
## **MeasureSpec**
　　MeasureSpec是View类中的一个静态内部类。一个MeasureSpec封装了父布局传递给子布局的布局要求。每个MeasureSpec都代表着一个高度或宽度的要求。每个MesureSpec都是由specSize和specMode组成，它代表着一个32位的int值，其中高2位代表specSize，低30位代表specMode。
　　MeasureSpec的测量模式有三种，下面介绍一下这三种测量模式：

 - **UNSPECIFIED**
父容器对子View没有任何的限制，子View可以是任何的大小。
 - **EXACTLY**
父容器为子View大小指定一个具体值，View的最终大小就是specSize。对应View属性match_parent和具体值。
 - **AT_MOST**
子View的大小最大只能是specSize，也就是所子View的大小不能超过specSize。对应View属性的wrap_content.

　　在MeasureSpec中可以通过specSize和specMode并使用makeMeasureSpec方法来创建一个MeasureSpec，还可以通过getMode和getSize来获取MeasureSpec的specMode和specSize。
## **View的测量过程**
　　在上面已经说到，View的Measure过程是由measure方法来完成的，而measure方法通过调用onMeasure方法来完成View的Measure过程。那么就来看一下onMeasure方法。
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
　　在View的onMeasure方法中只是调用了setMeasuredDimension方法，setMeasuredDimension方法的作用就是设置View的高和宽的测量值。对于View测量后宽和高的值是通过getDefaultSize方法来获取的。下面就来一下这个getDefaultSize方法。
```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```
　　对于MeasureSpec的AT_MOST和EXACTLY模式下，直接返回的就是MeasureSpec的specSize，也就是说这个specSize就是View测量后的大小。而对于在UNSPECIFIED模式下，View的测量值则为getDefaultSize方法中的第一个参数size。这个size所对应的宽和高是通过getSuggestedMinimumWidth和getSuggestedMinimumHeight两个方法获取的。下面就来看一下这两个方法。
```java
protected int getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());

}

protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```
　　在这里可以看到对于View宽和高的取值是根据View是否存在背景进行设置的。在这里以View的宽度来进行说明。若是View没有背景则是View的宽度mMinWidth。对于mMinWidth值得设置可以在XML布局文件中设置minWidth属性，它的默认值为0。也可以通过调用View的setMinimumWidth()方法其赋值。若是View存在背景的话，则取View本身最小宽度mMinWidth和View背景的最小宽度它们中的最大值。
## **ViewGroup的测量过程**
　　对于ViewGroup的Measure过程，ViewGroup处理Measure自己本身的大小，还需要遍历子View，并调用它们的measure方法，然后各个子元素再去递归执行Measure过程。在ViewGroup中并没有重写onMeasure方法，因为ViewGroup它是一个抽象类，对于不同的具体ViewGroup它的onMeasure方法中所实现的过程不一样。但是在ViewGroup中提供了一个measureChildren方法，对子View进行测量。下面就来看一下这个measureChildren方法。
```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
```
　　在这里获取ViewGroup中所有的子View。然后遍历ViewGroup中子View并调用measureChild方法来完成对子View的测量。下面看一下measureChild方法。
```java
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
　　在这段代码中通过getChildMeasureSpec方法获取子View宽和高的MeasureSpec。然后调用子View的measure方法开始对View进行测量。下面就来看一下是如何通过getChildMeasureSpec方法来获取View的MeasureSpec的。
```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
　　在这段代码对于MeasureSpec的获取主要是根据父容器的MeasureSpec和View本身的LayoutParams。下面通过一张表格来看一下它们之间的对应关系。

| layoutParams\specMode | 	EXACTLY | AT_MOST | UNSPECIFIED |
| -- | -- | -- | -- |
| dp/px|EXACTLY&childSize|EXACTLY&childSize|	EXACTLY&childSize |
| MATCH_PARENT|EXACTLY&parentSize| AT_MOST&parentSize | UNSPECIFIED&0 |
| WRAP_CONTENT|AT_MOST&parentSize| AT_MOST&parentSize | UNSPECIFIED&0 |

　　到这里通过getChildMeasureSpec方法获取到子View的MeasureSpec以后，便调用View的Measure方法，开始对View进行测量。
　　正如刚才说的那样对于ViewGroup它是一个抽象类，并没有重写View的onMeasure方法。但是到具体的ViewGroup时，例如FrameLayout，LinearLayout，RelativeLayout等，它们通过重写onMeasure方法来来完成自身以及子View的Measure过程。下面以FrameLayout为例，看一下的Measure过程。在FrameLayout中，它的Measure过程也算是比较简单，下面就来看一下FrameLayout中的onMeasure方法。
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);
                }
            }
        }
    }

    // Account for padding too
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // Check against our minimum height and width
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));

    count = mMatchParentChildren.size();
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            final View child = mMatchParentChildren.get(i);
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

            final int childWidthMeasureSpec;
            if (lp.width == LayoutParams.MATCH_PARENT) {
                final int width = Math.max(0, getMeasuredWidth()
                        - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                        - lp.leftMargin - lp.rightMargin);
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        width, MeasureSpec.EXACTLY);
            } else {
                childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                        lp.leftMargin + lp.rightMargin,
                        lp.width);
            }

            final int childHeightMeasureSpec;
            if (lp.height == LayoutParams.MATCH_PARENT) {
                final int height = Math.max(0, getMeasuredHeight()
                        - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                        - lp.topMargin - lp.bottomMargin);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        height, MeasureSpec.EXACTLY);
            } else {
                childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                        getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                        lp.topMargin + lp.bottomMargin,
                        lp.height);
            }

            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```
　　在这部分代码中逻辑也很简单，主要完成了两件事。首先FrameLayout完成自身的测量过程，然后在遍历子View，执行View的measure方法，完成View的Measure过程。在这里代码比较简单就不在进行详细描述。
# **总结**
　　最后对View和ViewGroup的Measure过程做一下总结。对于View，它的Measure很简单，在获取到View的高和宽的测量值之后，便为其设置高和宽。而对于ViewGroup来说，除了完成自身的Measure过程以外，还需要遍历子View，完成子View的测量过程。