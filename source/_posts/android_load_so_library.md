title: Android加载SO库测试
date: 2017-02-25 18:24:24
categories: Android
tags:
- jni
- Android
comments: false
reward: false
---

Android应用开发中会经常遇到引用第三方库的情况，特别是一些第三方的SDK，往往会携带一些必须的so静态库文件。正常情况下我们只需要将不同ABI结构下的.so文件分别放置。但很多时候第三方SDK并没有提供全量的ABI编译版本，比如有些SDK只提供了`armeabi`，有些只提供了`armeabi-v7a`，还有一些`armeabi``armeabi-v7a`都提供了，那么遇到这些情况的时候，我们应该怎样放置这些`.so`呢？（关于什么是`ABI`等这些问题参考[官方文档][50]）

下面来看几个例子：

**测试环境**：

* Meizu M355： *armeabi-v7a*

* 模拟器：*armeabi*

**测试一**

如下图所示，我们分别在工程中的`armeabi`和`armeabi-v7a`中都放了相应的`.so`文件，但是注意名字是不一样的：

![][3]

下图是我在`Meizu M355`手机上看到的结果（注意这个目录需要root权限才能查看，如果没有Root的手机，使用模拟器也可以）：

![][1]

在`模拟器`上运行的结果：

```bash
root@android:/data/data/com.heqingbao.demo/lib # ll
-rwxr-xr-x system   system      28076 1979-11-30 00:30 liba.so
-rwxr-xr-x system   system      28076 1979-11-30 00:30 libb.so
```

从上图可以看出来，`Meizu M355`只加载了`armeabi-v7a`里面的so文件，而`模拟器`却只加载了`armeabi`中的so文件。

<!-- more -->

**测试二**

下面我们删除工程中的`armeabi-v7a`，只保留`armeabi`：

![][4]

仍然使用`Meizu M355`查看结果：

![][2]

在`模拟器`上运行的结果：

```bash
root@android:/data/data/com.heqingbao.demo/lib # ll
-rwxr-xr-x system   system      28076 1979-11-30 00:30 liba.so
-rwxr-xr-x system   system      28076 1979-11-30 00:30 libb.so
```

可以看出，此时`Meizu M355`和`模拟器`都只加载了`armeabi`中的`so`文件。

**测试三**

举个实际点的例子，假设我们的工程中需要依赖[mars][51]：

```
compile 'com.tencent.mars:mars-wrapper:1.1.3'
```

待依赖同步完成后，

![][5]

可以看出，mars自己有对应的so库依赖（注意这个目录在`build/intermediates/exploded-aar`目录里面），并且mars只提供了`armeabi`和`x86`两种。

如果我们的代码工程中只提供了`armeabi`，在`Meizu M355`上面运行起来后的结果：

![][7]

可以看到，mars的相关so文件都被安装到了这部手机上。

如果我们的代码工程既提供了`armeabi`也提供了`armeabi-v7a`呢？

![][6]

可以看出来，mars所依赖的so文件并没有安装到手机上。自然也就不能使用mars的相关功能。

总结：

* **armeabi-v7a**向前兼容**ARM v5**，所以，对只提供**armeabi**版本的.so，可以原样复制一份到**armeabi-v7a**文件夹。
* 为了保证 apk 体积，只保留 **armeabi** 和 **armeabi-v7a** 两个文件夹，并保证这两个文件夹中 .so 数量一致。


[1]: android_load_so_library/1.jpg
[2]: android_load_so_library/2.jpg
[3]: android_load_so_library/3.jpg
[4]: android_load_so_library/4.jpg
[5]: android_load_so_library/5.jpg
[6]: android_load_so_library/6.jpg
[7]: android_load_so_library/7.jpg

[50]: https://developer.android.com/ndk/guides/abis.html
[51]: https://github.com/Tencent/mars