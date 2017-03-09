title: Integer类型自动装箱的疑惑
date: 2017-03-09 13:40:35
tags: 
- Integer
- 自动装箱
categories: Java
comments: true
reward: true
---

先看下面这段代码：

```java
public class Demo {
    public static void main(String args[]) {
        Integer a = 1000;
        Integer b = 1000;
        System.out.println(a == b);
    }
}
```
到此你可能会想，对Object（包装）类型的对象应用`==`操作，比较的是对象的内存地址（是否是同一个对象）。这里是两个Object类型的对象，所以返回`false`！没错，答对了。

那么再看下面这段代码：

```java
public class Demo {
    public static void main(String args[]) {
        Integer a = 100;
        Integer b = 100;
        System.out.println(a == b);
    }
}
```

如果你还认为它也返回`false`的话，你就需要往下看了。。。因为它实际上会返回`true`！

那么为什么会有这样的差异呢？上面两段代码并不复杂，所以不会存在逻辑陷阱，那有什么办法可以明确知道是什么原因呢？

说到这里，肯定有人会想到去看`Integer`的源码，没错，我们在`Integer`的源码中发现了这段代码：

```java
public static Integer valueOf(int i) {
	final int offset = 128;
	if (i >= -128 && i <= 127) { // must cache
		return IntegerCache.cache[i + offset];
	}
	return new Integer(i);
}
```

从这个方法可以看出，如果参数`i`在`[-128~127]`这个范围内的话，直接从IntegerCache里面取，否则就创建一个新的Integer对象。这个IntegerCache从命名上来看，有点像是`整数缓存`的意思，它是Integer的一个静态内部类：

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

由此可以看出，调用`valueOf(i)`创建Integer话，如果`i`的范围在`[-128~127]`里面，则创建的对象是同一个引用。这跟我们上面两段代码的结果有点像。到此，你可能会想，要不试试？

经过试验，你会发现当`Integer a = 127, b = 127`的时候：`a == b`，当超过127的时候：`a != b`。

经过试验，你可能会暗算高兴，不就是这个原因嘛，源码里面写着呢。。。。

但是，有什么能够证明`Integer a = 100`就是调用`valueOf`这个方法实现的呢？

到此，本文的主角该出场了，也是关键点。

我们可以反编译Java的字节码文件，通过查看编译器编译后的字节命令，我们能够更清楚地了解Java在处理一些代码操作的机制。

我们这里需要用到`javap`命令，这个命令包含在JDK里面。

```
➜ /Users/heqingbao/Desktop javap -verbose Demo
Classfile /Users/heqingbao/Desktop/Demo.class
  Last modified 2017-3-9; size 569 bytes
  MD5 checksum 337e4357e76e91e8e3f525fc9a95adb6
  Compiled from "Demo.java"
public class Demo
  SourceFile: "Demo.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#19         //  java/lang/Object."<init>":()V
   #2 = Methodref          #20.#21        //  java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #3 = Fieldref           #22.#23        //  java/lang/System.out:Ljava/io/PrintStream;
   #4 = Methodref          #24.#25        //  java/io/PrintStream.println:(Z)V
   #5 = Class              #26            //  Demo
   #6 = Class              #27            //  java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               StackMapTable
  #14 = Class              #28            //  "[Ljava/lang/String;"
  #15 = Class              #29            //  java/lang/Integer
  #16 = Class              #30            //  java/io/PrintStream
  #17 = Utf8               SourceFile
  #18 = Utf8               Demo.java
  #19 = NameAndType        #7:#8          //  "<init>":()V
  #20 = Class              #29            //  java/lang/Integer
  #21 = NameAndType        #31:#32        //  valueOf:(I)Ljava/lang/Integer;
  #22 = Class              #33            //  java/lang/System
  #23 = NameAndType        #34:#35        //  out:Ljava/io/PrintStream;
  #24 = Class              #30            //  java/io/PrintStream
  #25 = NameAndType        #36:#37        //  println:(Z)V
  #26 = Utf8               Demo
  #27 = Utf8               java/lang/Object
  #28 = Utf8               [Ljava/lang/String;
  #29 = Utf8               java/lang/Integer
  #30 = Utf8               java/io/PrintStream
  #31 = Utf8               valueOf
  #32 = Utf8               (I)Ljava/lang/Integer;
  #33 = Utf8               java/lang/System
  #34 = Utf8               out
  #35 = Utf8               Ljava/io/PrintStream;
  #36 = Utf8               println
  #37 = Utf8               (Z)V
{
  public Demo();
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0

  public static void main(java.lang.String[]);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=3, args_size=1
         0: bipush        100
         2: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
         5: astore_1
         6: bipush        100
         8: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        11: astore_2
        12: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: aload_1
        16: aload_2
        17: if_acmpne     24
        20: iconst_1
        21: goto          25
        24: iconst_0
        25: invokevirtual #4                  // Method java/io/PrintStream.println:(Z)V
        28: return
      LineNumberTable:
        line 4: 0
        line 5: 6
        line 6: 12
        line 7: 28
      StackMapTable: number_of_entries = 2
           frame_type = 255 /* full_frame */
          offset_delta = 24
          locals = [ class "[Ljava/lang/String;", class java/lang/Integer, class java/lang/Integer ]
          stack = [ class java/io/PrintStream ]
           frame_type = 255 /* full_frame */
          offset_delta = 0
          locals = [ class "[Ljava/lang/String;", class java/lang/Integer, class java/lang/Integer ]
          stack = [ class java/io/PrintStream, int ]

}
```

我猜你已经看到了`Integer.valueOf`了。