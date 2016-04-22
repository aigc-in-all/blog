title: Android Handler源码分析
date: 2016-04-21 20:55:14
tags: Handler
categories: Android
comments: false
---

## Handler

在Android开发中，Handler被经常用来更新UI，先通过一个例子看一下Handler的用法：

```
public class MainActivity extends Activity {
	
	private TextView mTextView;

	private Handler mHandler = new Handler() {
		@Override
		public void handleMessage(Message msg) {
			mTextView.setText("UI成功更新");
		}
	}

	@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTextView = (TextView) findViewById(R.id.text_view);

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                mHandler.sendEmptyMessage(1);
            }
        }).start();
    }
}
```

上面的代码先创建了一个Handler实例，并且重写了handleMessage方法，在这个方法里，便是根据接受到的消息类型进行相应的UI更新。那么看一下Handler的构造方法的源码：

<!-- more -->

```

public class Handler {

    /**
     * Default constructor associates this handler with the {@link Looper} for the
     * current thread.
     *
     * If this thread does not have a looper, this handler won't be able to receive messages
     * so an exception is thrown.
     */
    public Handler() {
        this(null, false);
    }

    /**
     * Use the {@link Looper} for the current thread with the specified callback interface
     * and set whether the handler should be asynchronous.
     *
     * Handlers are synchronous by default unless this constructor is used to make
     * one that is strictly asynchronous.
     *
     * Asynchronous messages represent interrupts or events that do not require global ordering
     * with respect to synchronous messages.  Asynchronous messages are not subject to
     * the synchronization barriers introduced by {@link MessageQueue#enqueueSyncBarrier(long)}.
     *
     * @param callback The callback interface in which to handle messages, or null.
     * @param async If true, the handler calls {@link Message#setAsynchronous(boolean)} for
     * each {@link Message} that is sent to it or {@link Runnable} that is posted to it.
     *
     * @hide
     */
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

}

```

在构造方法中，通过调用Looper.myLooper()获得了Looper对象。如果mLooper为空，那么会抛异常`Can't create handler inside thread that has not called Looper.prepare()`，意思是：不能在未调用Looper.prepare()的线程创建Handler。上面的例子并没有调用这个方法，但是却没有抛出异常。其实是因为主线程在启动的时候已经帮我们调用过了(在ActivityThread里面)，所以可以直接创建Handler。如果是在其它子线程，直接创建Handler是会抛异常的。

在得到Handler之后，又获取了它的内部变量mQueue，这是MessageQueue对象，也就是消息队列，用于保存Handler发送的消息。

到此，Android的消息机制的三个重要角色全部出现了，分别是Handler，Looper和MessageQueue。一般在代码中我们接触比较多的是Handler，但Looper和MessageQueue却是Handler运行时不可或缺的。

## Looper

前面分析了Handler的构造方法，其中调用了Looper.myLooper()方法，下面是它的源码：

```
public final class Looper {

    // sThreadLocal.get() will return null unless you've called prepare().
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
}
```

这个方法代码很简单，就是从sThreadLocal中获取Looper对象，sThreadLocal是ThreadLocal对象，这说明Looper是线程独立的。

在Handler的构造方法中，从抛出的异常可知，每个线程想要获得Looper需要调用prepare()方法。那这个prepare()是在哪里调用的呢？答案在ActivityThread里面：

```


/**
 * This manages the execution of the main thread in an
 * application process, scheduling and executing activities,
 * broadcasts, and other operations on it as the activity
 * manager requests.
 *
 * {@hide}
 */
public final class ActivityThread {

    public static void main(String[] args) {
        
        ...
        
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        ...

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}

```

看到main方法有没有种很熟悉的感觉？这里是我们的main线程，也是App的入口 。从源码可知，在main方法里面调用了`Looper.prepareMainLooper()`，再来看看Looper：

```
    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

从上面可以看出，sThreadLocal设置了一个全新的Looper，不过需要注意的是如果sThreadLocal已经设置过了，那么会抛出异常，也就是说一个线程只会有一个Looper。创建Looper的时候，内部会创建一个消息列队。

现在的问题是，Looper看上去很重要的样子，它到底是干嘛的？这里不防先告诉答案：Looper开启消息循环，不断从消息队列MessageQueue取出消息交由Handler处理。

为什么这样说呢，看一下Looper的loop方法：

```
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        ...

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            ...

            msg.target.dispatchMessage(msg);

            ...

            msg.recycleUnchecked();
        }
    }
```

可以看出，在这个方法内部有个死循环，里面通过MessageQueue的next()方法获取下一条消息，没有获取到会阻塞。如果成功获取，便调用`msg.target.dispatchMessage(msg)`，msg.target是Handler对象(下面会讲到)，dispatchMessage则是分发消息（此时已经运行在UI线程），下面分析消息的发送及处理流程。

## MessageQueue

这部分在下篇文章里面分析，暂时略过，不影响后面的理解。

## 消息发送与处理

在调用Handler程发送消息时，是调用一系列的sendMessage方法，最终会辗转调用sendMessageAtTime(Message msg, long uptimeMillis)，代码如下：

```
public class Handler {

    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
}
```

这个方法就是调用enqueueMessage在消息队列中插入一条消息，在enqueueMessage中，会把msg.target设置为当前的Handler对象。

消息插入到队列后，Looper负责从队列中取出，然后调用Handler的dispatchMessage方法。接下来看看这个方法怎么处理消息的：

```
public class Handler {

    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }

    private static void handleCallback(Message message) {
        message.callback.run();
    }

    /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }
}

```

首先，如果消息的callback不是空，便调用handleCallback处理。否则判断Handler的mCallback是否为空，不为空则调用它的handleMessage方法。如果仍然为空（或者Handler的mCallback返回false），才调用Handler的handleMessage，也就是我们创建Handler时重写的那个方法。

如果发送消息时调用Handler的post(Runnable r)方法，会把Runnable封装到消息对象的callback，然后调用sendMessageDelayed，相关代码如下 ：

```
public class Handler {
    public final boolean post(Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
}
```

此时在dispatchMessage中便会调用handleCallback进行处理。可以发现它是直接调用run方法处理消息。

如果在创建Handler时，直接提供一个Callback对象，消息就交给这个对象的handleMessage方法处理。Callback是Handler内部的一个接口：

```
public class Handler {
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
}
```

以上便是消息发送与处理的流程，发送时是在子线程，但处理时dispatchMessage方法运行在主线程。

终上，我们发现在App启动的时候，在主线程（UI线程）中创建了Looper，然后我们在UI线程中创建一个Handler（new Handler()/new Handler(Looper.getMainLooper()))，这样在子线程中不用做任何处理，即可使用handler发送消息。那么我们可以在子线程中创建Handler吗？前面已经提到过，这里仔细分析一下为什么不行。

在前面所贴的源码中，有一段

```
public class Handler {
    public Handler(Callback callback, boolean async) {
        
        ...

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
}
```

从源码来看，如果直接在子线程中new Handler()会抛RuntimeException。因为从myLooper()方法的注释也可以看出来，这里的myLooper()方法返回的是与当前线程相关联的Looper（还记得ThreadLocal的作用吧？），很显然，我们并没有给sThreadLocal设置Looper。

那么难道在子线程中就不能使用Handler/Looper机制吗？当然不是，从上面的异常信息也可以看出，需要先调用Looper.prepare()。其实从另一个角度去想也很好理解，主线程也是一个线程（看上去比较特殊而已），子线程也是一个线程，常规来说主线程能实现的，子线程应该也能实现。从前面的源码分析可以知道，ActivityThread在main方法里面调用了Looper.prepare()，而Looper.prepare()方法里面会给sThreadLocal设置一个Looper，然后再调用Looper.loop()开启消息循环。

所以，如果我们要在子线程中创建Handler处理消息的话，一样需要先调用Looper.prepare()方法，再new Handler()就不会有问题了，当然最后不要忘了调用Looper.loop()开启消息循环。并且注意在子线程中事情处理完成后调用quit()退出消息循环。

## 总结 

至此，Android的消息处理机制的原理就分析结束了，现在可以知道，消息处理是通过Handler、Looper以及MessageQueue共同完成。Handler负责发送以及处理消息，Looper创建消息队列并不断从队列中取出消息交给Handler，MessageQueue则用于保存消息。





