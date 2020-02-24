---
title: Java创建线程的方式
comments: true
date: 2020-02-21
tags:
	- Java
	- Thread
	- ThreadPool
categories:	
	- 学习笔记
---

尽管提倡使用线程池高效地管理线程，但了解线程创建的基础方法还是非常有必要的。

<!--more-->

### 一、前言

创建线程的四种方式：

- 继承Thread类，无返回结果
- 实现Runnable接口，无返回结果
- 通过FutureTask类，有返回结果
- 通过线程池创建



### 二、继承Thread类创建线程类

> Java的单继承机制，使得类继承Thread类后，不能再继承其它类。

##### 1. 具体实现过程如下：

1. 继承Thread类，重写**run( )**方法。run( )方法表示线程要执行的任务，被称为执行体。
2. 创建Thread子类对象实例，即创建线程对象。
3. 调用线程对象的**start( )**方法启动该线程。



##### 2. 实现代码

```java
// 继承Thread类，重写run( )方法
public class MyThread extends Thread {
	@Override
	public void run() {
		System.out.println("Java创建线程的方式之一：继承Thread类，重写run方法。");
	}
}
```

```java
public static void main(String[] args){
   MyThread myThread = new MyThread(); // 实例化线程对象
   myThread.start(); // 启动线程
}
```



##### 3. 匿名内部类形式

*上述还可以写成匿名内部类的形式*

```java
public static void main(String[] args) {
    Thread thread = new Thread() {
        @Override
        public void run() {
            System.out.println("启动线程");
        }
    };
    thread.start();
}
```



### 二、通过Runnable接口创建线程类

> 在多数情况下，如果仅覆盖run( )方法而不涉及Thread类的其它方法，不建议使用第一种途径新建线程。

##### 1. 具体实现过程如下：

1. 定义Runnable接口的实现类，重写run( )方法。该实现类的实例作为**任务对象**，run( )方法为执行体。
2. 传入任务对象，构造Thread类实例对象，即真正的线程对象。
3. 调用线程对象的start( )方法启动该线程。



##### 2. 实现代码

```java
// RunnableTask类——任务类
public class RunnableTask implements Runnable {
	@Override
	public void run() {
		System.out.println("Java创建线程的方式之二：实现Runnable接口，重写run方法。");
	}
}
```

```java
public static void main(String[] args) {
    RunnableTask runTask = new RunnableTask(); // 实例化任务类
    Thread thread = new Thread(runTask); // 传入任务对象，创建线程对象
    thread.start();
}
```



##### 3. 匿名内部类形式

```java
public static void main(String[] args) {
    Thread thread = new Thread(new Runnable() {
        @Override
        public void run() {
            System.out.println("启动线程");
        }
    });
    thread.start();
}
```



### 三、通过FutureTask类创建有返回结果的线程

> FutureTask实现 `Future` 和 `Runnable` 接口，是一个可取消的**异步计算任务**，通过其可跟踪任务的执行情况，获取任务的执行结果。

FutureTask类的两个构造方法

```java
FutureTask(Callable<V> callable); // 执行给定的Callable对象
FutureTask(Runnable runnable, V result); // 执行给定的Runnable对象，成功后返回给定的result
```



##### 1. 具体实现过程如下：

1. 创建Callable接口的实现类，重写call( )方法。call( )方法为执行体，**有返回值**。或者，创建Runnable接口的实现类，重写run( )方法。run( )方法为执行体，无返回值。
2. 调用FutureTask类的构造方法创建FutureTask对象，其中对Runnable对象需要指定result。
3. 传入FutureTask对象，构造Thread类实例对象，即真正的线程对象。
4. 调用线程对象的start( )方法启动该线程。
5. 调用FutureTask对象的**get( )**方法获取线程执行结束的返回值。



##### 2. 实现代码

1. *对于 Callable对象*

```java
// 创建Callable接口的实现类，重写call( )方法
public class CallableTask implements Callable<Double> {
	@Override
	public Double call() { // call( )方法有返回值
		double number = 0;
		try {
			System.out.println("异步计算任务开始");
			number = Math.random()*10;
			number += 500;
		}catch (Exception e) {
			e.printStackTrace();
		}
		return number;
	}
}
```

```java
public static void main(String[] args) throws Exception {
    FutureTask<Double> futureTask = new FutureTask<>(new CallableTask()); // 传入Callable对象，创建FutureTask对象
    Thread thread = new Thread(futureTask); // 传入FutureTask对象，创建线程对象
    thread.start(); // 启动线程
    Double result = futureTask.get(); // 通过FutureTask类的get()方法获取异步计算结果
    System.out.println("计算结果为："+result);
}
```

2. *对于 Runnable 对象*

```java
// 创建Runnable接口的实现类，重写run( )方法
public class RunnableTask implements Runnable {
	@Override
	public void run() {
		System.out.println("开启线程");
	}
}
```

```java
public static void main(String[] args) throws Exception {
    FutureTask<Double> futureTask = new FutureTask<Double>(new RunnableTask(), 520.0);
    Thread thread = new Thread(futureTask);
    thread.start();
    Double result = futureTask.get();
    System.out.println(result);
}
```



### 四、通过线程池管理线程

> 通过工厂类 Executors 或 ThreadPoolExecutor 类可创建线程池，通过线程池可管理线程。

*以 Executors 为例*

（1）Executors 可生成四种不同的线程池

```java
public static ExecutorService newFixedThreadPool(int nThreads); // 固定长度
public static ExecutorService newCachedThreadPool(); // 可缓存
public static ExecutorService newSingleThreadExecutor(); // 单线程
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize); // 延时/定时执行任务，固定长度
```

（2）演示

```java
// 实现Runnable接口
class RunnableTask implements Runnable {
	@Override
	public void run() {
		System.out.println("实现Runnable接口，重写run方法，无返回值");
	}
}
```

```java
public static void main(String[] args) throws Exception {
    ExecutorService es = Executors.newSingleThreadExecutor();
    es.execute(new RunnableTask());
}
```



### 参考资料

[1] https://www.cnblogs.com/jinggod/p/8485106.html