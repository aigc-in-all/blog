title: Be Careful With Transient Data
date: 2016-02-29 17:06:03
categories: Java
tags: transient
comments: true
---

> 原文：http://www.devx.com/tips/Tip/13726
> 作者：Behrouz Fallahi


Java序列化提供一种优雅的和易于使用的持久化对象实例的机制。当持久化对象时，可能有一个特殊的对象数据成员，我们不想用serialization机制来保存它。为了在一个特定对象的一个域上关闭serialization，可以在这个域前加上关键字transient。

transient是Java语言的关键字，用来表示一个域不是该对象串行化的一部分。当一个对象被串行化的时候，transient型变量的值不包括在串行化的表示中，然而非transient型的变量是被包括进去的。

看一下例子，假设我们定义一个类：

```java
public class LoggingInfo implements Serializable {

    private static final long serialVersionUID = 7702603582718186313L;

    private Date loggingDate = new Date();
    private String uid;
    private transient String pwd;

    LoggingInfo(String user, String password) {
        uid = user;
        pwd = password;
    }

    @Override
    public String toString() {
        String password = null;
        if (pwd == null) {
            password = "NOT SET";
        } else {
            password = pwd;
        }
        return "logon info: \n   " + "user: " + uid + 
        	"\n   logging date : " + loggingDate.toString() + 
        	"\n   password: " + password;
    }
}
```

现在我们可以创建这个类的实例，并且序列化它，最后把序列化的对象写入到磁盘：

```java
LoggingInfo logInfo = new LoggingInfo("MIKE", "MECHANICS"); 
System.out.println(logInfo.toString()); 
try 
{ 
   ObjectOutputStream o = new ObjectOutputStream( 
                new FileOutputStream("logInfo.out")); 
   o.writeObject(logInfo); 
   o.close(); 
} 
catch(Exception e) {//deal with exception} 
```

然后读取这个对象：

```java
try 
{ 
   ObjectInputStream in =new ObjectInputStream( 
                new FileInputStream("logInfo.out")); 
   LoggingInfo logInfo = (LoggingInfo)in.readObject(); 
   System.out.println(logInfo.toString()); 
} 
catch(Exception e) {//deal with exception} 

```

完整的例子：

```java
public class Client {

    public static void main(String[] args) {
        LoggingInfo logInfo = new LoggingInfo("MIKE", "MECHANICS");
        System.out.println(logInfo.toString());

        ObjectOutputStream out = null;
        try {
            out = new ObjectOutputStream(new FileOutputStream("logInfo.out"));
            out.writeObject(logInfo);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (out != null) {
                try {
                    out.close();
                } catch (IOException ignore) {
                }
            }
        }

        ObjectInputStream in = null;
        try {
            in = new ObjectInputStream(new FileInputStream("logInfo.out"));
            LoggingInfo info = (LoggingInfo) in.readObject();
            System.out.println(info);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出：

```java
logon info: 
   user: MIKE
   logging date : Mon Feb 29 17:14:34 CST 2016
   password: MECHANICS
logon info: 
   user: MIKE
   logging date : Mon Feb 29 17:14:34 CST 2016
   password: NOT SET
```

我们会注意到从磁盘中读回(read--back(de-serializing))的对象打印password为`NOT SET`。这是当我们定义pwd域为transient时，所期望的正确结果。

现在，让我们来看一下粗心对待transient域可能引起的潜在问题。假设我们修改了类定义，提供给transient域一个默认值：

```java
public class GuestLoggingInfo implements Serializable {

    private static final long serialVersionUID = 7702603582718186313L;

    private Date loggingDate = new Date();
    private String uid;
    private transient String pwd;

    GuestLoggingInfo() {
        uid = "guest";
        pwd = "guest";
    }

    @Override
    public String toString() {
        // 跟上面一样
    }
}
```

现在如果我们串行化GuestLoggingInfo的一个实例，将它写入磁盘，并且再将它从磁盘中读出，我们仍然看到读回的对象打印password为`NOT SET`。当从磁盘中读出某个类的实例时，实际上并不会执行这个类的构造函数，而是载入了一个该类对象的持久化状态，并将这个状态赋值给该类的另一个对象。

<!-- more -->

--------------------------

### 补充

**结论1：静态变量不管是否被transient修饰，均不能被序列化。**

还是上面的例子，只是给uid声明成static类型。

```java
public class LoggingInfo implements Serializable {

    private static final long serialVersionUID = 7702603582718186313L;

    private Date loggingDate = new Date();
    private static String uid;
    private transient String pwd;

    LoggingInfo(String user, String password) {
        uid = user;
        pwd = password;
    }

    public void setUid(String uid) {
        this.uid = uid;
    }

    @Override
    public String toString() {
        // 跟上面一样
    }
}
```

Client不动，直接run后结果：

```java
logon info: 
   user: MIKE
   logging date : Mon Feb 29 17:39:42 CST 2016
   password: MECHANICS
logon info: 
   user: MIKE
   logging date : Mon Feb 29 17:39:42 CST 2016
   password: NOT SET

```

我们发现在uid字段前面加上static关键字后，**程序运行结果依然不变**，即static类型的uid也读出来为`MIKE`了，这不与上面的结论矛盾吗？

实际上是这样的：上面的结论确实没错，反序列化后类中static变量uid的值为当前JVM中对应static变量的值，这个值是JVM的不是反序列化得出的。不相信？好吧，下面我来证明：

在Client类里面加上一行
```java
public class Client {

    public static void main(String[] args) {
        LoggingInfo logInfo = new LoggingInfo("MIKE", "MECHANICS");
        System.out.println(logInfo.toString());

        // 序列化到磁盘，跟上面一样

        logInfo.setUid("TOM");

        // 从磁盘中读取，跟上面一样
    }
}
```

输出：
```java
logon info: 
   user: MIKE
   logging date : Mon Feb 29 17:45:53 CST 2016
   password: MECHANICS
logon info: 
   user: TOM
   logging date : Mon Feb 29 17:45:53 CST 2016
   password: NOT SET

```

从结果中可以看到，user的值为修改后的 `TOM`，而不是序列化时的`MIKE`。

**结论2：被transient关键字修改的变量真的不能被序列化吗？**

```java
public class ExternalizableTest implements Externalizable {

    private transient String content = "我会被序列化";

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeObject(content);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        content = (String) in.readObject();
    }

    public static void main(String[] args) throws Exception {
        ExternalizableTest et = new ExternalizableTest();
        ObjectOutput out = new ObjectOutputStream(new FileOutputStream(new File("test")));
        out.writeObject(et);

        ObjectInput in = new ObjectInputStream(new FileInputStream(new File("test")));
        et = (ExternalizableTest) in.readObject();
        System.out.println(et.content);

        out.close();
        in.close();
    }
}
```

运行的结果是：
```java
我会被序列化
```

这是为什么呢，不是说类的变量被transient关键字修饰以后将不能序列化了吗？

我们知道在Java中，对象的序列化可以通过实现两种接口来实现，若实现的是Serializable接口，则所有的序列化将会自动进行，若实现的是Externalizable接口，则没有任何东西可以自动序列化，需要在writeExternal方法中进行手工指定所要序列化的变量，这与是否被transient修饰无关。因此第二个例子输出的是变量content初始化的内容，而不是null。