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