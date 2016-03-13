title: Android中点击事件的来源
date: 2016-03-13 15:52:02
tags: events
categories: Android
comments: true
---

> 原文 http://blog.csdn.net/singwhatiwanna/article/details/50775201#rd
> 作者 任玉刚

首先看Activity的实现，如下，Activity实现了一个特殊的接口：Window.Callback

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback {
...
}
```

Window.Callback:
```java
public abstract class Window {

	/**
     * API from a Window back to its caller.  This allows the client to
     * intercept key dispatching, panels and menus, etc.
     */
    public interface Callback {
        /**
         * Called to process key events.  At the very least your
         * implementation must call
         * {@link android.view.Window#superDispatchKeyEvent} to do the
         * standard key processing.
         *
         * @param event The key event.
         *
         * @return boolean Return true if this event was consumed.
         */
        public boolean dispatchKeyEvent(KeyEvent event);

        /**
         * Called to process a key shortcut event.
         * At the very least your implementation must call
         * {@link android.view.Window#superDispatchKeyShortcutEvent} to do the
         * standard key shortcut processing.
         *
         * @param event The key shortcut event.
         * @return True if this event was consumed.
         */
        public boolean dispatchKeyShortcutEvent(KeyEvent event);

        /**
         * Called to process touch screen events.  At the very least your
         * implementation must call
         * {@link android.view.Window#superDispatchTouchEvent} to do the
         * standard touch screen processing.
         *
         * @param event The touch screen event.
         *
         * @return boolean Return true if this event was consumed.
         */
        public boolean dispatchTouchEvent(MotionEvent event);

        ...
    }
}
```

然后我们似乎看出了一些端倪，难道这个接口和点击事件的传递有关？嗯，你猜对了。在艺术探索这本书中，并没有描述事件是如何传递给Activity的，但是这里我们可以猜测，如果外界想要传递点击事件给Activity，那么它就必须持有Activity的引用，这没错，在Activity的attach方法中，有如下一段：

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context);

    mFragments.attachActivity(this, mContainer, null);

    mWindow = PolicyManager.makeNewWindow(this);
    mWindow.setCallback(this);
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    ...
}        
```

<!-- more -->

显然，mWindow持有了Activity的引用，它通过setCallback方法来持有Activity，因此，事件是从Window传递给了Activity。也许你会说：“我不信，这理由不充分！”，没关系，我们继续分析。

我们知道，Activity启动以后，在它的onResume以后，DecorView才开始attach给WindowManager从而显示出来。（什么？你不知道？回去看艺术探索第8章），请看Activity的makeVisible方法，代码如下：

```java
    void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```

```java
    public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
       ...

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ...

            root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }

        // do this last because it fires off messages to start doing things
        try {
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            // BadTokenException or InvalidDisplayException, clean up.
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }
```

可以看到，ViewRootImpl创建了，在ViewRootImpl的setView方法（此方法运行在UI线程）中，会通过跨进程的方式向WMS（WindowManagerService）发起一个调用，从而将DecorView最终添加到Window上，在这个过程中，ViewRootImpl、DecorView和WMS会彼此向关联，同时会创建InputChannel、InputQueue和WindowInputEventReceiver来接受点击事件的消息。

好了，言归正传，下面来说，点击事件到底怎么传递给Activity的。首先要明白，点击事件是由用户的触摸行为所产生的，因此它必须要通过硬件来捕获，然后点击事件会交给WMS来处理。

在ViewRootImpl中，有一个方法，叫做dispatchInputEvent，如下：

```java
    public void dispatchInputEvent(InputEvent event, InputEventReceiver receiver) {
        SomeArgs args = SomeArgs.obtain();
        args.arg1 = event;
        args.arg2 = receiver;
        Message msg = mHandler.obtainMessage(MSG_DISPATCH_INPUT_EVENT, args);
        msg.setAsynchronous(true);
        mHandler.sendMessage(msg);
    }
```

那么什么是InputEvent呢？InputEvent有2个子类：KeyEvent和MotionEvent，其中KeyEvent表示键盘事件，而MotionEvent表示点击事件。在上面的代码中，mHandler是一个在UI线程创建的Handder，所以它会把执行逻辑切换到UI线程中。

```java
final ViewRootHandler mHandler = new ViewRootHandler();
```

这个消息的处理如下：

```java
case MSG_DISPATCH_INPUT_EVENT: {
    SomeArgs args = (SomeArgs)msg.obj;
    InputEvent event = (InputEvent)args.arg1;
    InputEventReceiver receiver = (InputEventReceiver)args.arg2;
    enqueueInputEvent(event, receiver, 0, true);
    args.recycle();
} break;
```

除此之外，WindowInputEventReceiver也可以来接收点击事件的消息，同样它也有一个dispatchInputEvent方法，注意，WindowInputEventReceiver中的Looper为UI线程的Looper。

```java
final class WindowInputEventReceiver extends InputEventReceiver {
    public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
        super(inputChannel, looper);
    }

    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchInputEvent(int seq, InputEvent event) {
        mSeqMap.put(event.getSequenceNumber(), seq);
        onInputEvent(event);
    }

    @Override
    public void onInputEvent(InputEvent event) {
        enqueueInputEvent(event, this, 0, true);
    }
}

WindowInputEventReceiver mInputEventReceiver;

...

mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());


```

可以发现，不管是ViewRootImpl的dispatchInputEvent方法，还是WindowInputEventReceiver的dispatchInputEvent方法，它们本质上都是调用deliverInputEvent方法来处理点击事件的消息，如下：

```java
private void deliverInputEvent(QueuedInputEvent q) {
    Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
            q.mEvent.getSequenceNumber());
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
    }

    InputStage stage;
    if (q.shouldSendToSynthesizer()) {
        stage = mSyntheticInputStage;
    } else {
        stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
    }

    if (stage != null) {
        stage.deliver(q);
    } else {
        finishInputEvent(q);
    }
}
```

在ViewRootImpl中，有一系列类似于InputStage（输入事件舞台）的概念，每种InputStage可以处理一定的事件类型，比如AsyncInputStage、ViewPreImeInputStage、ViewPostImeInputStage等。当一个InputEvent到来时，ViewRootImpl会寻找合适它的InputStage来处理。对于点击事件来说，ViewPostImeInputStage可以处理它，ViewPostImeInputStage中，有一个processPointerEvent方法，如下，它会调用mView的dispatchPointerEvent方法，注意，这里的mView其实就是DecorView。

```java
        private int processPointerEvent(QueuedInputEvent q) {
            final MotionEvent event = (MotionEvent)q.mEvent;

            mAttachInfo.mUnbufferedDispatchRequested = false;
            boolean handled = mView.dispatchPointerEvent(event);
            if (mAttachInfo.mUnbufferedDispatchRequested && !mUnbufferedInputDispatch) {
                mUnbufferedInputDispatch = true;
                if (mConsumeBatchedInputScheduled) {
                    scheduleConsumeBatchedInputImmediately();
                }
            }
            return handled ? FINISH_HANDLED : FORWARD;
        }
```

在View的实现中，dispatchPointerEvent的逻辑如下，这样一来，点击事件就传递给了DecorView的dispatchTouchEvent方法。

```java
    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```

DecorView的dispatchTouchEvent的实现如下，需要强调的是，DecorView是PhoneWindow的内部类，还记得前面提到的Window.Callback吗？没错，在下面的代码中，这个cb对象其实就是Activity，就这样点击事件就传递给了Activity了。

PhoneWindow#DecorView:

```java
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker {
	...
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Callback cb = getCallback();
        return cb != null && !isDestroyed() && mFeatureId < 0 ? cb.dispatchTouchEvent(ev)
                : super.dispatchTouchEvent(ev);
    }
	...
}
```

Window:
```java
public void setCallback(Callback callback) {
    mCallback = callback;
}

/**
 * Return the current Callback interface for this window.
 */
public final Callback getCallback() {
    return mCallback;
}
```

这个mCallback就是前面Activity的attach方法里面调用的`mWindow.setCallback(this)`

这样点击事件就传给Activity了。

## 例子

写一个简单的例子，验证下。选择一个View，重写其onTouchEvent方法，然后通过dumpStack方法来打印出当前线程的调用栈信息。

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    Log.d(TAG, "onTouchEvent, ev=" + event.getAction());
    Thread.dumpStack();
    return true;
}
```

选择Google nexus 6运行一下，log如下所示：

```java
06-22 13:25:21.368  7365  7365 D FrameLayoutEx: onTouchEvent, ev=0
06-22 13:25:21.368  7365  7365 W System.err: java.lang.Throwable: stack dump
06-22 13:25:21.368  7365  7365 W System.err:    at java.lang.Thread.dumpStack(Thread.java:490)
06-22 13:25:21.368  7365  7365 W System.err:    at com.ryg.reveallayout.ui.FrameLayoutEx.onTouchEvent(FrameLayoutEx.java:27)
06-22 13:25:21.368  7365  7365 W System.err:    at android.view.View.dispatchTouchEvent(View.java:9294)
06-22 13:25:21.368  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2547)
06-22 13:25:21.368  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2240)
06-22 13:25:21.368  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2553)
06-22 13:25:21.369  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2197)
06-22 13:25:21.369  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2553)
06-22 13:25:21.369  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2197)
06-22 13:25:21.369  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:2553)
06-22 13:25:21.369  7365  7365 W System.err:    at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2197)
06-22 13:25:21.369  7365  7365 W System.err:    at com.android.internal.policy.PhoneWindow$DecorView.superDispatchTouchEvent(PhoneWindow.java:2403)
06-22 13:25:21.369  7365  7365 W System.err:    at com.android.internal.policy.PhoneWindow.superDispatchTouchEvent(PhoneWindow.java:1737)
06-22 13:25:21.369  7365  7365 W System.err:    at android.app.Activity.dispatchTouchEvent(Activity.java:2765)
06-22 13:25:21.369  7365  7365 W System.err:    at com.android.internal.policy.PhoneWindow$DecorView.dispatchTouchEvent(PhoneWindow.java:2364)
06-22 13:25:21.369  7365  7365 W System.err:    at android.view.View.dispatchPointerEvent(View.java:9514)
06-22 13:25:21.369  7365  7365 W System.err:    at android.view.ViewRootImpl$ViewPostImeInputStage.processPointerEvent(ViewRootImpl.java:4230)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:4096)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:3642)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:3695)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:3661)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:3787)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:3669)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:3844)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:3642)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:3695)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:3661)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:3669)
06-22 13:25:21.370  7365  7365 W System.err:    at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:3642)
06-22 13:25:21.371  7365  7365 W System.err:    at android.view.ViewRootImpl.deliverInputEvent(ViewRootImpl.java:5922)
06-22 13:25:21.371  7365  7365 W System.err:    at android.view.ViewRootImpl.doProcessInputEvents(ViewRootImpl.java:5896)
06-22 13:25:21.371  7365  7365 W System.err:    at android.view.ViewRootImpl.enqueueInputEvent(ViewRootImpl.java:5857)
06-22 13:25:21.371  7365  7365 W System.err:    at android.view.ViewRootImpl$WindowInputEventReceiver.onInputEvent(ViewRootImpl.java:6025)
06-22 13:25:21.371  7365  7365 W System.err:    at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:185)
06-22 13:25:21.371  7365  7365 W System.err:    at android.os.MessageQueue.nativePollOnce(Native Method)
06-22 13:25:21.371  7365  7365 W System.err:    at android.os.MessageQueue.next(MessageQueue.java:323)
06-22 13:25:21.371  7365  7365 W System.err:    at android.os.Looper.loop(Looper.java:135)
06-22 13:25:21.371  7365  7365 W System.err:    at android.app.ActivityThread.main(ActivityThread.java:5417)
06-22 13:25:21.371  7365  7365 W System.err:    at java.lang.reflect.Method.invoke(Native Method)
06-22 13:25:21.371  7365  7365 W System.err:    at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)
06-22 13:25:21.371  7365  7365 W System.err:    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)
```

通过上述log，大家不难看出MotionEvent的来源以及传递顺序，本文止。