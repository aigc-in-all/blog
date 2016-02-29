title: ReadWriteLock
date: 2016-02-29 13:03:28
categories: Java
tags: Java Thread
comments: true
---

java.util.concurrent.locks.ReadWriteLock 是一个高级的线程锁机制，它允许多个线程来读取一个特定的资源，但在同一时间，只有能一个线程执行写操作。
一个ReadWriteLock中保持一对相关的锁，一个用于只读操作，另一个用于写操作。在没有写操作的情况下，读锁(read lock)可以同时被多个线程执行。写锁(write lock)是排它性的。在成功获取读锁后所读到的数据都是最新的。

* 读锁可以允许多个进行读操作的线程同时进入，但不允许写进程进入
* 写锁只允许一个写进程进入，在这期间任何线程都不能再进入


```java
public class ReadWriteLockTest {
    public static void main(String[] args) {
        final Queue3 q3 = new Queue3();
        for (int i = 0; i < 3; i++) {
            new Thread("Read Thread-" + i) {
                @Override
                public void run() {
                    while (true) {
                        q3.get();
                    }
                }

            }.start();

            new Thread("Write Thread-" + i) {
                @Override
                public void run() {
                    while (true) {
                        q3.put(new Random().nextInt(10000));
                    }
                }

            }.start();
        }
    }

    private static class Queue3 {
        private Object data = null;
        ReadWriteLock rwl = new ReentrantReadWriteLock();

        public void get() {
            rwl.readLock().lock();
            try {
                System.out.println(Thread.currentThread().getName() + " is ready to read data");
                Thread.sleep((long) (Math.random() * 1000));
                System.out.println(Thread.currentThread().getName() + " has been read data : " + data);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                rwl.readLock().unlock();
            }
        }

        public void put(Object data) {
            rwl.writeLock().lock();
            try {
                System.out.println(Thread.currentThread().getName() + " is ready to write data");
                Thread.sleep((long) (Math.random() * 1000));
                this.data = data;
                System.out.println(Thread.currentThread().getName() + " has been write data : " + data);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                rwl.writeLock().unlock();
            }
        }
    }
}
```
输出：
```java
Read Thread-0 is ready to read data
Read Thread-1 is ready to read data
Read Thread-2 is ready to read data
Read Thread-2 has been read data : null
Read Thread-1 has been read data : null
Read Thread-0 has been read data : null
Write Thread-0 is ready to write data
Write Thread-0 has been write data : 577
Write Thread-0 is ready to write data
Write Thread-0 has been write data : 8005
Write Thread-2 is ready to write data
Write Thread-2 has been write data : 8935
Write Thread-2 is ready to write data
Write Thread-2 has been write data : 0
Write Thread-2 is ready to write data
Write Thread-2 has been write data : 5878
Write Thread-1 is ready to write data
Write Thread-1 has been write data : 963
Write Thread-1 is ready to write data
Write Thread-1 has been write data : 4703
Write Thread-1 is ready to write data
Write Thread-1 has been write data : 7550
Read Thread-2 is ready to read data
Read Thread-1 is ready to read data
Read Thread-0 is ready to read data
Read Thread-1 has been read data : 7550
Read Thread-2 has been read data : 7550
Read Thread-0 has been read data : 7550
Write Thread-0 is ready to write data
Write Thread-0 has been write data : 1733
Write Thread-0 is ready to write data
Write Thread-0 has been write data : 6151
```
<!-- more -->

--------------------------

```java

class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();
     if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();
        rwl.writeLock().lock();
        try {
          // Recheck state because another thread might have
          // acquired write lock and changed state before we did.
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // Downgrade by acquiring read lock before releasing write lock
          rwl.readLock().lock();
        } finally {
          rwl.writeLock().unlock(); // Unlock write, still hold read
        }
     }

     try {
       use(data);
     } finally {
       rwl.readLock().unlock();
     }
   }
 }
```

```java
 class RWDictionary {
    private final Map<String, Data> m = new TreeMap<String, Data>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock r = rwl.readLock();
    private final Lock w = rwl.writeLock();

    public Data get(String key) {
        r.lock();
        try { return m.get(key); }
        finally { r.unlock(); }
    }
    public String[] allKeys() {
        r.lock();
        try { return m.keySet().toArray(); }
        finally { r.unlock(); }
    }
    public Data put(String key, Data value) {
        w.lock();
        try { return m.put(key, value); }
        finally { w.unlock(); }
    }
    public void clear() {
        w.lock();
        try { m.clear(); }
        finally { w.unlock(); }
    }
 }
```