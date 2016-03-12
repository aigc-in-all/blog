title: Android IPC
date: 2015-04-5 19:28:40
tags: ipc
categories: Android
comments: true
---

## Android IPC简介

IPC是Inter-Process Communication的缩写，含义为进程间通信或才跨进程通信，是指两个进程之间进行数据交换的过程。说起进程间通信，我们首先要理解什么是进程，什么是线程。进程和线程是截然不同的概念。按照操作系统中的描述，线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般指一个执行单元，在PC和移动设备上指一个程序或一个应用。一个进程可以包含多个线程，因此进程与线程是包含与被包含的关系。最简单的情况下，一个进程可以只有一个线程，即主线程，在Android里面主线程也叫UI线程，在UI线程中才可以操作界面元素。很多时候，一个进程中需要执行大量耗时任务，如果这些任务放在主线程中去执行就会造成界面无法响应，严重影响用户体验，这种情况在PC和移动操作系统上都存在，在Android中有一个特殊的名字叫做ANR（Application Not Response），即应用无响应。解决这个问题就要用到线程，把一些耗时任务放到线程里面去执行。

IPC不是Android中所独有的，任何一个操作系统都需要相应的IPC机制，比如Windows上可以通过剪贴板来进行进程间通信，Linux上可以通过命名管道、共享内存、信号量等来进行进程间通信。不同的操作系统平台有着不同的进程间通信方式。对于Android来说，它是一种基于Linux内核的移动操作系统，它的进程间通信方式并不是完全继承自Linux，它也有自己的进程间通信方式。在Android中最有特色的进程间通信方式就是Binder了，通过Binder可以轻松实现进程间通信。除了Binder，Android还支持Socket，通过Socket也可以实现任意两个终端之间的通信，当然一个设备上的两个进程通过Socket通信也是可以的。

只有在多进程的场景下才需要使用IPC跨进程通信，多进程的情况分为两种。第一种情况是一个应用因为某些原因自身需要采用多进程模式来实现，比如为了加大一个应用可使用的内存所有需要通过多进程来获取多份内存空间。Android对单个应用使用的最大内存做了限制，早期的一些版本可能是16MB，不同设备有不同的大小。另一种情况是当前应用需要向其它应用获取数据，由于不是同一个应用，所以必须采用跨进程的方式来获取所需数据，当我们通过系统提供的ContentProvider去查询数据库的时候，其实也是一种进程间通信，只不过通信细节系统内部屏蔽了。

<!-- more -->

## Android中的多进程模式

### 开启多进程模式

在Android中使用多进程只有一种方法，就是给四大组件（Activity、Service、BroadcastReceiver、ContentProvider）在AndroidMenifest中指定`android:process`属性，除此之外没有其它方法，也就是说无法给一个线程或者一个实体烊指定其运行时所在的进程。其实还有一种非常规的多进程方法，就是通过JNI在native层去fork一个新的进程，但这种方法属于特殊方法。

```java
<activity
	android:name="com.example.MainActivity"
	...
	<intent-filter>
		<action android:name="android.intent.action.Main" />
		<category android:name="android.intent.category.DEFAULT" />
	</intent-filter>
</activity>
<activity
	android:name="com.example.SecondActivity"
	...
	android:process=":remote" />
<activity
	android:name="com.example.ThirdActivity"
	...
	android:process="com.example.remote" />
```

上面示例分别为SecondActivity和ThirdActivity指定了process属性，并且它们属性值不同。假设当前的包为是`com.example`，当SecondActivity启动时，系统会为它创建一个单独的进程`com.example:remote`，当ThirdActivity启动的时候，系统会为它创建一个单独的进程`com.example.remote`.同时入口Activity没有指定process，那它运行在默认进程中，默认进程名是包名。

对于两种命名的区别：

1. `:`的含义是指要在当前的进程名前面附加上当前的包名，这是一种简单的写法，对于SecondActivity来说，它完事的进程名为`com.example:remote`，而对于ThirdActivity的声明方式，它是一种完整的命名方式。

2. 进程名以`:`开头的进程属于当前应用的私有进程，其它应用的组件不可以和它跑在同一个进程中，而进程名不以`:`开头的进程属于全局进程，其它应用通过SharUID方式可以和它跑在同一个进程中。

Android系统会为每个应用分配一个唯一的UID，具有相同UID的应用才能共享数据。这里要说明的是，两个应用通过ShareUID跑在同一个进程中是有条件的，需要这两个应用具有相同的ShareUID并且签名相同才可以。在这种情况下，它们可以互相访问对方的私有数据，比如data目录、组件信息等，不管它们是否跑在同一个进程中。当然如果它们跑在同一个进程中，那么除了共享data目录、组件信息，还可以共享内存数据，或者说它们看起来就像是一个应用的两个部分。

### 多进程模式的运行机制

所有运行在不同进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败，这是多进程带来的主要影响。

一般来说，使用多进程会造成如下账方面的问题：

1. 静态成员和单例模式失效
2. 线程同步机制完全失效
3. SharedPreference的可可靠性下降
4. Application会多次创建

第4个问题，当一个组件跑在一个新的进程中的时候，由于系统要在创建新的进程中同时分配独立的虚拟机，所以这个过程其实就是启动一个应用的过程。因此相当于系统又把这个应用重新启动了一遍，既然重新启动了，那么自然会创建新的Application。

## IPC 基础概念

主要包含三方面内容：Serializable接口和Parcelable接口以及Binder。Serializable和Parcelable接口可以完成对象的序列化过程，当我们需要通过Intent和Binder传输数据时就需要使用Parcelable或者Serializable，还有的时候我们需要对象持久化到存储设备上或者通过网络传输给其它客户端，这个时候也需要使用Serialzable完成对象的序列化。

### Serializable接口

Serialization接口是Java所提供一个序列化接口，它是一个空接口，为对象提供标准的序列化和反序列化操作。这里要注意serialVersionUID这个字段。

### Parcelable接口

Parcelable也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并可以通过Intent和Binder传递。系统已经为我们提供了许多实现了Parcelable接口的类，它们都是可以直接序列化的，比如Intent、Bundle、Bitmap等，同时List和Map也可以序列化，前提是它们里面的每个元素都是可序列化的。

既然Parcelable和Serializable都能实现序列化并且都可用于Intent间数据传递，那么二者该如何选取呢？Serializable是Java中的序列化接口，其使用起来简单但是开销比较大，序列化和反序列化过程需要大量的I/O操作。而Parcelable是Android中的序列化方式，因此更适合用在Android平台上，它的缺点就是使用起来稍微麻烦点，但它的效率比较高，这是Android推荐的序列化方式，因此应当首选Parcelable。

### Binder

直观来说，Binder是Android中的一个类，它实现了IBinder接口。从IPC的角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有。从Android Framework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager等）和相应ManagerService的桥梁。从Android应用层来说，Binder是客户端和服务端进行通信的媒体，当bindService的时候，服务端会返回一个包含了服务端业务调用Binder对象，通过个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通的服务和基于AIDL的服务。

Android开发中，Binder主要用在Service中，包括AIDL和Messenger，其中普通Service中的Binder不涉及进程间通信，Messenger的底层其实是AIDL。这里选择AIDL来分析Binder的工作机制。

Book.java

```java
package com.example.myapplication;

import android.os.Parcel;
import android.os.Parcelable;

public class Book implements Parcelable {

    private int bookId;
    private String bookName;

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    protected Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }
}


```

```java
// Book.aidl
package com.example.myapplication;

parcelable Book;
```

```java
// IBookManager.aidl
package com.example.myapplication;

import com.example.myapplication.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

Book.java是一个表示图书信息的类，它实现了Parcelable接口。Book.aidl是Book类在AIDL中的声明。IBookManager.aidl是我们定义的一个接口，里面有两个方法：getBookList和addBook，其中getBookList用于从远程服务端获取图书列表，addBook用于往图书列表中添加一本图书。

可以看到，尽管Book类和IBookManager位于相同的包中，但是在IBookManager中仍然要导入Book类，这就是AIDL的特殊之处。来看一下系统为IBookManager.aidl生产的Binder类：

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/heqingbao/Documents/Android/workspace/MyApplication/app/src/main/aidl/com/example/myapplication/IBookManager.aidl
 */
package com.example.myapplication;

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.myapplication.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.example.myapplication.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.myapplication.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.example.myapplication.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.myapplication.IBookManager))) {
                return ((com.example.myapplication.IBookManager) iin);
            }
            return new com.example.myapplication.IBookManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.example.myapplication.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.myapplication.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.myapplication.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.example.myapplication.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public java.util.List<com.example.myapplication.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.myapplication.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.myapplication.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.example.myapplication.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public java.util.List<com.example.myapplication.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.example.myapplication.Book book) throws android.os.RemoteException;
}

```

上述代码是系统生成的。它继承了IInterface接口，同时它自己也还是个接口，所有可以在Binder中传输的接口都需要继承IInterface接口。

首先，它声明了两个方法getBookList和addBook，显然这就是我们在IBookManager.aidl中声明的方法，同时在内部类Stub里面还声明了两个int类型的id分别用于标识这两个方法，这两个id用于标识在transact过程中客户端所请求的到底是哪个方法。接着，它声明了一个内部类Stub，这个Stub就是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位不同进程时，方法调用需要走transact过程，这个逻辑由Stub内部代理类Proxy来完成。这么来看这个IBookManager这个接口的核心实现就是它的内部类Stub和Stub的内部代理类Proxy。

**DESCRIPTOR**

Binder的唯一标识，一般用当前Binder的类名表示。

**asInterface(android.os.IBinder obj)**

用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身。

**asBinder**

此方法返回当前的Binder对象。

**onTransact**

这个方法运行在服务端的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。该方法的原型为```public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)```。服务端通过code可以确定客户端请求的方法是什么，接着从data中取出目标方法所需的参数(如果目标方法有参数的话)，然后执行目标方法。当目标方法执行完毕后，就向reply中写入返回值，如果此方法返回false，那么客户端的请求会失败，因此我们可以利用这个特性来做权限验证，毕竟我们也没希望随便一个远程都能远程调用我们的服务。

**Proxy#getBookList**

这个方法运行在客户端，当客户端远程调用此方法时，它的内部实现是这样的：首先创建该方法所需要的输入类型Parcel对象_data、输出型Parcel对象_reply和返回值对象List _result，然后把该方法的参数信息写入_data中(如果有参数的话)，接着调用transact方法来发起RPC（远程方法调用）请求，同时当前线程挂起，然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果，最后返回_reply中的数据。

**Proxy#addBook**

这个方法运行在客户端，它的执行过程和getBookList是一样的，addBook没有返回值，所以它不需要从_reply中取出返回值。

通过上面的分析，应该已经了解了Binder的工作机制，但是有两点还是需要说明：首先，当客户端发起远程请求时，由于当前线程会被挂起起码至服务端进程返回数据，所以如果一个远程方法是很耗时的，那么不能在UI线程中发起此远程请求；其次，由于服务端的Binder方法运行在Binder线程池中，所以Binder方法不管是否耗时都该采用同步的方法去实现，因为它已经运行在一个线程池中了。

## Android中的IPC方式

### 使用Bundle

四大组件中的三大组件（Activity、Service、BroadcastReceiver）都支持在Intent中传递Bundle数据，由于Bundle实现了Parcelable接口，所以它可以方便在不同进程间传输。基于这一点，当我们在一个进程中启动了另一个进程的Activity、Service和BroadcastReceiver，就可以在Bundle中附加需要传输给远程进程的信息并通过Intent发送出去。

### 使用文件共享

文件共享也是一种不错的进程间通信方式，两个进程通过读写同一个文件来交换数据。除了可以交换文本数据外，还可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象。但是这种方式是有局限性的，比如并发读写的问题。

### 使用Messenger

Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。

```java
public final class Messenger implements Parcelable {

    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }

     /**
     * Create a Messenger from a raw IBinder, which had previously been
     * retrieved with {@link #getBinder}.
     * 
     * @param target The IBinder this Messenger should communicate with.
     */
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
}
```

Messenger的使用方法很简单，它对AIDL做了封装，使得我们可以更简便进行进程间通信。同时，由于它一次处理一请求，因此在服务端我们不用考虑线程同步的问题。

### 使用AIDL

前面通过Messenger可以发现它是以串行的方式处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适了。同时，Messenger的作用主要是为了传递消息，很多时候我们可能需要跨进程调用服务端的方法，这种情形Messenger就无法做到了。但我们还可以使用AIDL来实现跨进程方法调用。

**AIDL所支持的数据类型**

* 基本数据类型（int、long、char、boolean、double等）
* String和CharSequence
* List：只支持ArrayList，里面每个元素都必须能够被AIDL支持
* Map：只支持HashMap，里面每个元素都必须能够被AIDL支持
* Parcelable：所有实现了Parcelable接口的对象
* AIDL：所有的AIDL接口本身也可以在AIDL文件中使用

特别说明，自定义的Parcelable对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件在同个package下。

另一个需要注意的，如果AIDL文件中用到了自定久的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并在其中声明它为parcelable类型。除此之外，AIDL中除了基本数据类型，其它类型的参数必须标上方向：in、out或者inout，in表示输入型参数，out表示输出型参数，inout表示输入输出型参数。最后，AIDL接口中只支持方法，不支持声明静态常量。

**如何在AIDL中使用权限验证**

* 在onBind中进行验证，难不通过就直接返回null。验证的方式可以有多种，比如常用的permission。
* 在服务端的onTransact方法中进行权限验证，如果验证失败，直接返回false。这样服务端就会终止执行AIDL中的方法从而达到保护服务端的上的。

除了这两种，肯定还有其它方法，比如为Service指定`android:permisstion`属性等。

### 使用ContentProvider

ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，从这一点来看，它天生就适合进行间通信。

### 使用Socket

Socket也称为`套接字`，是网络通信中的概念，它分为流式套接字和用户数据报套接字两种，分别对应于网络的传输控制层中的TCP和UDP协议。TCP协议是面向连接的协议，提供稳定的双向通信功能，TCP的建立需要经过`三次握手`才能完成，为了提供稳定的数据传输功能，其本身提供了超时重传机制，因此具有很高的稳定性；UDP是无连接的，提供不稳定的单向通信功能，当然UDP也可以实现双向通信功能。在性能上，UDP具有更好的效率，其缺点是不保证数据能够正常传输，尤其在网络拥塞的情况下。