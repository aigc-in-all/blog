title: View事件分发机制
date: 2015-12-13 22:04:37
tags: events
categories: Android
comments: true
---

## 点击事件的传递规则

所谓点击事件的事件分发，其实就是对MotionEvent事件的分发过程，即当一个MotionEvent产生了以后，系统需要把这个事件传递给一个具体的View，而这个传递的过程就是分发的过程。点击事件的分发过程由三个很重要的方法来共同完成：

**public boolean dispatchTouchEvent(MotionEvent ev)**

用来进行事件分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

**public boolean onInterceptTouchEvent(MotionEvent event)**

在上述方法内部调用，用来判断是否拦截这个事件，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

**public boolean onTouchEvent(MotionEvent event)**

在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

<!-- more -->

----------------

上述三个方法的关系可以用如下伪代码表示：

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
	boolean consume = false;
	if (onInterceptTouchEvent(ev)) {
		consume = onTouchEvent(ev);
	} else {
		consume = child.dispatchTouchEvent(ev);
	}

	return consume;
}
```

对于一个根ViewGroup来说，点击事件产生以后，首先会传递给它，这时它的dispatchTouchEvent就会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回True就表示它要拦截这个事件，接着事件就会交给这个ViewGroup处理，即它的onTouchEvent方法就会被调用；如果这个ViewGroup的onInterceptTouchEvent方法返回False就表示它不拦截这个事件，这时这个事件就会继续传递给它的子元素，接着子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件最终被处理。

## 事件分发源码解析

### Activity对点击事件的分发过程

Activity#dispatchTouchEvent

```java
/**
 * Called to process touch screen events.  You can override this to
 * intercept all touch screen events before they are dispatched to the
 * window.  Be sure to call this implementation for touch screen events
 * that should be handled normally.
 *
 * @param ev The touch screen event.
 *
 * @return boolean Return true if this event was consumed.
 */
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

当一个点击操作发生时，事件最先传递给当前的Activity，由Activity的dispatchTouchEvent来进行事件派发，事件首先交给Activity所属的Window，如果返回True，整个事件循环就结束了，返回False就意味着没人处理，所有View的onTouchEvent都返回了False，那么Activity的onTouchEvent就会被调用。

Window#superDispatchTouchent
```java
/**
 * Used by custom windows, such as Dialog, to pass the touch screen event
 * further down the view hierarchy. Application developers should
 * not need to implement or call this.
 *
 */
public abstract boolean superDispatchTouchEvent(MotionEvent event);
```

这是一个抽象方法，这里Window的实现类就是PhoneWindow，这点从Window的类注释中可以看出来：

```java
/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.policy.PhoneWindow, which you should instantiate when needing a
 * Window.  Eventually that class will be refactored and a factory method
 * added for creating Window instances without knowing about a particular
 * implementation.
 */
public abstract class Window {
...
}
```

PhoneWindow#superDispatchTouchEvent
```java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

PhoneWindow把事件直接传递给了DecorView。这个DecorView是什么呢？

```
PhoneWindow.java

// This is the top-level view of the window, containing the window decor.
private DecorView mDecor;

private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
...
}

@Override
public final View getDecorView() {
    if (mDecor == null) {
        installDecor();
    }
    return mDecor;
}
```

这个DecorView就是`getWindow().getDecorView()`返回的View。我们通过`setContentView`设置的View就是它的一个子View。目前事件传递了DecorView这里，由于DecorView继承自FrameLayout且是父View，所以最终事件传递给View。

### 顶级View对点击事件的分发过程

点击事件达到顶级View（一般是一个ViewGroup）以后，会调用ViewGroup的dispatchTouchEvent方法，然后的逻辑是这样的：如果顶级ViewGroup拦截事件即onInteerceptTouchEvent返回True，则事件由ViewGroup处理，这时如果ViewGroup的mOnTouchListener被设置，则onTouch被调用，否则onTouchEvent会被调用。也就是说，如果都提供的话，onTouch会屏蔽掉onTouchEvent。在onTouchEvent中，如果设置了mOnClickListener，则onClick被调用。如果顶级ViewGroup不拦截事件，则事件会传递给它所在的点击事件链上的子View，这时子View的dispatchTouchEvent会被调用。至此为止，事件已经从顶级View传递给了下一层View，接下来的传递过程和顶级View是一致的，如此循环，完成整个事件的分发。


ViewGroup#dispatchTouchEvent

```java

// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState(); // 重置FLAG_DISALLOW_INTERCEPT标记位
}

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

从上面的代码可以看出，ViewGroup在如下两种情况下会判断是否要拦截当前事件：事件类型为ACTION_DOWN或者mFirstTouchTarget != null。ACTION_DOWN事件好理解，那么mFirstTouchTarget != null是什么意思呢？这个从后面代码逻辑可以看出来，当事件由ViewGroup的子元素处理成功时，mFirstTouchTarget会被赋值并指向子元素，换种方式说，当ViewGroup不拦截事件并将事件交由子元素处理埋mFirstTouchTarget != null。反过来，一旦事件由当前ViewGroup拦截时，mFirstTouchTarget != null就不成立 。那么当ACTION_MOVE和ACTION_UP到来时，由于`actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null`条件为false，将导致ViewGroup的onInterceptTouchEvent不会再被调用，并且同一序列中的其它事件都会默认交给它处理。

FLAG_DISALLOW_INTERCEPT这个标记位是通过requestDisallowInterceptTouchEvent来设置的，一般用于子View中。FLAG_DISALLOW_INTERCEPT一旦设置后，ViewGroup将无法拦截除了ACTION_DOWN以外的其它点击事件。当面对ACTION_DOWN事件时，ViewGroup总会调用自己的onInterceptTouchEvent方法来询问自己是否要拦截事件。因此子View调用requestDisallowInterceptTouchEvent方法并不能影响ViewGroup对ACTION_DOWN的处理。

从ViewGroup的dispatchTouchEvent中可以看出，当ViewGroup决定拦截事件后，那么后续的点击事件会默认交给它处理并且 不再调用它的onInterceptTouchEvent方法。

接着看ViewGroup不拦截事件的时候，事件会向下分发交给它的子View进行处理：

```java
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

    // 判断子元素是否能够接收到点击事件
    // 1. 子元素是否在播放动画
    // 2. 点击事件的坐标是否落在子元素的区域内

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

    // dispatchTransformedTouchEvent实际上调用的就是子元素的dispatchTouchEvent方法
    // 如果子元素的dispatchTouchEvent返回True，那么mFirstTouchTarget会被赋值并且跳出循环
    // 如果返回Fase，就会继续循环下一个子元素
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

        // addTouchTarget里面会对mFirstTouchTarget赋值
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }

    // The accessibility focus didn't handle the event, so clear
    // the flag and do a normal dispatch to all children.
    ev.setTargetAccessibilityFocus(false);
}
```

mFirstTouchTarget会在addTouchTarget里面被赋值，mFirstTouchTarget是一种单链表结构，mFirstTouchTarget是否被赋值，将直接影响到ViewGroup对事件的拦截策略，如果mFirstTouchTarget为null，那么ViewGroup就默认拦截接下来的同一序列中的所有点击事件。

```java
/**
* Adds a touch target for specified child to the beginning of the list.
* Assumes the target child is not already present.
*/
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
target.next = mFirstTouchTarget;
mFirstTouchTarget = target;
return target;
}
```

如果遍历完所有子元素后都没有被合适地处理，包含两种情况：第一种是没有子元素，第二种是子元素处理了点击事件，但是在dispatchTouchEvent中返回了false，这一般是因为子元素在onTouchEvent中返回了false。这两种情况下，ViewGroup会自己处理点击事件：

```java
// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
```

```java
/**
 * Transforms a motion event into the coordinate space of a particular child view,
 * filters out irrelevant pointer ids, and overrides its action if necessary.
 * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
 */
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    if (child == null) {
        handled = super.dispatchTouchEvent(event);
    } else {
        handled = child.dispatchTouchEvent(event);
    }
}
```

上面第三个参数child为null，它会调用`super.dispatchTouchEvent(event)`，这里就转到了View的dispatchTouchEvent方法。

### View对点击事件的处理

View#dispatchTouchEvent

```java
/**
 * Pass the touch screen motion event down to the target view, or this
 * view if it is the target.
 *
 * @param event The motion event to be dispatched.
 * @return True if the event was handled by the view, false otherwise.
 */
public boolean dispatchTouchEvent(MotionEvent event) {

	...

    boolean result = false;

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

    ...

    return result;
}
```

因为View（这里不包含子ViewGroup）是一个单独的元素，它没有子元素可以向下传递事件，所以它只能自己处理事件。从上面源码可以看出View对点击事件的处理过程。首先判断有没有设置OnTouchListener，如果OnTouchListener中的onTouch方法返回True，那么onTouchEvent就不会被调用，可见OnTouchListener的优先级高于onTouchEvent.

View#onTouchEvent

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;

    // 当View处于不可用状态下照样会消耗点击事件
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
    }

    // 如果有设置代理，会调用TouchDelegate的onTouchEvent方法，这个onTouchEvent的机制看起来和OnTouchListener类似
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // take focus if we don't have it already and we should in
                    // touch mode.
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        focusTaken = requestFocus();
                    }

                    if (prepressed) {
                        // The button is being released before we actually
                        // showed it as pressed.  Make it show the pressed
                        // state now (before scheduling the click) to ensure
                        // the user sees it.
                        setPressed(true, x, y);
                   }

                    if (!mHasPerformedLongPress) {
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
                    ...
                }
                break;
                ...
        }

        return true;
    }

    return false;
}

```
从源码来看，只要View的CLICKABLE和LONG_CLICKABLE有一个为True，那么它就会消耗这个事件，即onTouchEvent返回True（DISABLE状态也会消耗事件，见代码中的注释）。当ACTION_UP事件发生时会触发performClick()方法，如果View有设置OnClickListener，那么performClick方法内部会调用它的onClick方法：

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

View的LONG_CLICKABLE属性默认为False，而CLICKABLE属性是否为False和具体的View有关，确切地说是可点击的View其CLICKABLE为True，不可点击的View其CLICKABLE为False，比如Button是可点击的，TextView是不可点击的。通过setClickable和setLongClickable可以分别改变View的CLICKABLE和LONG_CLICKABLE属性。setOnClickListener/setLongClickListener会改变这两个属性：

```java
public void setOnClickListener(OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}

public void setOnLongClickListener(OnLongClickListener l) {
    if (!isLongClickable()) {
        setLongClickable(true);
    }
    getListenerInfo().mOnLongClickListener = l;
}
```