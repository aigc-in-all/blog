title: 理解targetSdkVersion
date: 2016-04-02 13:05:00
tags: targetSdkVersion
categories: Android
comments: true
---

[上一篇文章](http://www.heqingbao.net/2016/03/31/Android-SdkVersion/)介绍了compileSdkVersion、minSdkVersion和targetSdkVersion的含义，以及合理设置各个值的意义。其中compileSdkVersion和minSdkVersion都非常好理解，前者表示编译的SDK版本，后者表示应用兼容的最低SDK版本。但是对于targetSdkVersion其实很难一句话解释清楚，本文试图彻底解决这个问题。

随着Android系统的升级，某个系统的API或者模块的行为可能会发生变化，但是为了保证老APK的行为还是和以前兼容，只要APK的targetSdkVersion不变，即使这个APK安装在新Android系统上，其行为还是保持老的系统上的行为，这样就保证了对老应用的向前兼容性。

这里还是用原文的例子，在Android 4.4 (API 19)以后，AlarmManager的`set()`和`setRepeat()`这两个API的行为发生了变化。在Android 4.4以前，这两个API设置的都是精确的时间，系统能保证在API设置的时间点上唤醒Alarm。因为省电原因Android 4.4系统实现了AlarmManager的对齐唤醒，这两个API设置唤醒的时间，系统都对待成不精确的时间，系统只能保证在你设置的时间点之后某个时间唤醒。

这时，虽然API没有任何变化，但是实际上API的行为却发生了变化，如果老的APK中使用了此API，并且在应用中的行为非常依赖AlarmManager在精确的时间唤醒，例如闹钟应用。如果Android系统不能保证兼容，老的APK安装在新的系统上，就会出现问题。

<!-- more -->

```
public void set (int type, long triggerAtMillis, PendingIntent operation)
```

> Note: Beginning in API 19, the trigger time passed to this method is treated as inexact: the alarm will not be delivered before this time, but may be deferred and delivered some time later. The OS will use this policy in order to "batch" alarms together across the entire system, minimizing the number of times the device needs to "wake up" and minimizing battery use. In general, alarms scheduled in the near future will not be deferred as long as alarms scheduled far in the future.
> 
> With the new batching policy, delivery ordering guarantees are not as strong as they were previously. If the application sets multiple alarms, it is possible that these alarms' actual delivery ordering may not match the order of their requested delivery times. If your application has strong ordering requirements there are other APIs that you can use to get the necessary behavior; see setWindow(int, long, long, PendingIntent) and setExact(int, long, PendingIntent).
> 
> Applications whose targetSdkVersion is before API 19 will continue to get the previous alarm behavior: all of their scheduled alarms will be treated as exact.

Android系统是怎么保证这种兼容性的呢？这时候targetSdkVersion就起作用了。APK在调用系统AlarmManager的`set()`或`setRepeat()`的时候，系统首先会检查调用的APK的targetSdkVersion信息，如果小于19，就还是按照老的行为，即精确设置唤醒时间，否则执行新的行为。

我们来看一下Android 4.4上AlarmManager的一部分源码：

```
    private final boolean mAlwaysExact;

    AlarmManager(IAlarmManager service, Context ctx) {
        mService = service;

        final int sdkVersion = ctx.getApplicationInfo().targetSdkVersion;
        mAlwaysExact = (sdkVersion < Build.VERSION_CODES.KITKAT);
    }
```

看到这里，首选获取应用的 targetSdkVersion，判断是否是小于 Build.VERSION_CODES.KITKAT (即 API Level 19)，来设置 mAlwaysExact 变量，表示是否使用精确时间模式。

```
    private long legacyExactLength() {
        return (mAlwaysExact ? WINDOW_EXACT : WINDOW_HEURISTIC);
    }

    public void set(int type, long triggerAtMillis, PendingIntent operation) {
        setImpl(type, triggerAtMillis, legacyExactLength(), 0, operation, null, null);
    }
```

这里看到，直接影响到 `set()` 方法给 `setImpl()` 传入不同的参数，从而影响到了 `set()` 的执行行为。具体的实现在 AlarmManagerService.java，这里就不往下深究了。

看到这里，发现其实 Android 的 targetSdkVersion 并没有什么特别的，系统使用它也非常直接，甚至很“粗糙”。仅仅是用过下面的 API 来获取 targetSdkVersion，来判断是否执行哪种行为：

```
getApplicationInfo().targetSdkVersion
```

所以，我们可以猜测到，如果 Android 系统升级，发生这种兼容行为的变化时，一般都会在原来的保存新旧两种逻辑，并通过 if-else 方法来判断执行哪种逻辑。果然，在源码中搜索，我们会发现不少类似 getApplicationInfo().targetSdkVersion < Buid.XXXX 这样的代码，相对于浩瀚的 Android 源码量来说，这些还是相对较少了。其实原则上，这种会导致兼容性问题的修改还是越少越好，所以每次发布新的 Android 版本的时候，Android 开发者网站都会列出做了哪些改变，在[这里](http://developer.android.com/intl/zh-cn/about/versions/lollipop.html)，开发者需要特别注意。

最后，我们也可以理解原文中说的那句话的含义，明白了为什么修改了 APK 的 targetSdkVersion 行为会发生变化，也明白了为什么修改 targetSdkVersion 需要做完整的测试了。


