title: 三个线程，顺序打印ABC
date: 2016-02-29 13:32:08
categories: Java
tags: 
- Java
- Thread
comments: true
---

## 三个线程，顺序打印ABC

输出：
```java
A
B
C
A
B
C
A
B
C
A
B
C
A
B
C
A
B
C
A
B
C
A
B
C
A
B
C
A
B
C
```

### 使用wait()和notify()

```java
public class ABC_1 {

    private int status = 0;

    public static void main(String[] args) {
        new ABC_1().setup();
    }

    private void setup() {
        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printA();
                }
            }
        });

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printB();
                }
            }
        });

        Thread t3 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printC();
                }
            }
        });

        t1.start();
        t2.start();
        t3.start();
    }

    private synchronized void printA() {
        while (status % 3 != 0) {
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("A");
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        status++;
        this.notifyAll();
    }

    private synchronized void printB() {
        while (status % 3 != 1) {
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("B");
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        status++;
        this.notifyAll();
    }

    private synchronized void printC() {
        while (status % 3 != 2) {
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("C");
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        status++;
        this.notifyAll();
    }
}
```

<!-- more -->

### 使用Lock和Condition

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ABC_2 {

    private Lock lock = new ReentrantLock();

    private Condition aCondition = lock.newCondition();
    private Condition bCondition = lock.newCondition();
    private Condition cCondition = lock.newCondition();

    private int status = 0;

    public static void main(String[] args) {
        new ABC_2().setup();
    }

    private void setup() {
        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printA();
                }
            }
        });

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printB();
                }
            }
        });

        Thread t3 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printC();
                }
            }
        });

        t1.start();
        t2.start();
        t3.start();
    }

    private void printA() {
        lock.lock();
        try {
            while (status % 3 != 0) {
                try {
                    aCondition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("A");
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            status++;
            bCondition.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    private void printB() {
        lock.lock();
        try {
            while (status % 3 != 1) {
                try {
                    bCondition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("B");
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            status++;
            cCondition.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    private void printC() {
        lock.lock();
        try {
            while (status % 3 != 2) {
                try {
                    cCondition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("C");
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            status++;
            aCondition.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

### 使用Semaphore

```java
import java.util.concurrent.Semaphore;

public class ABC_3 {

    private Semaphore aSemaphore = new Semaphore(1);
    private Semaphore bSemaphore = new Semaphore(0);
    private Semaphore cSemaphore = new Semaphore(0);

    public static void main(String[] args) {
        new ABC_3().setup();
    }

    private void setup() {
        Thread t1 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printA();
                }
            }
        });

        Thread t2 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printB();
                }
            }
        });

        Thread t3 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    printC();
                }
            }
        });

        t1.start();
        t2.start();
        t3.start();
    }

    private void printA() {
        try {
            aSemaphore.acquire();

            System.out.println("A");
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            bSemaphore.release();
        } catch (Exception e) {
            e.printStackTrace();
            aSemaphore.release();
        }
    }

    private void printB() {
        try {
            bSemaphore.acquire();

            System.out.println("B");
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            cSemaphore.release();
        } catch (Exception e) {
            e.printStackTrace();
            bSemaphore.release();
        }
    }

    private void printC() {
        try {
            cSemaphore.acquire();

            System.out.println("C");
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            aSemaphore.release();
        } catch (Exception e) {
            e.printStackTrace();
            cSemaphore.release();
        }
    }
}
```
