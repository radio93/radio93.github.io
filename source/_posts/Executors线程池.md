---
title: Executors线程池
date: 2020-10-28 14:21:17
tags:
- java
---

##### newCachedThreadPool

创建一个可缓存的线程池

> 如果线程池的大小超过了处理任务所需要的线程，那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。

示例：

```java
public static ExecutorService executor = Executors.newCachedThreadPool();
executor.submit(new Runnable() {
	public void run () {
		//TODO
	}
});
```

##### newFixedThreadPool

创建固定大小的线程池

>每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

示例：

```java
package test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class FixedThreadPoolDemo extends Thread {
    private int index;

    public FixedThreadPoolDemo(int index) {
        this.index = index;
    }

    public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(2);
        for (int i = 0; i < 5; i++) {
            service.execute(new FixedThreadPoolDemo(i));
        }
        System.out.println("finish");
        service.shutdown();
    }

    public void run() {
        try {
            System.out.println(Thread.currentThread().getName());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

运行结果：

```java
finish
pool-1-thread-2
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
pool-1-thread-2
```

##### newSingleThreadExecutor

创建一个单线程的线程池

>   这个线程池只有一个线程在工作，也就是相当于单线程串行执行所有任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

示例：

```java
package test;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SingleThreadPoolDemo extends Thread {
    private int index;

    public SingleThreadPoolDemo(int index) {
        this.index = index;
    }

    public static void main(String[] args) {
        ExecutorService service = Executors.newSingleThreadExecutor();
        for (int i = 0; i < 5; i++) {
            service.execute(new SingleThreadPoolDemo(i));
        }
        System.out.println("finish");
        service.shutdown();
    }

    public void run() {
        try {
            System.out.println(Thread.currentThread().getName());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

运行结果：

```java
finish
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
```

##### newScheduledThreadPool

创建一个周期任务线程池

>此线程池支持定时以及周期性执行任务的需求

示例：

```java
package test;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledThreadPoolDemo {
    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newScheduledThreadPool(1);
        service.schedule(() -> System.out.println(Thread.currentThread().getName()), 2, TimeUnit.SECONDS);
    }

}
```

运行结果(该程序表示延迟2s执行)：

```
pool-1-thread-1
```

##### scheduleAtFixedRate

周期线程中的定时任务

示例：

```java
package test;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledThreadPoolDemo {
    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newScheduledThreadPool(1);
        service.scheduleAtFixedRate(() -> System.out.println(Thread.currentThread().getName()), 5, 2,TimeUnit.SECONDS);
    }

}
```

运行结果(该程序表示程序启动5s后，每隔2s执行一次)：

```java
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
.
.
.
```

##### scheduleWithFixedDelay

周期线程中的定时任务

示例：

```java
package test;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledThreadPoolDemo {
    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newScheduledThreadPool(1);
        service.scheduleWithFixedDelay(() -> System.out.println(Thread.currentThread().getName()), 5, 2,TimeUnit.SECONDS);
    }

}
```

运行结果

```java
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
.
.
.
```

> 说明：scheduleAtFixedRate和scheduleWithFixedDelay的区别在于前者是时间间隔过后，再检查任务是否结束，如果结束了，立即执行下个任务，后者是先等待任务结束，然后再等待时间间隔过后再执行。

##### newSingleThreadScheduledExecutor

定时任务

示例：

```java
package test;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class SingleThreadScheduledPoolDemo {
    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor();
        service.scheduleAtFixedRate(() -> System.out.println(Thread.currentThread().getName()), 5, 2,TimeUnit.SECONDS);
    }

}
```

运行结果(该程序表示程序启动5s后，每隔2s执行一次)：

```java
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
.
.
.
```

> ScheduledExecutorService执行的周期任务，如果执行过程中抛出了异常，那么任务就会停止，周期也会停止。