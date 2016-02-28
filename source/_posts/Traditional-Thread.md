title: 传统线程机制的回顾
date: 2016-02-29 00:02:13
tags: Java Thead
---

## 传统线程机制的回顾

### 创建线程的两种传统方式

* 在Thread子类覆盖的run方法中编写运行代码
* 在传递给Thread对象的Runnable对象的run方法中编写代码

总结：查看Thread类的run方法的源代码可以看到，其实这两种方式都是在调用Thread对象的run方法，如果Thread类的run方法没有被覆盖，并且为该Thread对象设置了Runnable对象，该run方法会调用Runnable的run方法。

问题：如果在Thread子类覆盖的run方法中编写了运行代码，也为Thread子类对象传递了一个Runnable对象，那么，线程运行时的执行代码是子类的run方法还是Runnable的run方法？

```java
public class Thread implements Runnable {
	/* What will be run. */
    private Runnable target;

    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }

    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
        init(g, target, name, stackSize, null);
    }

    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        ...
        this.target = target;
        ...
    }

    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
}
```

### 定时器的应用
* Time类
* TimeTask类

```java
    /**
     * Schedules the specified task for execution after the specified delay.
     */
    public void schedule(TimerTask task, long delay) {...}

    /**
     * Schedules the specified task for execution at the specified time.  If
     * the time is in the past, the task is scheduled for immediate execution.
     */
    public void schedule(TimerTask task, Date time) {...}

    /**
     * Schedules the specified task for repeated <i>fixed-delay execution</i>,
     * beginning after the specified delay.  Subsequent executions take place
     * at approximately regular intervals separated by the specified period.
     */
    public void schedule(TimerTask task, long delay, long period) {...}

    /**
     * Schedules the specified task for repeated <i>fixed-delay execution</i>,
     * beginning at the specified time. Subsequent executions take place at
     * approximately regular intervals, separated by the specified period.
     */
    public void schedule(TimerTask task, Date firstTime, long period) {...}

    /**
     * Schedules the specified task for repeated <i>fixed-rate execution</i>,
     * beginning after the specified delay.  Subsequent executions take place
     * at approximately regular intervals, separated by the specified period.
     */
    public void scheduleAtFixedRate(TimerTask task, long delay, long period) {...}

    /**
     * Schedules the specified task for repeated <i>fixed-rate execution</i>,
     * beginning at the specified time. Subsequent executions take place at
     * approximately regular intervals, separated by the specified period.
     */
    public void scheduleAtFixedRate(TimerTask task, Date firstTime, long period) {...}
```

## 线程的互斥与同步

* 使用synchronized代码块及其原理
* 使用synchronized方法(this)
* 分析静态方法所使用的同步监视器是什么(class)
```java
public class TraditionalThreadSynchronized {

	public static void main(String[] args) {
		new TraditionalThreadSynchronized().init();
	}

	private void init() {
		final Outputer outputer = new Outputer();
		new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					try {
						Thread.sleep(10);
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
					outputer.output("hello");
				}
			}
		}).start();

		new Thread(new Runnable() {
			@Override
			public void run() {
				while (true) {
					try {
						Thread.sleep(10);
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
					outputer.output3("world");
				}
			}
		}).start();
	}

	static class Outputer {
		public void output(String name) {
			int len = name.length();
			synchronized (Outputer.class) {
				for (int i = 0; i < len; i++) {
					System.out.print(name.charAt(i));
				}
				System.out.println();
			}
		}

		public synchronized void output2(String name) {
			int len = name.length();
			for (int i = 0; i < len; i++) {
				System.out.print(name.charAt(i));
			}
			System.out.println();
		}

		public static synchronized void output3(String name) {
			int len = name.length();
			for (int i = 0; i < len; i++) {
				System.out.print(name.charAt(i));
			}
			System.out.println();
		}
	}
}
```

### wait与notify实现线程间的通信

子线程循环10次，接着主线程循环100次，接着又回到子线程循环10次，接着再回到主线程循环100次，如此循环50次。
```java
public class TraditionalThreadCommunication {

	public static void main(String[] args) {

		final Business business = new Business();
		new Thread(new Runnable() {

			@Override
			public void run() {

				for (int i = 1; i <= 50; i++) {
					business.sub(i);
				}

			}
		}).start();

		for (int i = 1; i <= 50; i++) {
			business.main(i);
		}

	}

}

class Business {
	private boolean bShouldSub = true;

	public synchronized void sub(int i) {
		while (!bShouldSub) {
			try {
				this.wait();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		for (int j = 1; j <= 10; j++) {
			System.out.println("sub thread sequence of " + j + ",loop of " + i);
		}
		bShouldSub = false;
		this.notify();
	}

	public synchronized void main(int i) {
		while (bShouldSub) {
			try {
				this.wait();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		for (int j = 1; j <= 100; j++) {
			System.out.println("main thread sequence of " + j + ",loop of " + i);
		}
		bShouldSub = true;
		this.notify();
	}
}
```
