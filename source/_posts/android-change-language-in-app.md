title: Android应用内国际化
date: 2017-02-26 15:22:24
categories: Android
tags:
- 国际化
comments: true
---

最近Android项目需要做国际化，支持多种语言，借此机会学习学习。

我们首先看一下微信的。

微信首页：

```bash
➜ /Users/heqingbao adb shell dumpsys activity com.tencent.mm | grep -e 'TASK' -e "ACTIVITY"
TASK com.tencent.mm id=1801
  ACTIVITY com.tencent.mm/.ui.LauncherUI 25527ac4 pid=5962
```

进入到`多语言`设置页面后：

```bash
➜ /Users/heqingbao adb shell dumpsys activity com.tencent.mm | grep -e 'TASK' -e "ACTIVITY"
TASK com.tencent.mm id=1801
  ACTIVITY com.tencent.mm/.plugin.setting.ui.setting.SettingsLanguageUI 1d07a70 pid=5962
  ACTIVITY com.tencent.mm/.plugin.setting.ui.setting.SettingsAboutSystemUI 14ce296 pid=5962
  ACTIVITY com.tencent.mm/.plugin.setting.ui.setting.SettingsUI 8f2316c pid=5962
  ACTIVITY com.tencent.mm/.ui.LauncherUI 25527ac4 pid=5962
```

切换语言完成后：

```bash
➜ /Users/heqingbao adb shell dumpsys activity com.tencent.mm | grep -e 'TASK' -e "ACTIVITY"
TASK com.tencent.mm id=1805
  ACTIVITY com.tencent.mm/.ui.LauncherUI 2a9c43ff pid=5962
```

注意上面打印的结果，切换语言后：

1. 栈里只保留了LauncherUI
2. 切换前后LauncherUI不是同一个对象。由`25527ac4`变成`2a9c43ff`。
3. 切换前后Task也不是同一个。由`1801`变成了`1805`。

由此，我们至少获取了几点信息：

1. 微信切换语言后，会重建Task。
2. 各页面的名称
	* 首页：com.tencent.mm.LauncherUI
	* 语言设置页：com.tencent.mm.SettingsLanguageUI

<!-- more -->

接下来反编译微信，看看它是怎么实现的。

具体反编译过程就不仔细说了，最后找到切换语言的关键实现：

![][100]
![][101]

由此可见，主要是通过更改`Resource.Configuration.locale`实现的。

-------------------
写个Demo验证一下呗~

Demo中切换语言的关键代码是操作`Resources`对象实现的：

```java
Resources res = MainApplication.getContext().getResources();
Configuration config = res.getConfiguration();
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
    config.setLocale(locale);
} else {
    config.locale = locale;
}
res.updateConfiguration(config, res.getDisplayMetrics());
```

需要注意的地方：

1. 需要在App的入口处（Application）初始化local。
2. 默认情况下，系统切换语言后，各App已经打开的Activity回到前台之前都会自动重建以刷新的语言，这是系统默认处理的。另外切换语言会发一个系统广播`android.intent.action.LOCALE_CHANGED`，并且调用Application的`onConfigurationChanged`方法，可以在这两个位置处理避免应用内的语言一直跟随系统设置。

Demo地址：[https://android.googlesource.com/platform/packages/apps/Settings/](https://android.googlesource.com/platform/packages/apps/Settings/)

[1]: https://github.com/shwenzhang/AndResGuard "AndResGuard"
[100]: android-change-language-in-app/WX20170224-182020.png "通过grep找到全局找字符串"
[101]: android-change-language-in-app/WX20170227-142853.png "jd-gui查看结果"
