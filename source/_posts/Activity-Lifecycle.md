title: Activity生命周期
date: 2014-03-12 13:50:28
categories: Android
tags: lifecycle
comments: true
---

## Activity生命周期全面分析

### 典型情况下的生命周期分析

在正常情况下，Activity会经历如下生命周期

1. onCreate：表示Activity正在被创建，这是生命周期的第一个方法。在这个方法中，我们可以做一些初始化工作，比如调用setContentView去加载界面布局资源、初始化Activity所需数据等。

2. onRestart：表示Activity正在重新启动。一般情况下，当当前Activity从不可见重新变为可见状态时，onRestart就会被调用。这种情形一般是用户行为所导致的，比如用户按Home键切换到桌面或者用户打开了一个新的Activity，这时当前的Activity就会暂停，也就是onPause和onStop被执行了，接着用户又回到这个Activity，就会出现这种情况。

3. onStart：表示Activity正在被启动，即将开始，这时Activity已经可见了，但是还没有出现在前台，还无法和用户交互。这个时候其实可以理解为Activity已经显示出来了，但是我们还看不到。

4. onResume：表示Activity已经可见了，并且出现在前台并开始活动。要注意这个和onStart的对比，onStart和onResume都表示Activity已经可见，但onStart的时候Activity还在后台，onResume的时候Activity才显示到前台。

5. onPause：表示Activity正在停止，正常情况下，紧接着onStop就会被调用。在特殊情况下，如果这个时候快速地再回到当前Activity，那么onResume会被调用。此时可以做一些存储数据、停止动画等工作，但是注意不能太耗时，因为这会影响到新Activity的显示，onPause必须先执行，新Activity的onResume才会执行。

6. onStop：表示Activity即将停止，可以做一些稍微重量级的回收工作，同样不能太耗时。

7. onDestroy：表示Activity即将被销毁，这是Activity生命周期中的最后一个回调方法，在这里，我们可以做一些回收工作和最终的资源释放。

<!-- more -->

正常情况下，Activity的常用生命周期就只有上面7个。

1. 针对一个特定的Activity，第一次启动，回调如下：onCreate->onStart->onResume。

2. 当用户打开新的Activity或者切换到桌面的时候，回调如下：onPause->onStop。如果新Activity采用了透明主题，那么当前Activity不会回调onStop。

3. 当用户再次回到原Activity时，回调如下：onRestart->onStart->onResume。

4. 当用户按Back键回退时，回调如下：onPause->onStop->onDestroy。

5. 当Activity被系统回收后再次打开，生命周期方法回调过程和1一样，注意只是生命周期方法一样，不代表所有过程都一样。

6. 从整个生命周期来说，onCreate和onDestroy是配对的，分别标识着Activity的创建和销毁，并且只可能有一次调用。从Activity是否可见来说，onStart和onStop是配对的，随着用户的操作或者设备屏幕的点亮和熄灭，这两个方法可能被调用多次；从Activity是否在前台来说，onResume和onPause是配对的，随着用户操作或者设备屏幕的点亮和熄灭，这两个方法可能会被多次调用。

问题：假设当前Activity为A，如果这时用户打开一个新的Activity B，那么B的onResume和A的onPause哪个先执行呢？

这个可以从Android的源码里得到解释。Activity的启动过程的源码相当复杂，涉及Instrumentation、ActiivtyThread和ActivityManagerService（简称AMS）。启动Acitivity的请求会由Instrumentation来处理，然后它通过Binder向AMS发请求，AMS内部维护着一个ActivityStack并负责栈内Activity的状态同步，AMS通过ActivityThread去同步Activity的状态从而完成生命周期方法的调用。在ActivityStack中的resumeTopActivityInnerLocked方法中有这么一段代码：

```java
// We need to start pausing the current activity so the top one
// can be resumed...
boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
if (mResumedActivity != null) {
    if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumedActivity);
    pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
}

```

可以看出，在新Activity启动之前，栈顶的Activity需要先onPause后，新Activity才能启动。最终在ActivityStackSupervisor中的realStartActivityLocked方法会调用如下代码：

```java
app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
        System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
        r.compat, r.launchedFromPackage, r.task.voiceInteractor, app.repProcState,
        r.icicle, r.persistentState, results, newIntents, !andResume,
        mService.isNextTransitionForward(), profilerInfo);

```

这个app.thread的类型是IApplicationThread，而IApplicationThread的具体实现是ActivityThread中的ApplicationThread。所以这段代码实际上调用了ActivityThread中，即ApplicatonThread的scheduleLaunchActivity方法，而scheduleLaunchActivity最终会完成Activity的onCreate、onStart、onReusme的调用过程。因此，可以得出结论，是旧Activity先onPause，然后新Activity再启动。

至于ApplicationThread的scheduleLaunchActivity方法为什么会完成新Activity的onCreate、onPause、onResume的调用过程，请看下面的代码。scheduleLaunchActivity最终会调用如下方法，而如下方法的确会完成onCreate、onPause、onResume的调用过程。


ActivityThread里面的ApplicationThread类:
```java

public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,
        String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state,
        PersistableBundle persistentState, List<ResultInfo> pendingResults,
        List<ReferrerIntent> pendingNewIntents, boolean notResumed, boolean isForward,
        ProfilerInfo profilerInfo) {

    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();

    r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.referrer = referrer;
    r.voiceInteractor = voiceInteractor;
    r.activityInfo = info;
    r.compatInfo = compatInfo;
    r.state = state;
    r.persistentState = persistentState;

    r.pendingResults = pendingResults;
    r.pendingIntents = pendingNewIntents;

    r.startsNotResumed = notResumed;
    r.isForward = isForward;

    r.profilerInfo = profilerInfo;

    updatePendingConfiguration(curConfig);

    sendMessage(H.LAUNCH_ACTIVITY, r);
}

```

最后sendMessage

```java
private void sendMessage(int what, Object obj) {
    sendMessage(what, obj, 0, 0, false);
}

private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) Slog.v(
        TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
        + ": " + arg1 + " / " + obj);
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}
```

这里mH实际上是一个Handler：
```java
private class H extends Handler {

	public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            ...
        }
    }
}

```

所以这里实际上是调用handleLaunchActivity方法：

```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
	// If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }

    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);

    if (localLOGV) Slog.v(
        TAG, "Handling launch of " + r);

    // Initialize before creating the activity
    WindowManagerGlobal.initialize();

    // 这里新Activity被创建出来，其onCreate和onStart会被调用
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        Bundle oldState = r.state;
        // 这里新Activity的onResume会被调用
        handleResumeActivity(r.token, false, r.isForward,
                !r.activity.mFinished && !r.startsNotResumed);
                ...
    }
}
```

从上面的分析可以看出，当前启动一个Activity的时候，旧Activity的onPause会先调用，然后才会启动新的Activity。


### 异常情况下的生命周期分析

Activity除了受用户操作所导致的正常的生命周期方法调用，还有一些异常情况，比如当资源相关的系统配置发生改变以及系统内存不足时，Activity就可能被杀死，下面具体分析这两种情况：

#### 资源相关的系统配置发生改变导致Activity被杀死并重新创建

理解这个问题，我们首先要对系统的资源加载机制有一定了解。拿最简单的图片来说，当我们把一张图片放在drawable目录后，就可以通过Resource去获取这张图片。同时为了兼容不同的设备，我们可能还需要在其他一些目录放置不同的图片，比如drawable-mdpi、drawable-hdpi、drawable-land等。这样，当应用程序启动时，系统就会根据当前情况去加载合适的Resource资源，比如说横屏手机和竖屏手机会拿到两张不同的图片（设定了landscape或portrait状态下的图片）。比如说当前Actiivty处于竖屏状态，如果突然旋转屏幕，由于系统配置发生了改变，在默认情况下，Activity会被销毁并且重新创建，当然我们也可以阻止系统重新创建我们的Activity。

在默认情况下，如果我产的Activity不做特殊处理，那么当系统配置发生改变后，Activity就会并重新创建。

当系统配置发生改变后，Activity会被销毁，其onPause、onStop、onDestroy均会被调用，同时由于Actiivty是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前Activity的状态。这个方法的调用时机是在onStop之前，它和onPause没有既定的时序关系，它既可能在onPause之前调用，也可以之后调用。需要强调的一点，这个方法只会出现在Activity被异常终止的情况下，正常情况下系统不会回调这个方法。当Activity被重新创建后，系统会调用onRestoreInstanceState，并且把Activity销毁时onSaveInstanceState方法所保存的Bundle对象作为参数同时传递给onRestoreInstanceState和onCreate方法。因此我们可以通过onRestoreInstanceState和onCreate来判断Activity是否被重建了，如果被重建了，那么我们就可以取出数据并恢复，从时序上来说，onRestoreInstanceState的调用时机在onStart之后。

同时，我们要知道，在onSaveInstanceState和onRestoreInstanceStae方法中，系统自动为我们做了一定的恢复工作。当Activity在异常情况下需要重新创建时，系统会默认为我们保存当前Activity的视图结构，并且在Activity重启后为我们恢复这些数据，比如EditText中的用户输入数据、ListView滚动的位置等，这些View相关的状态系统都能够默认为我们恢复。具体针对某一个特定的View系统能为我们恢复哪些数据，我们可以查看View的源码。和Activity一样，每个View都有onSaveInstanceState和onRestoreInstanceState方法，看一下它们的具体实现，就能知道系统能够自动为每个View恢复哪些数据。

关于保存和恢复View层次结构，系统的工作流程是这样的：首先Activity被异常终止时，Activity会调用onSaveInstanceState去保存数据，然后Activity会委托Window去保存数据，接着Window再委托它上面的顶级容器去保存数据。顶层容器是一个ViewGroup，一般来说它很可能是DecorView。最后顶层容器再去一一通知它的子元素来保存数据，这样整个数据保存过程就完成了。可以发现，这是一种典型的委托思想，上层委托下层、父容器委托子元素去处理一件事情，这种思想在Android中有很多应用，比如View的绘制过程、事件分发等都是采用类似的思想。至于数据恢复过程也是类似的，这里就不再介绍了。

Activity#onSaveInstanceState:
```java
protected void onSaveInstanceState(Bundle outState) {
    outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());
    Parcelable p = mFragments.saveAllState();
    if (p != null) {
        outState.putParcelable(FRAGMENTS_TAG, p);
    }
    getApplication().dispatchActivitySaveInstanceState(this, outState);
}
```

这里调用mWindow.saveHierarchyState()方法：mWindow是抽象类Window里面一个抽象方法，这里的实际类型是PhoneWindow：

```java
@Override
public Bundle saveHierarchyState() {
    Bundle outState = new Bundle();
    if (mContentParent == null) {
        return outState;
    }

    SparseArray<Parcelable> states = new SparseArray<Parcelable>();
    // mContentParent是一个ViewGroup类型，一般情况下是DecorView
    mContentParent.saveHierarchyState(states);
    outState.putSparseParcelableArray(VIEWS_TAG, states);
    ...
    return outState;
}

```

View：
```java
public void saveHierarchyState(SparseArray<Parcelable> container) {
        dispatchSaveInstanceState(container);
}

/**
 * Called by {@link #saveHierarchyState(android.util.SparseArray)} to store the state for
 * this view and its children. May be overridden to modify how freezing happens to a
 * view's children; for example, some views may want to not store state for their children.
 *
 * @param container The SparseArray in which to save the view's state.
 *
 * @see #dispatchRestoreInstanceState(android.util.SparseArray)
 * @see #saveHierarchyState(android.util.SparseArray)
 * @see #onSaveInstanceState()
 */
protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
    if (mID != NO_ID && (mViewFlags & SAVE_DISABLED_MASK) == 0) {
        mPrivateFlags &= ~PFLAG_SAVE_STATE_CALLED;
        Parcelable state = onSaveInstanceState();
        if ((mPrivateFlags & PFLAG_SAVE_STATE_CALLED) == 0) {
            throw new IllegalStateException(
                    "Derived class did not call super.onSaveInstanceState()");
        }
        if (state != null) {
            // Log.i("View", "Freezing #" + Integer.toHexString(mID)
            // + ": " + state);
            container.put(mID, state);
        }
    }
}
```

dispatchSaveInstanceState方法被ViewGroup override了：

ViewGroup#dispatchSaveInstanceState：
```java
@Override
protected void dispatchSaveInstanceState(SparseArray<Parcelable> container) {
    super.dispatchSaveInstanceState(container);
    final int count = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < count; i++) {
        View c = children[i];
        if ((c.mViewFlags & PARENT_SAVE_DISABLED_MASK) != PARENT_SAVE_DISABLED) {
            c.dispatchSaveInstanceState(container);
        }
    }
}
```

到此，这里来看一下TextView到底保存了哪些数据。

TextView#onSaveInstanceState：
```java
@Override
public Parcelable onSaveInstanceState() {
    Parcelable superState = super.onSaveInstanceState();

    // Save state if we are forced to
    boolean save = mFreezesText;
    int start = 0;
    int end = 0;

    if (mText != null) {
        start = getSelectionStart();
        end = getSelectionEnd();
        if (start >= 0 || end >= 0) {
            // Or save state if there is a selection
            save = true;
        }
    }

    if (save) {
        SavedState ss = new SavedState(superState);
        // XXX Should also save the current scroll position!
        ss.selStart = start;
        ss.selEnd = end;

        if (mText instanceof Spanned) {
            Spannable sp = new SpannableStringBuilder(mText);

            if (mEditor != null) {
                removeMisspelledSpans(sp);
                sp.removeSpan(mEditor.mSuggestionRangeSpan);
            }

            ss.text = sp;
        } else {
            ss.text = mText.toString();
        }

        if (isFocused() && start >= 0 && end >= 0) {
            ss.frozenWithFocus = true;
        }

        ss.error = getError();

        return ss;
    }

    return superState;
}
```

从源码中可以看出，TextView保存了自己的文本选中状态和文本内容，并且通过查看其onRestoreInstanceState方法的源码，可以发现它的确恢复了这些数据，具体源码就不贴出来了。

#### 资源内存不足导致低优先级的Activity被杀死

Activity按照优先级从高到低，可以分为如下三种：

1. 前台Activity——正在和用户交互的Activity的，优先级最高。
2. 可见但非前台Activity——比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法和用户直接交互。
3.后台Activity——已经被暂停的Activity，比如执行了onStop，优先级最低。

当系统内存不足时，系统就会按照上述优先级去回收Activity，并在后续通过onSaveInstanceState和onRestoreInstanceState来存储和恢复数据。迢迢晒个进程中没有四大组件在执行，那么这个进程将很快被系统杀死，因此一些后台工作不适合脱离四大组件而独自支行在后台中，这样进程很容易被杀死。比较好的方法是将后台工作放入Service中从而保证进程有一定的优先级，这样就不会轻易地被系统杀死。

上面分析了系统的数据存储和恢复机制，我们知道，当系统配置发生改变后，Activity会被重新创建，那么有没有办法不重新创建呢？系统配置中有很多内容，如果当某项内容发生改变后，我们不想系统重新创建Activity可以给Activity指定configChanges属性。比如不想让Activity在屏幕旋转的时候重新创建，就可以给configChanges属性添加orientation这个值：

```xml
android:configChanges="orientation"
```

如果想指定多个值，可以用`|`连接起来，比如`android:configChagnes="orientation|keyboardHidden"`

下面列出一些常见的属性：

 属性 | 含义
------|------
locale | 设备的本地位置发生了改变，一般指切换了系统语言
keyboard | 键盘类型发生了改变，比如用户使用了外接键盘 
keyboardHidden | 键盘的可访问性发生了改变，比如用户调出了键盘 
fontScale | 系统字体缩放比例发生了改变，比如用户选择了一个新的字号
uiMode | 用户界面模式发生了改变，比如是否开户了夜间模式（API8新添加）
orientation | 屏幕方向发生了改变，这个是最常用的，比如旋转了手机屏幕
layoutDirection | 当布局方向发生变化，这个属性用的比较少，正常情况下无须修改布局的layoutDirection属性（API17新添加）