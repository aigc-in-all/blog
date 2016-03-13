title: About Android Splash Screen
date: 2016-03-11 19:39:59
categories: Android
tags: splash screen
comments: true
---

在Android开发中，个人感觉有一个很影响体验的问题，那就是闪屏（这个词可能是从Splash Scrren翻译过来的）。

> The launch screen is a user’s first experience of your application.

找两个例子对比一下先：

![](http://7xrcq5.com1.z0.glb.clouddn.com/splash_screen_myapp.gif)

![](http://7xrcq5.com1.z0.glb.clouddn.com/splash_screen_google_maps.gif)

很明显Google Map的体验好一些。在好奇心的驱使下，反编译了Google Map，我们一起来看看它是怎样实现的。

<!-- more -->

首先看它的AndroidManifest.xml，看看它的Launcher Activity有没有什么特殊的配置。

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
	<application ...>
	...
        <activity 
        	android:alwaysRetainTaskState="true" 
	        android:configChanges="orientation|uiMode|screenSize|fontScale" 
	        android:launchMode="singleTask" 
	        android:name="com.google.android.maps.MapsActivity" 
	        android:screenOrientation="user" 
	        android:theme="@style/GmmTheme.SplashScreen">
	            <intent-filter>
	                <action android:name="android.intent.action.MAIN"/>
	                <category android:name="android.intent.category.LAUNCHER"/>
	                <category android:name="android.intent.category.APP_MAPS"/>
	            </intent-filter>
	...
    </application>
</manifest>
```

可以看到Mapsactivity设置了一个SplashScreen的Theme：

```
<resources>
...
	<style name="GmmTheme" parent="@android:style/Theme.Holo.Light.NoActionBar">
		<item name="android:textColorHint">@color/no_type_grey</item>
	</style>
	<style name="GmmTheme.SplashScreen" parent="@style/GmmTheme">
        <item name="android:windowBackground">@drawable/local_new_launchscreen_maps</item>
    </style>
...
</resources>
```

接下来找到这个local_new_launchscreen_maps：

```
<layer-list android:opacity="opaque"
  xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@android:color/white" />
    <item android:bottom="@dimen/launchscreens_product_logo_bottom">
        <bitmap 
	        android:gravity="center" 
	        android:src="@drawable/product_logo_maps_color_144" />
    </item>
    <item android:bottom="@dimen/launchscreens_google_logo_bottom">
        <bitmap 
	        android:gravity="bottom|center" 
	        android:src="@drawable/googlelogo_dark20_color_120x44" />
    </item>
</layer-list>

```

这里**product_logo_maps_color_144.png**是中间显示的类似App Logo，**googlelogo_dark20_color_120x44.png**自然是下面那个`Google`字样的图片了。

看到这里应该明白了。

关于闪屏，还可参考：

https://www.youtube.com/watch?v=pEGWcMTxs3I&feature=youtu.be&t=1434

https://www.google.com/design/spec/patterns/launch-screens.html#

http://www.cyrilmottier.com/2012/05/03/splash-screens-are-evil-dont-use-them/