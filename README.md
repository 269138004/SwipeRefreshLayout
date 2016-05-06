



## 简介
[官方文档](http://developer.android.com/intl/zh-cn/reference/android/support/v4/widget/SwipeRefreshLayout.html)

`SwipeRefreshLayout` 是一个下拉刷新控件，几乎可以包裹一个任何可以滑动的内容（ListView GridView ScrollView RecyclerView），可以自动识别垂直滑动手势。使用起来非常方便。

| | |
|:-:|:-:|
|![](http://img.blog.csdn.net/20150127120706062)|![](http://img.blog.csdn.net/20150127121649015)|

1. 将需要下拉刷新的空间包裹起来

```xml
<android.support.v4.widget.SwipeRefreshLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

</android.support.v4.widget.SwipeRefreshLayout>
```


2. 设置刷新动画的触发回调

```java

//设置下拉出现小圆圈是否是缩放出现，出现的位置，最大的下拉位置
mySwipeRefreshLayout.setProgressViewOffset(true, 50, 200);

//设置下拉圆圈的大小，两个值 LARGE， DEFAULT
mySwipeRefreshLayout.setSize(SwipeRefreshLayout.LARGE);

/*
 * 设置手势下拉刷新的监听
 */
mySwipeRefreshLayout.setOnRefreshListener(
    new SwipeRefreshLayout.OnRefreshListener() {
        @Override
        public void onRefresh() {
            // 刷新动画开始后回调到此方法
        }
    }
);

```

通过 `setRefreshing(false)` 和 `setRefreshing(true)` 来手动调用刷新的动画（以前版本好像设置了不生效，现在修复了）。
**注意!** `onRefresh` 的回调只有在手势下拉的情况下才会触发，通过 `setRefreshing` 只能调用刷新的动画是否显示。

## SwipeRefreshLayout 源码分析

本文基于 v4 版本 `23.2.0`

extends `ViewGroup` implements `NestedScrollingParent` `NestedScrollingChild`
```
java.lang.Object
   ↳	android.view.View
 	   ↳	android.view.ViewGroup
 	 	   ↳	android.support.v4.widget.SwipeRefreshLayout
```

其实就是一个自定义的 ViewGroup ，结合我们自己平时自定义 ViewGroup 的步骤：

1. 初始化变量
2. onMeasure
3. onLayout
4. 处理交互 `dispatchTouchEvent` `onInterceptTouchEvent` `onTouchEvent`
5. 暴露出公共接口供其他类调用

接下来就按照上面的步骤进行分析。

>此外，实现了两个接口 `NestedScrollingParent` `NestedScrollingChild`。关于 `NestedScroll`机制，可以去 google。
![UML](https://dn-coding-net-production-pp.qbox.me/a175b761-9513-42e8-bce1-a1e407388c2e.png)
简单总结一下，如果你的 View 实现 `NestedScrollingChild` 接口就可以支持嵌套滑动了（什么是嵌套滑动，就是滑动子View，父 View 也可以根据子 View 状态进行滑动，见 `CoordinatorLayout`）。同理，实现了 `NestedScrollingParent`接口就可以处理内部的子 View （实现了 `NestedScrollingChild` 的子 View）的滑动了。 `NestedScrollingChildHelper` 和 `NestedScrollingParentHelper` 是实现了对应的接口的类，可以帮助我们更简单的实现嵌套滑动（见上面的2篇文章）。

`SwipeRefreshLayout` 作为一个下拉刷新的动画，按理说只需要实现`NestedScrollingParent` 就行了，但是为了考虑到有其他可以滑动的组件嵌套 `SwipeRefreshLayout`（如 `CoordinatorLayout` ），所以也实现了`NestedScrollingChild`。Android 5.0 的大部分可以滑动的控件都支持了 `NestScrolling` 接口，最新的 Support V4 中也一样。


### 初始化变量


`SwipeRefreshLayout` 内部有 2 个 View，一个`圆圈（mCircleView）`，一个内部可滚动的` View（mTarget）`。除了 View，还包含一个 `OnRefreshListener` 接口，当刷新动画被触发时回调。

 
 ![图片](https://dn-coding-net-production-pp.qbox.me/8e02212d-b364-4df8-bfaa-47f3084f89e7.png) 


```java
/**
 * Constructor that is called when inflating SwipeRefreshLayout from XML.
 *
 * @param context
 * @param attrs
 */
public SwipeRefreshLayout(Context context, AttributeSet attrs) {
    super(context, attrs);

    // 系统默认的最小滑动距离
    mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();

    // 系统默认的动画时长
    mMediumAnimationDuration = getResources().getInteger(
            android.R.integer.config_mediumAnimTime);

    setWillNotDraw(false);
    mDecelerateInterpolator = new DecelerateInterpolator(DECELERATE_INTERPOLATION_FACTOR);

    // 获取 xml 中定义的属性
    final TypedArray a = context.obtainStyledAttributes(attrs, LAYOUT_ATTRS);
    setEnabled(a.getBoolean(0, true));
    a.recycle();

    // 刷新的圆圈的大小，单位转换成 sp
    final DisplayMetrics metrics = getResources().getDisplayMetrics();
    mCircleWidth = (int) (CIRCLE_DIAMETER * metrics.density);
    mCircleHeight = (int) (CIRCLE_DIAMETER * metrics.density);

    // 创建刷新动画的圆圈
    createProgressView();

    ViewCompat.setChildrenDrawingOrderEnabled(this, true);
    // the absolute offset has to take into account that the circle starts at an offset
    mSpinnerFinalOffset = DEFAULT_CIRCLE_TARGET * metrics.density;
    // 刷新动画的临界距离值
    mTotalDragDistance = mSpinnerFinalOffset;

    // 通过 NestedScrolling 机制来处理嵌套滚动
    mNestedScrollingParentHelper = new NestedScrollingParentHelper(this);
    mNestedScrollingChildHelper = new NestedScrollingChildHelper(this);
    setNestedScrollingEnabled(true);
}
```

// 创建刷新动画的圆圈
```java
private void createProgressView() {
    mCircleView = new CircleImageView(getContext(), CIRCLE_BG_LIGHT, CIRCLE_DIAMETER/2);
    mProgress = new MaterialProgressDrawable(getContext(), this);
    mProgress.setBackgroundColor(CIRCLE_BG_LIGHT);
    mCircleView.setImageDrawable(mProgress);
    mCircleView.setVisibility(View.GONE);
    addView(mCircleView);
}
```
可以看出使用背景圆圈是 v4 包里提供的 `CircleImageView` 控件，中间的是 `MaterialProgressDrawable` 进度条。

### onMeasure

```java
@Override
public void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    if (mTarget == null) {
        // 确定内部要滚动的View，如 RecycleView
        ensureTarget();
    }
    if (mTarget == null) {
        return;
    }

    // 测量子 View （mTarget）
    mTarget.measure(MeasureSpec.makeMeasureSpec(
            getMeasuredWidth() - getPaddingLeft() - getPaddingRight(),
            MeasureSpec.EXACTLY), MeasureSpec.makeMeasureSpec(
            getMeasuredHeight() - getPaddingTop() - getPaddingBottom(), MeasureSpec.EXACTLY));

    // 测量刷新的圆圈 mCircleView
    mCircleView.measure(MeasureSpec.makeMeasureSpec(mCircleWidth, MeasureSpec.EXACTLY),
            MeasureSpec.makeMeasureSpec(mCircleHeight, MeasureSpec.EXACTLY));

    if (!mUsingCustomStart && !mOriginalOffsetCalculated) {
        mOriginalOffsetCalculated = true;
        mCurrentTargetOffsetTop = mOriginalOffsetTop = -mCircleView.getMeasuredHeight();
    }

    // 计算 mCircleView 在 ViewGroup 中的索引
    mCircleViewIndex = -1;
    // Get the index of the circleview.
    for (int index = 0; index < getChildCount(); index++) {
        if (getChildAt(index) == mCircleView) {
            mCircleViewIndex = index;
            break;
        }
    }
}
```


```java
private void ensureTarget() {
    // Don't bother getting the parent height if the parent hasn't been laid out yet.
    if (mTarget == null) {
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (!child.equals(mCircleView)) {
                mTarget = child;
                break;
            }
        }
    }
}
```

找到除了 mCircleView 的第一个子 View ，也就是内部可以滚动的 View， 并且赋值为 mTarget


### onLayout

```java
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
  // 获取 SwipeRefreshLayout 的宽高
   final int width = getMeasuredWidth();
   final int height = getMeasuredHeight();
   if (getChildCount() == 0) {
       return;
   }
   if (mTarget == null) {
       ensureTarget();
   }
   if (mTarget == null) {
       return;
   }
   // 考虑到给控件设置 padding，去除 padding 的距离
   final View child = mTarget;
   final int childLeft = getPaddingLeft();
   final int childTop = getPaddingTop();
   final int childWidth = width - getPaddingLeft() - getPaddingRight();
   final int childHeight = height - getPaddingTop() - getPaddingBottom();
   // 设置 mTarget 的位置
   child.layout(childLeft, childTop, childLeft + childWidth, childTop + childHeight);
   int circleWidth = mCircleView.getMeasuredWidth();
   int circleHeight = mCircleView.getMeasuredHeight();
   // 根据 mCurrentTargetOffsetTop 变量的值来设置 mCircleView 的位置
   mCircleView.layout((width / 2 - circleWidth / 2), mCurrentTargetOffsetTop,
           (width / 2 + circleWidth / 2), mCurrentTargetOffsetTop + circleHeight);
}
```
 ![图片](https://dn-coding-net-production-pp.qbox.me/8df6d458-700b-4ec5-b731-c6b8c34cdddc.png) 

//mCurrentTargetOffsetTop = mOriginalOffsetTop = -mCircleView.getMeasuredHeight();
在 onLayout 中放置了 mCircleView 的位置，注意 顶部位置是 mCurrentTargetOffsetTop ，mCurrentTargetOffsetTop 初始距离是`-mCircleView.getMeasuredHeight()`，所以是在 SwipeRefreshLayout 外。


### 处理 Touch 事件

SwipeRefreshLayout 通过实现 `NestedScrollingParent` `NestedScrollingChild` 接口来分发触摸事件，如果不太了解的话，建议看一下相关的文档。

看一下 SwipeRefreshLayout 实现 NestedScrollingParent 的相关方法
```java
// NestedScrollingParent

// 子 View （NestedScrollingChild）开始滑动前回调此方法,返回 true 表示接收嵌套滑动，然后调用 onNestedScrollAccepted
// 具体可以看 NestedScrollingChildHelper 的源码
@Override
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    // 子 View 回调，判断是否开始嵌套滑动 ，
    return isEnabled() && !mReturningToStart && !mRefreshing
            && (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
}

@Override
 public void onNestedScrollAccepted(View child, View target, int axes) {
     // Reset the counter of how much leftover scroll needs to be consumed.
     mNestedScrollingParentHelper.onNestedScrollAccepted(child, target, axes);
     // Dispatch up to the nested parent
     startNestedScroll(axes & ViewCompat.SCROLL_AXIS_VERTICAL);
     mTotalUnconsumed = 0;
     mNestedScrollInProgress = true;
 }
```
SwipeRefreshLayout 只接受竖直方向（Y轴）的滑动，并且在刷新动画进行中不接受滑动。

```java
// NestedScrollingChild 在滑动的时候会触发， 看父类消耗了多少距离
//   * @param dx x 轴滑动的距离
//   * @param dy y 轴滑动的距离
//   * @param consumed 代表 父 View 消费的滑动距离
//
@Override
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    // If we are in the middle of consuming, a scroll, then we want to move the spinner back up
    // before allowing the list to scroll
    if (dy > 0 && mTotalUnconsumed > 0) {
        if (dy > mTotalUnconsumed) {
            consumed[1] = dy - (int) mTotalUnconsumed; // 消费的 y 轴的距离
            mTotalUnconsumed = 0;
        } else {
            mTotalUnconsumed -= dy;
            consumed[1] = dy;
        }
        // 出现动画圆圈，并向下移动
        moveSpinner(mTotalUnconsumed);
    }

    // If a client layout is using a custom start position for the circle
    // view, they mean to hide it again before scrolling the child view
    // If we get back to mTotalUnconsumed == 0 and there is more to go, hide
    // the circle so it isn't exposed if its blocking content is moved
    if (mUsingCustomStart && dy > 0 && mTotalUnconsumed == 0
            && Math.abs(dy - consumed[1]) > 0) {
        mCircleView.setVisibility(View.GONE);
    }

    // Now let our nested parent consume the leftovers
    final int[] parentConsumed = mParentScrollConsumed;
    if (dispatchNestedPreScroll(dx - consumed[0], dy - consumed[1], parentConsumed, null)) {
        consumed[0] += parentConsumed[0];
        consumed[1] += parentConsumed[1];
    }
}


// onStartNestedScroll 返回 true 才会调用此方法。此方法表示子View将滑动事件分发到父 View（SwipeRefreshLayout）
//  @param target The descendent view controlling the nested scroll
//  @param dxConsumed Horizontal scroll distance in pixels already consumed by target
//  @param dyConsumed Vertical scroll distance in pixels already consumed by target
//  @param dxUnconsumed Horizontal scroll distance in pixels not consumed by target
//  @param dyUnconsumed Vertical scroll distance in pixels not consumed by target
@Override
public void onNestedScroll(final View target, final int dxConsumed, final int dyConsumed,
        final int dxUnconsumed, final int dyUnconsumed) {
    // Dispatch up to the nested parent first
    dispatchNestedScroll(dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed,
            mParentOffsetInWindow);

    // This is a bit of a hack. Nested scrolling works from the bottom up, and as we are
    // sometimes between two nested scrolling views, we need a way to be able to know when any
    // nested scrolling parent has stopped handling events. We do that by using the
    // 'offset in window 'functionality to see if we have been moved from the event.
    // This is a decent indication of whether we should take over the event stream or not.
    final int dy = dyUnconsumed + mParentOffsetInWindow[1];
    if (dy < 0 && !canChildScrollUp()) {
        mTotalUnconsumed += Math.abs(dy);
        moveSpinner(mTotalUnconsumed);
    }
}
```



首先是 onInterceptTouchEvent，返回 true 表示拦截触摸事件。

```java


@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    ensureTarget();

    final int action = MotionEventCompat.getActionMasked(ev);

    // 手指按下时恢复状态
    if (mReturningToStart && action == MotionEvent.ACTION_DOWN) {
        mReturningToStart = false;
    }

    // 控件可用 || 刷新事件刚结束正在恢复初始状态时 || 子 View 可滚动 || 正在刷新 || 父 View 正在滑动
    if (!isEnabled() || mReturningToStart || canChildScrollUp()
            || mRefreshing || mNestedScrollInProgress) {
        // Fail fast if we're not in a state where a swipe is possible
        return false;
    }

    switch (action) {
        case MotionEvent.ACTION_DOWN:
            setTargetOffsetTopAndBottom(mOriginalOffsetTop - mCircleView.getTop(), true);
            mActivePointerId = MotionEventCompat.getPointerId(ev, 0);
            mIsBeingDragged = false;
            // 记录手指按下的位置，为了判断是否开始滑动
            final float initialDownY = getMotionEventY(ev, mActivePointerId);
            if (initialDownY == -1) {
                return false;
            }
            mInitialDownY = initialDownY;
            break;

        case MotionEvent.ACTION_MOVE:
            if (mActivePointerId == INVALID_POINTER) {
                Log.e(LOG_TAG, "Got ACTION_MOVE event but don't have an active pointer id.");
                return false;
            }

            final float y = getMotionEventY(ev, mActivePointerId);
            if (y == -1) {
                return false;
            }
            // 判断当拖动距离大于最小距离时设置 mIsBeingDragged = true;
            final float yDiff = y - mInitialDownY;
            if (yDiff > mTouchSlop && !mIsBeingDragged) {
                mInitialMotionY = mInitialDownY + mTouchSlop;
                mIsBeingDragged = true;
                // 正在拖动状态，更新圆圈的 progressbar 的 alpha 值
                mProgress.setAlpha(STARTING_PROGRESS_ALPHA);
            }
            break;

        case MotionEventCompat.ACTION_POINTER_UP:
            onSecondaryPointerUp(ev);
            break;

        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_CANCEL:
            mIsBeingDragged = false;
            mActivePointerId = INVALID_POINTER;
            break;
    }

    return mIsBeingDragged;
}

```
可以看到源码也就是进行简单处理，DOWN 的时候记录一下位置，MOVE 时判断移动的距离，返回值 mIsBeingDragged 为 true 时， 即 onInterceptTouchEvent 返回true，SwipeRefreshLayout 拦截触摸事件，不分发给 mTarget，然后把 MotionEvent 传给 onTouchEvent 方法。其中有一个判断子View的是否还可以滑动的方法 `canChildScrollUp`。

```java
/**
 * @return Whether it is possible for the child view of this layout to
 *         scroll up. Override this if the child view is a custom view.
 */
public boolean canChildScrollUp() {
    if (android.os.Build.VERSION.SDK_INT < 14) {
        // 判断 AbsListView 的子类 ListView 或者 GridView 等
        if (mTarget instanceof AbsListView) {
            final AbsListView absListView = (AbsListView) mTarget;
            return absListView.getChildCount() > 0
                    && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                            .getTop() < absListView.getPaddingTop());
        } else {
            return ViewCompat.canScrollVertically(mTarget, -1) || mTarget.getScrollY() > 0;
        }
    } else {
        return ViewCompat.canScrollVertically(mTarget, -1);
    }
}

```
可以通过上面的方法判断 View 是否可以继续向上滑动。

```java

@Override
public boolean onTouchEvent(MotionEvent ev) {
    final int action = MotionEventCompat.getActionMasked(ev);
    int pointerIndex = -1;

    if (mReturningToStart && action == MotionEvent.ACTION_DOWN) {
        mReturningToStart = false;
    }

    if (!isEnabled() || mReturningToStart || canChildScrollUp() || mNestedScrollInProgress) {
        // Fail fast if we're not in a state where a swipe is possible
        return false;
    }

    switch (action) {
        case MotionEvent.ACTION_DOWN:
            // 获取第一个按下的手指
            mActivePointerId = MotionEventCompat.getPointerId(ev, 0);
            mIsBeingDragged = false;
            break;

        case MotionEvent.ACTION_MOVE: {
          // 处理多指触控
            pointerIndex = MotionEventCompat.findPointerIndex(ev, mActivePointerId);
            if (pointerIndex < 0) {
                Log.e(LOG_TAG, "Got ACTION_MOVE event but have an invalid active pointer id.");
                return false;
            }

            final float y = MotionEventCompat.getY(ev, pointerIndex);
            final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
            if (mIsBeingDragged) {
                if (overscrollTop > 0) {
                    // 正在拖动状态，更新圆圈的位置
                    moveSpinner(overscrollTop);
                } else {
                    return false;
                }
            }
            break;
        }

        case MotionEventCompat.ACTION_POINTER_DOWN: {
            pointerIndex = MotionEventCompat.getActionIndex(ev);
            if (pointerIndex < 0) {
                Log.e(LOG_TAG, "Got ACTION_POINTER_DOWN event but have an invalid action index.");
                return false;
            }
            mActivePointerId = MotionEventCompat.getPointerId(ev, pointerIndex);
            break;
        }

        case MotionEventCompat.ACTION_POINTER_UP:
            onSecondaryPointerUp(ev);
            break;

        case MotionEvent.ACTION_UP: {
            pointerIndex = MotionEventCompat.findPointerIndex(ev, mActivePointerId);
            if (pointerIndex < 0) {
                Log.e(LOG_TAG, "Got ACTION_UP event but don't have an active pointer id.");
                return false;
            }

            final float y = MotionEventCompat.getY(ev, pointerIndex);
            final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
            mIsBeingDragged = false;
            // 手指松开，将圆圈移动到正确的位置
            finishSpinner(overscrollTop);
            mActivePointerId = INVALID_POINTER;
            return false;
        }
        case MotionEvent.ACTION_CANCEL:
            return false;
    }

    return true;
}

```


```java
// 手指下拉过程中触发的圆圈的变化过程，透明度变化，渐渐出现箭头，大小的变化
private void moveSpinner(float overscrollTop) {

    // 设置为有箭头的 progress
    mProgress.showArrow(true);

    // 进度转化成百分比
    float originalDragPercent = overscrollTop / mTotalDragDistance;

    // 避免百分比超过 100%
    float dragPercent = Math.min(1f, Math.abs(originalDragPercent));
    // 调整拖动百分比，造成视差效果
    float adjustedPercent = (float) Math.max(dragPercent - .4, 0) * 5 / 3;
    //
    float extraOS = Math.abs(overscrollTop) - mTotalDragDistance;

    // 这里mUsingCustomStart 为 true 代表用户自定义了起始出现的坐标
    float slingshotDist = mUsingCustomStart ? mSpinnerFinalOffset - mOriginalOffsetTop
            : mSpinnerFinalOffset;

    // 弹性系数
    float tensionSlingshotPercent = Math.max(0, Math.min(extraOS, slingshotDist * 2)
            / slingshotDist);
    float tensionPercent = (float) ((tensionSlingshotPercent / 4) - Math.pow(
            (tensionSlingshotPercent / 4), 2)) * 2f;
    float extraMove = (slingshotDist) * tensionPercent * 2;

    // 因为有弹性系数，不同的手指滑动距离不同于view的移动距离
    int targetY = mOriginalOffsetTop + (int) ((slingshotDist * dragPercent) + extraMove);

    // where 1.0f is a full circle
    if (mCircleView.getVisibility() != View.VISIBLE) {
        mCircleView.setVisibility(View.VISIBLE);
    }
    // 设置的是否有缩放
    if (!mScale) {
        ViewCompat.setScaleX(mCircleView, 1f);
        ViewCompat.setScaleY(mCircleView, 1f);
    }
    // 设置缩放进度
    if (mScale) {
        setAnimationProgress(Math.min(1f, overscrollTop / mTotalDragDistance));
    }
    // 移动距离未达到最大距离
    if (overscrollTop < mTotalDragDistance) {
        if (mProgress.getAlpha() > STARTING_PROGRESS_ALPHA
                && !isAnimationRunning(mAlphaStartAnimation)) {
            // Animate the alpha
            startProgressAlphaStartAnimation();
        }
    } else {
        if (mProgress.getAlpha() < MAX_ALPHA && !isAnimationRunning(mAlphaMaxAnimation)) {
            // Animate the alpha
            startProgressAlphaMaxAnimation();
        }
    }
    // 出现的进度，裁剪 mProgress
    float strokeStart = adjustedPercent * .8f;
    mProgress.setStartEndTrim(0f, Math.min(MAX_PROGRESS_ANGLE, strokeStart));
    mProgress.setArrowScale(Math.min(1f, adjustedPercent));

    // 旋转
    float rotation = (-0.25f + .4f * adjustedPercent + tensionPercent * 2) * .5f;
    mProgress.setProgressRotation(rotation);
    setTargetOffsetTopAndBottom(targetY - mCurrentTargetOffsetTop, true /* requires update */);
}
```

```java
private void finishSpinner(float overscrollTop) {
    if (overscrollTop > mTotalDragDistance) {
        //移动距离超过了刷新的临界值，触发刷新动画
        setRefreshing(true, true /* notify */);
    } else {
        // 取消刷新的圆圈，将圆圈移动到初始位置
        mRefreshing = false;
        mProgress.setStartEndTrim(0f, 0f);
        // ...省略代码

        // 移动到初始位置
        animateOffsetToStartPosition(mCurrentTargetOffsetTop, listener);
        // 设置没有箭头
        mProgress.showArrow(false)
    }
}

```

可以看到调用 setRefresh(true,true) 方法触发刷新动画并进行回调，但是这个方法是 private 的。前面提到我们自己调用 setRefresh(true) 只能产生动画，而不能回调刷新函数，那么我们就可以用反射调用 2 个参数的 setRefresh 函数。 或者手动调 setRefreshing(true)+ OnRefreshListener.onRefresh 方法。


### setRefresh

```java
/**
  * 改变刷新动画的的圆圈刷新状态。Notify the widget that refresh state has changed. Do not call this when
  * refresh is triggered by a swipe gesture.
  *
  * @param refreshing 是否显示刷新的圆圈
  */
 public void setRefreshing(boolean refreshing) {
     if (refreshing && mRefreshing != refreshing) {
         // scale and show
         mRefreshing = refreshing;
         int endTarget = 0;
         if (!mUsingCustomStart) {
             endTarget = (int) (mSpinnerFinalOffset + mOriginalOffsetTop);
         } else {
             endTarget = (int) mSpinnerFinalOffset;
         }
         setTargetOffsetTopAndBottom(endTarget - mCurrentTargetOffsetTop,
                 true /* requires update */);
         mNotify = false;
         startScaleUpAnimation(mRefreshListener);
     } else {
         setRefreshing(refreshing, false /* notify */);
     }
 }
```

startScaleUpAnimation 开启一个动画，然后在动画结束后回调 onRefresh 方法。

```java
private Animation.AnimationListener mRefreshListener = new Animation.AnimationListener() {
   @Override
   public void onAnimationStart(Animation animation) {
   }

   @Override
   public void onAnimationRepeat(Animation animation) {
   }

   @Override
   public void onAnimationEnd(Animation animation) {
       if (mRefreshing) {
           // Make sure the progress view is fully visible
           mProgress.setAlpha(MAX_ALPHA);
           mProgress.start();
           if (mNotify) {
               if (mListener != null) {
                   // 回调 listener 的 onRefresh 方法
                   mListener.onRefresh();
               }
           }
           mCurrentTargetOffsetTop = mCircleView.getTop();
       } else {
           reset();
       }
   }
};

```
## 总结

分析 SwipeRefreshLayout 的流程就是按照平时我们自定义 ViewGroup 的流程，但是其中也有好多需要我们借鉴的地方，如何使用 NestedScroll ，多点触控的处理，onMeasure 中减去了 padding，如何判断子 View 是否可滚动，如何确定 ViewGroup 中某一个 View 的索引。
一个好的下拉刷新框架不仅仅要兼容各种滑动的子控件，还要考虑自己要兼容 NestedScrolling 的情况，比如放到 CooCoordinatorLayout 的情况。
