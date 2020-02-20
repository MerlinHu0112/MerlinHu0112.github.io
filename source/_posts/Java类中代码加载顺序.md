---
title: Java类中代码加载顺序
date: 2020-02-04
tags:
	- Java
---

Java中静态变量、静态代码块、构造代码块、成员变量和构造方法的加载顺序，以及父类、子类中的加载顺序。

<!-- more -->

**一、单一类中代码的首次加载顺序**

> 静态变量/静态代码块 >>> 成员变量/构造代码块 >>> 构造方法

*代码示例：*

```java
/**
 * @date 2020-02-04
 * @author Merlin
 * 静态变量、静态代码块和构造方法的加载次序
 */
public class Test {
	public static void main(String[] args) {
		new Test();
		//new Test().staticFunction();
	}
	
	/*
	static {
		System.out.println(a); // 报错，因为静态变量a的加载顺序在此静态代码块之后
	}
	*/
	
	public static int a = 1; // 静态变量
	
	// 构造代码块
	{
		System.out.println("静态变量比构造代码块先加载，a = "+a);
		a = 3;
		System.out.println("构造代码块被加载");
	}
	
	public Test(){
		System.out.println("构造方法被加载");
		System.out.println("构造代码块加载时，修改了静态变量a的值：a = "+a);
		System.out.println("构造方法被加载前，完成成员变量初始化：b = "+b); 
	}
	
	private int b = 5; // 成员变量初始化
	
	// 静态代码块
	static {
		System.out.println("静态代码块被加载");
	}
	
	// 静态方法只有在被调用时才会加载
	public static void staticFunction() {
		System.out.println("静态方法被调用");
	}
}
```

*代码的执行结果如下：*

```java
静态代码块被加载
静态变量比构造代码块先加载，a = 1
构造代码块被加载
构造方法被加载
构造代码块加载时，修改了静态变量a的值：a = 3
构造方法被加载前，完成成员变量初始化：b = 5
```

结合示例代码和其运行结果，可知在**（首次）加载类**的时候，

- 首先，加载**静态变量**和**静态代码块**，其中二者的加载顺序**与代码顺序相同**；
- 其次，*在调用构造方法后*，按代码顺序**先加载成员变量**和**构造代码块**；
- 最后，加载**构造函数**。



值得注意的是：

- 静态变量和静态代码块仅在首次加载类时被加载；
- 每调用一次构造方法，都要按序加载成员变量、构造代码块、构造函数；
- 静态方法仅在被调用时才会加载。

---



**二、首次加载时父类与子类的情况**

*加载顺序规则如下*：

- 1、按代码位置加载**父类**的静态变量和静态代码块；
- 2、按代码位置加载**子类**的静态变量和静态代码块；
- 3、按代码位置加载**父类**的成员变量和构造代码块；
- 4、加载**父类的构造方法**；
- 5、按代码位置加载**子类**的成员变量和构造代码块；
- 6、加载**子类的构造方法**。



*代码示例如下：*

（父类）

```java
public class Father {
	
	static {
		System.out.println("父类的静态代码块被加载");
	}
	
	public Father() {
		System.out.println("父类的成员变量：fatherEle = "+fatherEle);
		System.out.println("父类的构造方法被加载");
	}
	
	private int fatherEle = 520;
	
	{
		System.out.println("父类的构造代码块被加载");
	}
}
```

（子类）

```java
public class Son extends Father {
	
	public Son() {
		System.out.println("子类的成员变量：sonEle = "+sonEle);
		System.out.println("子类的构造方法被加载");
	}
	
	private int sonEle = 1314;
	
	{
		System.out.println("子类的构造代码块被加载");
	}
	
	static {
		System.out.println("子类的静态代码块被加载");
	}
}
```

（测试方法）

```java
/**
 * @date 2020-02-04
 * @author Merlin
 * 父类、子类代码加载顺序
 */
public class Test {
	public static void main(String[] args) {
		new Son();
	}
}
```

*代码的执行结果如下：*

```java
父类的静态代码块被加载
子类的静态代码块被加载
父类的构造代码块被加载
父类的成员变量：fatherEle = 520
父类的构造方法被加载
子类的构造代码块被加载
子类的成员变量：sonEle = 1314
子类的构造方法被加载
```

---



**三、非首次加载时父类与子类的情况**

> 静态变量和静态代码块不需要再加载，其余按上述（3~6）执行。