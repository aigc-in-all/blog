title: Activity LaunchMode
date: 2015-03-22 15:50:49
categories: Android
tags:
- Android 
- LaunchMode
comments: true
---

## Activity的LaunchMode

我们知道，在默认情况下，当我们多次启动同一个Activity的时候，系统会创建多个实例并把它们一一放入任务栈中，当我们单周Back返回时，会发现这些Activity会一一返回。任务栈是一种`后进先出`的栈结构，这个比较好理解，每按一下Back键就会有一个Activity出栈，直到栈空为止，当栈中无任何Activity的时候，系统会回收这个任务栈。

目前有四种启动模式：standard、singleTop、singleTask和singleInstance。

<!-- more -->

#### standard

 标准模式，这也是系统的默认模式。每次启动一个Activity都会重新创建一个新的实例，不管这个实例是否已经存在。被创建的实例的生命周期符合典型 情况下的Activity的生命周期，它的onCreate、onStart、onResume都会被调用。这是一种典型的多实例实现，一个任务栈中可以有多个实例。每个实例也可以属于不同的任务栈。在这种模式下，谁启动了这个Activity，那么这个Activity就支行在启动它的那个Activity所在的栈中。比如Activity A启动了Activity B（B是标准启动模式），那么B就会进入 到A所在的栈中。当我们用ApplicationContext去启动standardActivity的时候会报错，错误如下：

```java
E/AndroidRuntime(674):	android.util.AndroidRuntimeException:	Calling startActivity from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?
```

这是因为standardActivity默认会进入启动它的Activity所属的任务栈中，但是由于非Activity类型的Context（如ApplicationContext）并没有所谓的任务栈，所以这就有问题了。解决这个问题的方法是为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样启动的时候就会为它创建一个新的任务栈，这个时候待启动的Activity实际上以singleTask模式启动的。

#### singleTop

栈顶复用模式。在这种模式下，如果新Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被调用，通过此方法的参数我们可以取出当前Intent的信息。需要注意的是，这个Activity的onCreate、onStart方法不会被系统调用，因为它并没有发生改变。如果新Activity的实例已存在但不是位于栈顶，那么新Activity依然会重新创建。举个粟子，假设目前栈内的情况为ABCD，其中ABCD为四个Activity，A位于栈底，D位地栈顶，这个时候假设要再次启动D，如果D的启动模式是singleTop，那么栈内的情况仍然为ABCD；如果D的启动模式为standard，那么由于D被重新创建，导致栈内的情况就会变为ABCDD。

#### singleTask

栈内复用模式。这是一种单实例模式，在这种模式下，只要Activity在一个栈中存在 ，那么多次启动都不会重新创建实例，和singleTop一样，系统也会调用其onNewIntent。具体一点，当一个具有singleTask模式的Activity请求启动后，比如Activity A，系统首先会寻找是否存在A想要的任务栈，如果不存在，就重新创建一个任务栈，然后创建A的实例后把A放到栈中。如果存在A需要的任务栈，这时要看A是否在栈中有实例存在 ，如果有实例存在，那么系统就会把A调到栈顶并调用它的onNewIntent方法，如果实例不存在，就创建A的实例并把A压入栈中。举几个例子：

* 比如目前任务栈S1中的情况为ABC，这个时候D以singleTask模式请求启动，共所需要的任务栈为S2，由于S2和D的实例均不存在，所以系统会先创建任务栈S2，然后再wbffD的实例并将其入栈到S2.

* 另外一种情况，假设D所需的任务栈为S1，其它情况如下面例子所示，那么由于S1已经存在，所以系统会直接创建D的实例并将其入栈到S1.

* 如果D所需的任务栈为S1，并且当前任务栈S1的情况下ADBC，根本栈内利用的原则，此时D不会重新创建，系统会把D切换到栈顶并调用其onNewIntent方法，同时由于singleTask默认具有clearTop效果，会导致栈内所有在D上面的Activity全部出栈，于是最终S1的情况为AD。

#### singleInstance 

单实例模式，这是一种加强的singleTask模式，它除了具有singleTasl模式的所有特性外，还加强了一点，那就是具有此种模式的Activity只能单独地位于一个任务栈中，换句话说，比如A是singleInstance模式，当A启动后，系统会为它创建一个新的任务栈，然后A独自在这个新的任务栈中，由于栈内复用的特性，后续的请求均不会创建新的Activity，除非这个独特的任务栈被系统销毁了。

这里需要指出一种情况，我们假设目前有两个任务栈，前台任务栈的情况为AB，后台任务栈的情况为CD，这里假设CD的启动模式均为singleTask。现在请求启动D，那么整个后台任务栈都会被切换到前台，这个时候整个后退列表变成了ABCD。当用户按Back返回的时候，列表中的Activity会一一出栈。如果不是请求启动D而是启动C，那么整个后退表就变成ABC。

----------------

另外一个问题，在singleTask启动械模式中多次提到某个Activity所需的任务栈，什么是Activity所需的任务栈呢？这要从一个参数说起：TAskAffinity，可以翻译为任务相关性。这个参数标识了一个Activity所需要的任务栈的名字，默认情况下，所有Activity所需的任务栈的名字为应用的包名。当然我们可以为每个Activity都单独指定TaskAffinity属性，这个属性值必须不能和包名相同，否则就相当于没有指定。TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用，在其它模式下没有意义 。另外，任务栈分为前台任务栈和后台任务栈，后台任务栈中的Activity位为暂停状态，用户可以通过切换将后台任务栈再次调到前台。

当TaskAffinity和singleTask启动模式配对使用的时候，它是具有该 模式的Activity的目前任务栈的名字，待启动的Activity会支行在名字和TaskAffinity相同的任务栈中。

当TaskAffinity和allowTaskReparenting结合的时候，这种情况比较复杂，会产生特殊的效果。当一个应用A启动了应用B的某个Activity后，如果这个Activity的allowTaskReparenting属性为True的话，那么当应用B被启动后，此Activity会直接从应用A的任务栈转移到应用B的任务栈中。比如现在有两个应用A和B，A启动了B的一个Activity C，然后按Home键回到桌面，然后再单击B的桌面图标，这个时候并不是启动了B的主Activity，而是重新显示了已经被应用A启动的Activity C，或者说，C从A的任务栈转移 到了B的任务栈中。可以这么理解，由于A启动了C，这个时候 C只能运行在A的任务栈中，但是C属于B应用，正常情况下，它的TaskAffinity值能和A的任务栈相同（因为包名不同）。所以，当B被启动后，B会创建自己的任务栈，这个时候系统发现C原本所想要的任务栈已经被创建了，所以就把C从A的任务栈中转移过来了。

有两种方法给Activity指定启动模式：第一种是通过AndroidMenifest为Activity指定启动模式：

```xml
<activity
	android:anme"xxx"
	android:configChanges="xxx"
	android:launchMode="singleTask"
	android:label="@string/app_name" />
```

另一种情况是通过在Intent中设置标志位来为Activity指定启动模式，比如：

```java
Intent intent = new Intent(this, SecondActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

这两种方式都可以为Activity指定启动模式，但是二者还是有区别的。首先，优先级上，第二种方式的优先级要高于第一种，当两种同时存在时，以第二种方式为准。其次，上述两种方式在限定范围上有所不同，比如，第一种方式无法直接为Activity设定FLAG_ACTIVITY_CLEAR_TOP标识，第二种方式无法为Activity指定singleInstance模式。

## Activity的Flags

Activity的Flags有很多，这里主要分析一些比较常用的标识位。标识位的作用有很多，有的标识位可以设定Activity的启动模式，比如FALG_ACTIVITY_NEW_TASK和FALG_ACTIVITY_SINGLE_TOP等。还有的标识位可以影响Activity的运行状态，比如FLAG_ACTIVITY_CLEAR_TOP和FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS等。

* FLAG_ACTIVITY_NEW_TASK
这个标识位的作用是为Activity指定singleTask的启动模式，其效果和在XML中指定该模式相同。

* FALG_ACTIVITY_SINGLE_TOP
这个标识位的作用是为Activity指定singleTop启动模式，其效果和在XML中指定该 模式相同中。

* FALG_ACTIVITY_CLEAR_TOP
具有标记位的Activity，当它启动时，在同一个任务栈中所有位于它上面的Activity都要被出栈。这个标记位一般会和singleTask启动模式一起出现，在这种情况下，被启动Activity的实例如果已经存在，那么系统就会调用它的onNewIntent。如果被启动的Activity采用standard模式启动，那么它连同它之上的Activity都要出栈，系统会创建新的Activity实例并放入栈顶。

* FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS
具体这个标记位的Activity不会出现在历史Activity的列表中，当某些情况下我们不希望用户通过历史列表找到我们的Activity的时候这个标识比较有用。它等同于在XML中指定Activity的属性`android:excludeFromRecents="true"`。

## IntentFilter的匹配规则 

启动Activity分为两种，显式调用和隐式调用。显式调用需要明确地指定被启动的对象的组件信息，包括包名和类名，而隐式调用则不需要明确指定组件信息。原则上一个Intent不应该既是显式调用又是隐式调用，如果二者共存的话以显式调用为主。显式调用很简单，这里主要介绍一下隐式调用。隐式调用需要Intent能够匹配目标组件的IntentFilter中所设置的过滤信息，如果不匹配将无法启动目标Activity。IntentFilter的过滤信息有action、category、data。为了匹配过滤列表，需要同时匹配过滤列表中的action、category、data信息，否则匹配失败。一个过滤列表中的action、category、data可以有多个，所有的action、category、data分别构成不同类别，同一类别的信息共同约束当前类别的匹配过程 。只有一个Intent同时匹配action类别、category类别、data类别才算完全匹配，只有完全匹配才能成功启动目标Activity。另外一点，一个Activity可以有多个intent-filter，一个Intent只要能匹配任何一组intent-filter即可成功启动对应的Activity。

1. action的匹配规则 
action是一个字符串，系统预定义了一些action，同时我们也可以在应用中定义自己的action。action的匹配规则是Intent中的action必须能够和过滤规则中的action匹配，这里说的匹配是指action的字符串值完全一样。一个过滤规则中可以有多个action，那么只要Intent中的action能够和过滤规则中的任何一个action相同即可匹配成功。action的匹配要求Intent中的action存在且必须和过滤规则中的其中一个action相同，这里需要注意它和category匹配规则不同。另外action区分大小写，大小写不同字符串相同的action会匹配失败。

2. category的匹配规则
category是一个字符串，系统预定义了一些category，同进我们也可以在应用中定义自己的category。category的匹配规则和action不同，它要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。换句话说，Intent如果定义了category，不管有几个category，对于每个category来说，它必须是过滤器规则中已经定义了的category。当然，Intent可以没有category，如果没有category的话，按照上面的描述，这个Intent仍然可以匹配成功。这里要注意下它和action匹配过程的不同，action是要求Intent中必须有一个action且必须能够和过滤规则中的某个action相同，而category要求Intent可以没有category，但是如果你一旦有category，不管有几个，每个都要能够和过滤规则中的任何一个category相同。为什么不设置category也可以匹配呢？原因是系统在调用startActivity或者startActivityForResult的时候会默认为Intent加上`android:intent.category.DEFAULT`这个category，所以为了我们的Activity能够接收隐式调用，就必须在intent-filter中指定`android.intent.category.DEFAULT`这个category。

3. data的匹配规则
data的匹配规则和action类似，如果过滤规则中定义了data，那么Intent中必须也要定义可匹配的data。

data的语法：
```xml
<data android:scheme="string"
	android:host="string"
	android:port="string"
	android:path="string"
	android:pathPattern="string"
	android:mimeType="string" />
```

data由两部分组成，mimeType和URI。memeType指媒体类型，比如image/jpeg、audio/mpeg4-generic和video/*等，可以表示图片、文本、视频等不同的媒体格式，而URI中包含的数据就比较多了，下面是URI的结构：

```
<scheme>://<host>:<port>/[<path>|[<pathPrefix>]|[<pathPattern>]]
```
这里再给出几个实际例子就好理解了
```
content://com.example.project:200/folder/subfolder/etc
http://www.baidu.com:80/search/info
```

Scheme:URI的模式，比如http、file、content等，如果URI中没有指定scheme，那么整个URI无效。

Host:URI的主机名，比如www.baidu.com，如果host未指定，URI无效。

Port:URI中的端口号，比如80，仅当URI中指定了scheme和host参数的时候port参数才是有意义的。

Path、pathPattern和pathPrefix: 这三个参数表示路径信息，其中path表示完整的路径信息；pathPattern表示完整的路径信息，但是它里面可以包含通配符`*`，`*`表示0个或多个任意字符，需要注意的是，由于正则表达式的规范，如果想表示真实的字符串，那么`*`要写成`\\*`，`\`要写成`\\\\`；pathPrefix表示路径的前缀信息。

介绍完data的数据格式后，我们要说一下data的匹配规则了。前面说到，data的匹配规则和action类似，它也要求Intent中必须含有data数据，并且data数据能够完全过滤规则中的某一个data，这里的完全匹配是指过滤规则中出现的data部分也出现在了Intent中的data中。

* 如过滤如下规则：
```xml
<intent-filter>
	<data android:mimeType="image/*" />
	...
</intent-filter>
```
这种规则指定了媒体类型为所有类型的图片，那么Intent中的mimeType属性必须为`image/*`才能匹配，这种情况下虽然过滤规则没有指定URI，但是却有默认值，URI的默认值为content或file。也就是说，虽然没有指定URI，但是Intent中的URI部分的scheme必须为content或者file才能匹配，这点是需要尤其注意的。为了匹配上面的规则，我们可以写出如下示例：
```java
intent.setDAtaAndType(Uri.parse("file://abc"), "image/png");
```

另外，如果要为Intent指定完整的data，必须要调用setDataAndType方法，不能先计用setData再调用setType，因为这两个方法彼此会清除对方的值：

Intent#setData
```java
public Intent setData(Uri data) {
    mData = data;
    mType = null;
    return this;
}
```
可以发现，setData会把mimeType置为null，同理setType也会把URI置为null。

* 如下过滤规则：
```xml
<intent-filter>
	<data android:mimeType="video/mpeg" android:scheme="http" ... />
	<data android:mimeType="video/mpeg" android:scheme="http" ... />
</intent-filter>
```

这种规则指定了两组data规则，且每个data都指定了完整的属性值，既有URI又有mimeType。为了匹配上面的规则，可以这样：
```java
intent.setDataAndType(Uri.parse("http://abc"), "video/mpeg");
```
或者
```java
intent.setDataAndTyep(Uri.parse("http://abc"), "audio/mpeg");
```

通过上面两个示例，应该已经明白了data的匹配规则，关于data还有一个特殊情况需要说明下，这也是它和action不同的地方，如下两种特殊的写法，它们的作用是一样的：
```xml
<intent-filter>
	<data android:scheme="file" android:host="www.baidu.com" />
	...
</intent-filter>
<intent-filter>
	<data android:scheme="file" />
	<data android:host="www.baidu.com" />
	...
</intent-filter>
```

另外一点，intent-filter匹配规则对于Service和BroadcastReceiver也是同样的道理，不过系统对于Service的建议是尽量使用显式调用方式来启动服务。

最后当我们通过隐式方式启动一个Activity的时候，可以做一下判断，看是否有Activity能够匹配我们的隐式Intent，如果不做判断就有可能出现错误。判断方法有两种：采用PackageManager的resolveActivity方法或者Intent的resolveActivity方法，如果它们找不到匹配的Activity就会返回null，我们通过判断返回值就可以规避错误。另外PackageManager还提供了queryIntentActivities方法，这个方法和resolveActivity方法不同的是：它不是返回最佳匹配的Activity信息而是返回所有成功匹配的Activity信息：
```java
public abstract List<ResolveInfo> queryIntentActivities(Intent intent, int flags);
public abstract ResolveInfo resolveActivity(Intent intent, int flags);
```
上述两个方法第一个参数比较好理解，第二个参数需要注意，我们要使用MATCH_DEFAULT_ONLY这个标记位，这个标记位的含义是仅仅匹配那些在intent-filter中声明了`<category android:name="android.intent.category.DEFAULT">`这个category的Activity。使用这个标记位的意义在于，只要上述两个方法不返回null，那么startActivity一定可以成功。如果不用这个标记位，就可以把intent-filter中category不含DEFAULT的那些activity给匹配出来，从而导致startActivity可能失败。因为不含有DEFAULT这个category的Activity是无法接收隐式Intent的。在action和category中，有一类aciton和category比较重要，它们是：
```xml
<action android:name="android.intent.action.MAIN" />
<category android:name="android.intent.category.LAUNCHER" />
```

这二者共同作用是用来标明这是一个入口Activity并且会出现在系统的应用列表中，少了任何一个都没有意义，也无法出现在系统的应用列表中，也就是二者缺一不可。另外针对Service和BroadcastReceiver，PackageManager同样提供了类似的方法去获取成功匹配的组件信息。