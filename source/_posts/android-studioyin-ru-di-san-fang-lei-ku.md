---
layout: post
title: "Android Studio引入第三方类库"
date: 2014-03-23 01:57:13 +0800
comments: true
categories: Android
---
`Android Studio`中引入第三方类库有两种方式，一种是引入本地的jar包，另一种是远程的，类似于Maven里面添加dependency。

---
### 添加本地jar
把jar文件放到工程模块下面的`libs`文件夹下面，没有就新建一个。然后在当前模块下面的`build.gradle`添加本地dependency:

```
dependencies {
	...
    compile files('libs/weibosdkcore.jar')
    ...
}

```

或者直接指定添加某个目录下面的所有类库

```
dependencies {
	...
    compile fileTree(dir: 'libs', include: '*.jar’)
    ...
}
```
---
### 添加一个Maven仓库中的jar
这里拿`commons-io`这个类库为例：

```
dependencies {
	...
    compile 'commons-io:commons-io:2.4'
    ...
}
```
---
### 引入另外一个library工程源码
在Android Studio里面，引入一个library工程源码相对来说稍微麻烦一点。这里就拿[ViewPagerIndicator](https://github.com/JakeWharton/Android-ViewPagerIndicator/)来举例。假设说我们要把它项目里面的library项目源码引入到Android Studio里面来的话，需要把基于Eclipse的项目移植到Android Studio上面来。
先把library工程import到Eclipse里面，然后右建工程名，选择`export`然后选择`Android`列表下的`Generate gradle build files`，之后会为这个项目生成一个build.gradle的文件，这正是Android Studio所需要的。
然后把library整个目录拷贝到我们的目标workspace里面，与我们的目标工程文件夹（假设说叫app）平行的某个位置，并且改名叫ViewPagerIndicator。然后在Android Studio里面就可以看到一个叫ViewPagerIndicator的目录了，根据自己的需求删掉一些没用的目录，比如gen和bin。

然后修改在workspace级别目录下面的`settings.gradle`，把新添加的module添加进来：

```
include ':app','ViewPagerIndicator'
```

然后再在我们的app模板里面引入刚添加的library module，修改app下面的build.gradle
```
dependencies {
	...
    compile project(':ViewPagerIndicator')
    ...
}
```

至此，应该就ok了。

