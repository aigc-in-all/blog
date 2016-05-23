title: Difference between Overloading and Overriding in Java
date: 2016-03-01 12:29:57
categories: Java
tags: 
- Java 
- Overloading
- Overriding
comments: true
---

> 原文： http://java67.blogspot.jp/2012/09/difference-between-overloading-vs-overriding-in-java.html
> 作者： Java67

1. 第一个主要不同点是Overloading发生在编译时，而Overriding发生在运行时。

2. 第二个不同点是，overload可以发生在同一个类中，而override只能发生在子类里面。

3. 在Java里面，你可以overload静态方法，但是不能override静态方法。事实上当你在子类里面声明一个相同的方法，它只是隐藏而不是覆盖超类的方法。

4. Overloaded的方法结合使用静态绑定和使用类型的引用变量，而Overridden方法结合使用动态结合基于实际对象。（方法的重写Overriding和重载Overloading是Java多态性的不同表现。重写Overriding是父类与子类之间多态性的一种表现，重载Overloading是一个类中多态性的一种表现）

5. 在Java中Overloading和Overriding的规则是不同的。为了overload一个方法你需要改变方法的签名，但overriding是不需要的。

6. 另一个不同点是，私有(private)和final方法不能被overridden，但是可以overloaded。

7. 在Java里面，Overloaded方法比Overridden方法更快。

这是所有关于Java的方法重载和重写的区别。