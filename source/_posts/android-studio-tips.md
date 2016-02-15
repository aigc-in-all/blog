---
layout: post
title: "Android Studio Tips"
date: 2014-03-23 01:26:50 +0800
comments: true
categories: Android
---
关于`Android Studio`在引入第三方依赖包后可能会出现以下异常：

`Duplicate files copied in APK META-INF/LICENSE.txt`

或者

`Duplicate files copied in APK META-INF/NOTICE.txt`

然后异常信息指向两个或者多个相同名字的jar文件。解决办法是在`build.gradle`文件里面添加添加配置过滤掉这些文件：

```
android {
...
	packagingOptions {
        exclude 'META-INF/LICENSE.txt'
        exclude 'META-INF/NOTICE.txt'
    }
...
}
```