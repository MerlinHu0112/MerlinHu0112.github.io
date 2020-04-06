---
title: Java的反射机机制：原理与使用
comments: true
top: false
date: 2020-03-25
tags:
	- Reflection
categories:
	- 学习笔记
---

在面试过程中被问及Java反射，回答得比较简单、浅显。现复习、总结成文。

<!--more-->

#### 一、概述

##### 1. 什么是反射？

反射就是把Java类中的字段、方法、构造器和修饰符等映射成一个个的Java对象（Field、Method、Constructor、Modifier）。

通过反射，能在运行状态知道任意类的**所有属性和方法**（包括private），且能在运行状态调用任意对象的所有属性和方法。

##### 2. Java反射机制主要提供了以下功能：

- 在运行时判断任意一个对象所属的类。
- 在运行时构造任意一个类的对象。
- 在运行时判断任意一个类所具有的成员变量和方法。
- 在运行时调用任意一个对象的方法。
- 生成动态代理。

##### 3. 反射的主要涉及的类

- java.lang.Class

- java.lang.reflect.Field

- java.lang.reflect.Method

- java.lang.reflect.Constructor

- java.lang.reflect.Modifier



#### 二、反射的使用

反射的基石：JVM将相应的字节码文件（Class文件）加载到内存中，获得字节码文件对象（Class对象）。

##### 1. 如何获取Class对象

> 三种方式：1. Class.forName(String className)；2. obj.getClass()；3. Object.class。

*以People类为例*

```java
public class People {

    private String name;
    private int age;
    private String province;
    private String city;

    public People(){}

    public People(String name, int age,
                  String province, String city){
        this.name = name;
        this.age = age;
        this.province = province;
        this.city = city;
    }

    @Override
    public String toString() {
        return "name = "+name+", age = "+age+", address = "+province+city;
    }

    // get/set方法略
}
```

**（1）Class类的静态方法forName**

调用java.lang.Class类的静态方法，扩展性强，使用较为普遍。

```java
public static Class<?> forName(String className){} // 形参为类的全限定名
Class clazz = Class.forName("cn.merlin.test.reflect.People");
```

（2）Object基类中的getClass方法

前提是需要有相应的类的实例，如首先创建People类的实例，随后调用该实例对象的getClass方法。

```java
People p = new People();
Class clazz = p.getClass();
```

（3）任意Java类型（或viod关键字）的class属性

静态属性class。

```java
Class clazz = People.class;
```

 

##### 2. 字节码文件仅加载一次

下述代码输出结果为：`true`，说明 `clazzA` 与 `clazzB` 是同一个对象，即**在运行过程中，字节码文件是仅被加载一次的**。因此，对于同一个字节码文件，多次调用返回的 `Class` 对象是相同的。

```java
// clazzA字节码对象
String className = "cn.merlin.test.reflect.People";
Class clazzA = Class.forName(className);
// clazzB字节码对象
People people = new People();
Class clazzB = people.getClass();
// 判断二者的内存地址
System.out.println(clazzA==clazzB);
```



##### 3. 利用反射分析类

在java.lang.reflect包中有三个类Field、Method和Constructor，分别用于描述类的字段、方法和构造器。在reflect包中还有Modifier类，用于描述访问权限。

##### 3.1 Class类

> 在程序运行期间，Java运行时系统始终为所有的对象维护一个被称为运行时的类型标识，它跟踪着每个对象所属的类。JVM利用运行时类型信息选择相应的方法执行。
>
> Class类：Java提供的专门的类，用来保存并访问上述信息。

（1）Class类常见的方法如下：

```java
public int getModifiers() {} // 返回类或接口的修饰符，以整数编码。包括public, protected, private, final, static, abstract and interface
public String getName() {} // 返回名称
public Field[] getFields() {} // 返回公共的字段数组
public Field[] getDeclaredFields() {} // 返回全部的字段数组
public Method[] getMethods() {} // 返回公共的方法数组
public Method[] getDeclaredMethods() {} // 返回全部的方法数组
public Constructor<?>[] getConstructors() {} // 返回公共的构造器数组
public Constructor<?>[] getDeclaredConstructors() {} // 返回全部的构造器数组
public T newInstance() {} // 调用默认的无参构造器，实例化对象
```

（2）Class类中的getFields、getMethods、getConstructors方法，分别返回类的**public**的字段、方法和构造器数组，包括**父类的public成员**。

（3）Class类中的getDeclaredFields、getDeclaredMethods、getDeclaredConstructors方法，分别返回类的**全部**的域、方法和构造器数组，包括**私有和默认的成员，但不含父类的成员（包括父类的公共成员）**。



##### 3.2 Field类

描述类或接口的字段及动态访问信息，包括类静态字段和实例字段。

Field类的常用方法如下：

```java
public String getName(){} // 返回字段名称
public Class<?> getType(){} // 返回Class对象，该对象标识此Field对象标识的字段的声明类型
public int getModifiers(){} // 返回字段的修饰符，以整数编码
```



##### 3.3 Constructor类

Constructor类提供有关类的单个构造函数的信息，并提供对此类的访问。

```java
public String getName(){} // 以字符串形式返回构造器的名称
public int getModifiers(){} // 返回构造器的修饰符，以整数编码
public Class<?>[] getParameterTypes(){} // 返回Class对象数组，表示构造器的形参类型
public int getParameterCount() {} // 返回构造器的形参数量
public T newInstance(Object ... initargs) {} // 调用有参构造器，实例化对象
```



##### 3.4 Method类

```java
public String getName(){} // 以字符串形式返回构造器的名称
public int getModifiers(){} // 返回构造器的修饰符，以整数编码
public Class<?>[] getParameterTypes(){} // 返回Class对象数组，表示方法的形参类型
public int getParameterCount() {} // 返回方法的形参数量
public Class<?> getReturnType(){} // 返回方法的返回值类型，Class对象
public Object invoke(Object obj, Object... args){} // 调用对象的方法
```



##### 3.5 Modifier类

Modifier类提供静态方法以获取字段、方法和类的修饰符，其中修饰符以整数表示。

Modifier提供的静态方法判断修饰符的核心算法—— `&` 运算。

```java
// 以判断是否是public为例
public static boolean isPublic(int mod) {
    return (mod & PUBLIC) != 0;
}
```

|  Java修饰符  | Modifier类的静态变量名 |  十六进制  | 十进制 |
| :----------: | :--------------------: | :--------: | :----: |
|    public    |         PUBLIC         | 0x00000001 |   1    |
|   private    |        PRIVATE         | 0x00000002 |   2    |
|  protected   |       PROTECTED        | 0x00000004 |   4    |
|    static    |         STATIC         | 0x00000008 |   8    |
|    final     |         FINAL          | 0x00000010 |   16   |
| synchronized |      SYNCHRONIZED      | 0x00000020 |   32   |
|   volatile   |        VOLATILE        | 0x00000040 |   64   |
|  transient   |       TRANSIENT        | 0x00000080 |  128   |
|    native    |         NATIVE         | 0x00000100 |  256   |
|  interface   |       INTERFACE        | 0x00000200 |  512   |
|   abstract   |        ABSTRACT        | 0x00000400 |  1024  |
|    strict    |         STRICT         | 0x00000800 |  2048  |



#### 三、反射的底层原理

##### 1. 代码示例

*以使用 Class.forName()为例*

*为上述 People类添加 helloWorld方法*

```java
public void helloWorld() {
    System.out.println("Hello World!");
}
```

```java
public static void main(String[] args) throws Exception {
    String className = "cn.merlin.test.reflect.People";
    Class clazz = Class.forName(className); // (1)
    People p = (People) clazz.newInstance(); // (2)
    p.setName("Merlin");
    p.setAge(25);
    p.setProvince("安徽");
    p.setCity("安庆");
    System.out.println(p.toString());
    p.helloWorld();
}
```

（1）首先调用Class类的forName方法获取Class对象，形参为类或接口的全限定名。

```java
@CallerSensitive
public static Class<?> forName(String className)
    throws ClassNotFoundException {
    // 获取被调用的类的信息
    Class<?> caller = Reflection.getCallerClass();
    // 调用本地方法，获取Class对象
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}
```

调用ClassLoader类的getClassLoader方法，获取类加载器。

```java
// ClassLoader.getClassLoader(caller)
static ClassLoader getClassLoader(Class<?> caller) {
    // caller: "class cn.merlin.test.reflect.PeopleReflectDemoA"
    if (caller == null) {
        return null;
    }
    return caller.getClassLoader0();
}
```

调用Class类的方法，获取类加载器。

```java
// caller.getClassLoader0();
// 返回类加载器 
ClassLoader getClassLoader0() { return classLoader; }
```

forName0()是本地方法，由JVM实现。

```java
private static native Class<?> forName0(String name, 
    boolean initialize, ClassLoader loader, Class<?> caller)
    throws ClassNotFoundException;
```

（2）调用Class类的newInstance方法，实例化People对象。

> Class类的newInstance方法，调用的是People类的无参构造函数。若People类没有无参构造函数（显式或隐式），则抛出InstantiationException异常。

```java
@CallerSensitive
public T newInstance() throws InstantiationException, IllegalAccessException {
    /*
     * 判断是否为当前应用程序建立安全管理器
     * System.getSecurityManager()方法获取安全管理器
     */
    if (System.getSecurityManager() != null) {
        // 检查是否允许访问成员
        checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
    }

    // NOTE: the following code may not be strictly correct under
    // the current Java memory model.

    // Constructor lookup
    if (cachedConstructor == null) {
        // 当前调用者是Class.class，不允许调用
        if (this == Class.class) {
            throw new IllegalAccessException(
                "Can not call newInstance() on the Class for java.lang.Class"
            );
        }
        try {
            // 获取无参构造函数
            Class<?>[] empty = {};
            // 详解【1】
            final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
            // Disable accessibility checks on the constructor
            // since we have to do the security check here anyway
            // (the stack depth is wrong for the Constructor's
            // security check to work)
            java.security.AccessController.doPrivileged(
                new java.security.PrivilegedAction<Void>() {
                    public Void run() {
                        c.setAccessible(true);
                        return null;
                    }
                });
            cachedConstructor = c;
        } catch (NoSuchMethodException e) {
            throw (InstantiationException)
                new InstantiationException(getName()).initCause(e);
        }
    }
    Constructor<T> tmpConstructor = cachedConstructor;
    // Security check (same as in java.lang.reflect.Constructor)
    int modifiers = tmpConstructor.getModifiers();
    if (!Reflection.quickCheckMemberAccess(this, modifiers)) {
        Class<?> caller = Reflection.getCallerClass();
        if (newInstanceCallerCache != caller) {
            Reflection.ensureMemberAccess(caller, this, null, modifiers);
            newInstanceCallerCache = caller;
        }
    }
    // Run constructor
    try {
        return tmpConstructor.newInstance((Object[])null);
    } catch (InvocationTargetException e) {
        Unsafe.getUnsafe().throwException(e.getTargetException());
        // Not reached
        return null;
    }
}
```

*详解【1】*

```java
/**
 * Class类的私有方法
 * @param which 构造函数的访问控制符。1：public
 */
private Constructor<T> getConstructor0(Class<?>[] parameterTypes, int which) throws NoSuchMethodException {
    /*
 	 * 详解【2】
 	 * 若构造器的访问控制符为public，通过ReflectionFactory.copyConstructor
 	 * 复制得到构造函数数组。
 	  */
    Constructor<T>[] constructors = privateGetDeclaredConstructors((which == Member.PUBLIC));
    for (Constructor<T> constructor : constructors) {
        if (arrayContentsEq(parameterTypes,
                            constructor.getParameterTypes())) {
            return getReflectionFactory().copyConstructor(constructor);
        }
    }
    throw new NoSuchMethodException(getName() + ".<init>" + argumentTypesToString(parameterTypes));
}
```

*详解【2】*

```java
private Constructor<T>[] privateGetDeclaredConstructors(boolean publicOnly) {
    checkInitted();
    Constructor<T>[] res;
    ReflectionData<T> rd = reflectionData();
    if (rd != null) {
        res = publicOnly ? rd.publicConstructors : rd.declaredConstructors;
        if (res != null) return res;
    }
    // No cached value available; request value from VM
    if (isInterface()) {
        @SuppressWarnings("unchecked")
        Constructor<T>[] temporaryRes = (Constructor<T>[]) new Constructor<?>[0];
        res = temporaryRes;
    } else {
        res = getDeclaredConstructors0(publicOnly);
    }
    if (rd != null) {
        if (publicOnly) {
            rd.publicConstructors = res;
        } else {
            rd.declaredConstructors = res;
        }
    }
    return res;
}
```



---

2、Spring IoC是如何利用反射的

