---
title: Java代理模式
comments: true
top: false
date: 2020-04-29 15:23:21
tags:
	- Proxy
categories:	
	- 学习笔记
---

代理模式属于23种设计模式中的一种。代理提供了对**被代理对象**的间接访问方式，即通过**代理对象**访问被代理对象，并在被代理对象现有功能的基础上添加额外的功能，如**日志**、**事务管理**、**权限控制**等。

本文主要包括如下内容：

- 静态代理与动态代理的比较。
- JDK 动态代理过程及原理。
- WeakCache 缓存机制。
- CGLib 动态代理过程。

<!--more-->

实现代理模式的方法有以下五种：

- 静态代理：将增强的方法编写在代理类中，在编译期就明确了代理类。
- JDK 动态代理：通过 JDK 提供的 Proxy 类中的 newProxyInstance 方法**动态地创建代理类**，基于**反射**实例化代理对象。（只能代理接口）
- CGLib 动态代理：CGLib（Code Generation Library）基于 asm 字节码技术，**生成被代理类的子类并覆盖其中的方法**以实现增强。（代理非 final 修饰的类或接口）
- Aspectj 动态代理：暂略
- Instrumentation 动态代理：暂略

> 备注：JDK 动态代理的底层原理涉及 WeakCache 缓存机制，在学习过程中遇到一些不理解的内容，待后续学习中逐步完善本篇笔记。

#### 一、静态代理

> 静态代理的特点在于编译期就确定了委托类和代理类，其本质是**代理类持有被代理类的实例**，在间接地执行被代理对象的具体方法前先执行新增的公共逻辑。

（1）委托类和代理类之间的约束接口

```java
/**
 * @Description 静态代理类接口，是委托类和代理类都需要实现的接口规范
 */
public interface Animal {
    public void eatfood(String foodName);
}
```

（2）委托类（被代理类）实现该接口

```java
/**
 * @Description 委托类，实现约束接口Animal
 */
public class Lion implements Animal {

    private String name;

    public Lion(String name){
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public void eatfood(String foodName) {
        System.out.println(this.name + " eat " + foodName +"!");
    }
}
```

（3）代理类，编写增强方法。

```java
/**
 * @Description 静态代理类
 */
public class MyProxy implements Animal {

    private Animal animal; // 被代理对象的引用

    public MyProxy(Animal animal){
        this.animal = animal;
    }

    @Override
    public void eatfood(String foodName) {
        System.out.println("=== 这里可以编写增强方法 ===");
        animal.eatfood(foodName);
        System.out.println("=== 这里可以编写增强方法 ===");
    }
}
```

（4）测试

```java
public class Main {
    public static void main(String[] args) {
        Animal animal = new Lion("Lion Herry");
        MyProxy proxyInstance = new MyProxy(animal); // 传入被代理对象，获取代理对象
        proxyInstance.eatfood("mutton");
    }
}
```



#### 二、JDK 动态代理

##### 1. 示例

（1）委托类和代理类之间的约束接口

> JDK 动态代理只能代理接口中声明的方法！

```java
/**
 * @Description 定义委托类和代理类之间的约束行为
 */
public interface Animal {
    public void eatfood(String foodName);
}
```

（2）委托类实现该接口

```java
/**
 * @Description 委托类
 */
public class Lion implements Animal{

    private String name;

    public Lion(String name){
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public void eatfood(String foodName) {
        System.out.println(this.name + " eat " + foodName +"!");
    }
}
```

（3）实现 InvocationHandler 接口，**定义横切逻辑，通过反射机制调用委托类的接口方法**。

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
/**
 * @Description LionInvocationHandler类实现InvocationHandler接口
 * 此类持有委托类的对象引用
 * 此类被Proxy类回调处理
 * 重写接口的invoke方法，添加公共逻辑代码并调用委托类的接口方法
 */
public class LionInvocationHandler<T> implements InvocationHandler {

    private T target; // 委托类的对象引用

    public LionInvocationHandler(T target){
        this.target = target;
    }

    /**
     * @param proxy 代理实例
     * @param method 代理实例调用的接口方法对应的Method实例
     * @param args 形参数组
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args); // 利用反射调用委托类中的相应方法
        after();
        return result; // 返回调用委托类中相应方法得到的返回值
    }
    
    private void before(){
        System.out.println("=== 调用被代理对象的方法前执行本方法 ===");
    }

    private void after(){
        System.out.println("=== 调用被代理对象的方法后执行本方法 ===");
    }
}
```

（4）测试

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

public class Main {
    public static void main(String[] args) {
        Animal animal = new Lion("Lion Herry"); // 目标对象，即被代理对象
        InvocationHandler handler = new LionInvocationHandler(animal);
        Animal animalProxy = (Animal) Proxy.newProxyInstance(animal.getClass().getClassLoader(),
                animal.getClass().getInterfaces(), handler); // 生成代理对象
        animalProxy.eatfood("mutton");
    }
}
```



##### 2. 过程

改写上述（4）中的代码

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

public class Main {
    public static void main(String[] args) throws Exception  {
        Animal animal = new Lion("Lion Herry"); // 1
        InvocationHandler handler = new LionInvocationHandler(animal); // 2
        // 3
        Class<?> proxyClass = Proxy.getProxyClass(Animal.class.getClassLoader(), Animal.class);
        // 4
        Constructor<?> proxyConstructor = proxyClass.getConstructor(InvocationHandler.class);
        // 5
        Animal animalProxy = (Animal) proxyConstructor.newInstance(handler);
        // 6
        animalProxy.eatfood("beaf");
    }
}
```

***JDK 动态代理的详细流程如下：***

1. 创建被代理的对象 `animal` 。
2. 创建调用处理器对象 `handler` 。
3. 根据指定的 ClassLoader 对象和委托类实现的接口，动态地创建代理类 `proxyClass`，此类**继承 Proxy** 。
4. 利用**反射**获取动态代理类的构造器 `proxyConstructor`，其构造函数形参类型是调用处理器接口类型。
5. 创建动态代理类实例 `animalProxy`，传入调用处理器对象 `handler` 。
6. 调用委托类的接口方法。



##### 3. 原理

在测试类中添加保存生成的代理类字节码文件的语句

```java
public class Main {
    public static void main(String[] args) {
        // 保存生成的代理类字节码文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");

        Animal animal = new Lion("Lion Herry"); // 目标对象，即被代理对象
        InvocationHandler handler = new LionInvocationHandler(animal);
        Animal animalProxy = (Animal) Proxy.newProxyInstance(animal.getClass().getClassLoader(),
                animal.getClass().getInterfaces(), handler); // 生成代理对象
        animalProxy.eatfood("mutton");
    }
}
```

（1）首先看 InvocationHandler 接口源码

java.lang.reflect 包中的 InvocationHandler 是由代理实例的调用处理程序实现的接口。**每个代理实例都有一个关联的调用处理程序对象。**在代理实例上调用接口方法时，该方法调用将被编码并分派至其调用处理程序对象的 invoke 方法。

接口中仅声明了一个 invoke 方法。此方法用于处理代理实例上的方法调用，并返回结果。

```java
public interface InvocationHandler {
    /**
     * @param proxy 代理实例
     * @param method 代理实例调用的接口方法对应的Method实例
     * @param args 形参数组
     * @return 返回代理实例调用接口方法的返回值
     */
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

（2）实现 InvocationHandler 接口，重写 invoke 方法并织入横切逻辑。（代码同前）

（3）调用 java.lang.reflect.Proxy 的静态 newProxyInstance 方法获取代理类的实例，该实例将方法调用分派到指定的调用处理程序。

接下来关注 newProxyInstance 方法的源码。

```java
@CallerSensitive
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,
                              InvocationHandler h) throws IllegalArgumentException {
    
    // 校验与代理实例关联的调用处理程序对象是否为null。若为null，抛NullPointerException。    
    Objects.requireNonNull(h);
	// 克隆委托类实现的接口
    final Class<?>[] intfs = interfaces.clone();
    // 获取系统安全性接口。若已为当前应用程序建立安全管理器，则返回该安全管理器。否则返回null。
    final SecurityManager sm = System.getSecurityManager();
    
    if (sm != null) {
        // 若系统安全性接口不为null，调用Proxy类的私有方法checkProxyAccess，检查创建代理类所需的权限。
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }
    
    // 调用Proxy类的私有方法getProxyClass0，获取代理类
    Class<?> cl = getProxyClass0(loader, intfs); // 此方法源码分析见详情[1]

    try {
        if (sm != null) {
            checkNewProxyPermission(Reflection.getCallerClass(), cl);
        }

        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException|InstantiationException e) {
        throw new InternalError(e.toString(), e);
    } catch (InvocationTargetException e) {
        Throwable t = e.getCause();
        if (t instanceof RuntimeException) {
            throw (RuntimeException) t;
        } else {
            throw new InternalError(t.toString(), t);
        }
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

---

*详情 [1]：Proxy 类的私有方法 getProxyClass0 源码*

```java
/**
 * @param loader 委托类类加载器
 * @param interfaces 委托类实现的接口组成的数组
 */
private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
    // 限制委托类实现的接口最多为65535个
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
	// 尝试从代理类的缓存中获取相应的代理类，否则通过ProxyClassFactory生成代理类
    return proxyClassCache.get(loader, interfaces); // 此方法的源码分析见详情[2]
}
```

---

*详情 [2]：查找或生成代理类*

Proxy 类内部使用 **WeakCache 来缓存代理类**。

Proxy 类加载时，静态常量 `proxyClassCache` 被初始化，其中二级缓存 key 的生成器（subKeyFactory）是 Proxy 类的 **KeyFactory**，二级缓存 value 的生成器（valueFactory）是 Proxy 类的 **ProxyClassFactory**。

```java
// Proxy类的静态字段proxyClassCache，表示代理类的缓存
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

*final 修饰的 java.lang.reflect.WeakCache 类源码（常量字段 & 构造方法）*

```java
final class WeakCache<K, P, V> {
    
    // 引用队列，存放被回收的WeakReference
    private final ReferenceQueue<K> refQueue = new ReferenceQueue<>();
    // 缓存容器，由一、二级缓存组成
    private final ConcurrentMap<Object, ConcurrentMap<Object, Supplier<V>>> map
        = new ConcurrentHashMap<>();
    // 记录所有代理类生成器是否可用，实现缓存的过期机制
    private final ConcurrentMap<Supplier<V>, Boolean> reverseMap
        = new ConcurrentHashMap<>();
    // BiFunction接口表示方法接收两个参数并返回一个结果
    // 二级缓存的key的生成器，(key, parameter) -> sub-key
    private final BiFunction<K, P, ?> subKeyFactory;
    // 二级缓存的value的生成器，(key, parameter) -> value
    private final BiFunction<K, P, V> valueFactory;
    
    /**
     * 构造函数，传入二级缓存的key和value的生成器
     */
    public WeakCache(BiFunction<K, P, ?> subKeyFactory,
                     BiFunction<K, P, V> valueFactory) {
        this.subKeyFactory = Objects.requireNonNull(subKeyFactory);
        this.valueFactory = Objects.requireNonNull(valueFactory);
    }
 
}
```

接下来看 WeakCache 类中获取代理类的 get 方法的源码。

WeakCache 类的缓存容器底层结构是 ConcurrentHashMap，因而实现线程安全。在 get 方法中，根据给定的类加载器和接口数组得到二级缓存的值，即 **Factory 实例**，最终代理类是通过这个 Factory 实例生成的。

```java
/**
 * @param key 传入的是委托类加载器
 * @param parameter 传入的是接口数组（委托类实现的接口）
 */
public V get(K key, P parameter) {
    // 校验接口数组是否为null。若为null，抛NullPointerException。
    Objects.requireNonNull(parameter);
	// 清除过期缓存
    expungeStaleEntries();
	// 将类加载器包装成CacheKey。CacheKey包含key的弱引用，在refQueue中注册。
    // 此对象作为一级缓存的key
    Object cacheKey = CacheKey.valueOf(key, refQueue);
	// 获取二级缓存容器
    ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
    // 当二级缓存容器为null时
    if (valuesMap == null) {
        // 以CAS操作的方式，新建二级缓存容器并存入一级缓存容器map中
        ConcurrentMap<Object, Supplier<V>> oldValuesMap
            = map.putIfAbsent(cacheKey,
                              valuesMap = new ConcurrentHashMap<>());
        if (oldValuesMap != null) {
            valuesMap = oldValuesMap;
        }
    }
	// 根据类加载器和接口数组生成二级缓存的key，即subKey
    Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
    // 尝试通过subKey获取二级缓存的value
    Supplier<V> supplier = valuesMap.get(subKey);
    Factory factory = null;

    while (true) {
        // 若二级缓存的value不为null
        if (supplier != null) {
            // 调用WeakCache的内部类Factory的get方法，尝试获取Factory实例
            V value = supplier.get();
            if (value != null) {
                return value;
            }
        }
        // 新建一个Factory实例
        if (factory == null) {
            factory = new Factory(key, parameter, subKey, valuesMap);
        }
		// 若上述获取的二级缓存的value为null
        if (supplier == null) {
            // 存入新建的Factory实例
            supplier = valuesMap.putIfAbsent(subKey, factory);
            if (supplier == null) {
                // 成功将Factory实例放入缓存
                supplier = factory;
            }
        } else {
            // 期间可能有其他线程修改了值，则将旧值替换
            if (valuesMap.replace(subKey, supplier, factory)) {
                // 替换成功
                supplier = factory;
            } else {
                // 替换失败，继续使用旧值
                supplier = valuesMap.get(subKey);
            }
        }
    }
}
```

接下来看 WeakCache 的内部类 Factory 的源码。

```java
private final class Factory implements Supplier<V> {
	// 一级缓存key，根据类加载器生成
    private final K key;
    // 接口数组
    private final P parameter;
    // 二级缓存key，根据接口数组生成
    private final Object subKey;
    // 二级缓存容器
    private final ConcurrentMap<Object, Supplier<V>> valuesMap;

    Factory(K key, P parameter, Object subKey,
            ConcurrentMap<Object, Supplier<V>> valuesMap) {
        this.key = key;
        this.parameter = parameter;
        this.subKey = subKey;
        this.valuesMap = valuesMap;
    }

    @Override
    public synchronized V get() { // serialize access
        // 获取
        Supplier<V> supplier = valuesMap.get(subKey);
        if (supplier != this) {
            // something changed while we were waiting:
            // might be that we were replaced by a CacheValue
            // or were removed because of failure ->
            // return null to signal WeakCache.get() to retry
            // the loop
            return null;
        }
        // else still us (supplier == this)

        // create new value
        V value = null;
        try {
            value = Objects.requireNonNull(valueFactory.apply(key, parameter));
        } finally {
            if (value == null) { // remove us on failure
                valuesMap.remove(subKey, this);
            }
        }
        // the only path to reach here is with non-null value
        assert value != null;

        // wrap value with CacheValue (WeakReference)
        CacheValue<V> cacheValue = new CacheValue<>(value);

        // put into reverseMap
        reverseMap.put(cacheValue, Boolean.TRUE);

        // try replacing us with CacheValue (this should always succeed)
        if (!valuesMap.replace(subKey, this, cacheValue)) {
            throw new AssertionError("Should not reach here");
        }

        // successfully replaced us with new CacheValue -> return the value
        // wrapped by it
        return value;
    }
}
```

【未完待续~~~】

---



#### 三、CGLib 动态代理

CGLib 动态代理的特点如下：

- CGLib 可代理接口和类，其中接口使用实现的方式，类使用的是继承的方式。
- final 修饰的类是不能被代理的。final、private、static 修饰的方法是不能被代理的。

（1）委托类

```java
/**
 * @Description 委托类
 */
public class Lion {
    public void eatfood(String foodName) {
        System.out.println("The lion is eating " + foodName +"!");
    }
}
```

（2）实现方法拦截器接口

```java
public class CglibProxy implements MethodInterceptor {

    /**
     * 生成Cglib动态代理类的工具方法
     * @param target 委托类的Class对象
     */
    public Object getProxy(Class target){
        Enhancer enhancer = new Enhancer(); // 创建加强器，用于创建动态代理类。
        enhancer.setSuperclass(target); // 指定委托类
        enhancer.setCallback(this); // 设置方法拦截器回调引用，对于代理类上所有方法的调用，都会调用CallBack，而Callback则需要实现intercept()方法进行拦截。
        return enhancer.create(); // 获取动态代理类对象并返回
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, 
                            MethodProxy proxy) throws Throwable {
        before();
        Object result = proxy.invokeSuper(obj, args);
        after();
        return result;
    }

    private void before(){
        System.out.println("=== 调用被代理对象的方法前执行本方法 ===");
    }

    private void after(){
        System.out.println("=== 调用被代理对象的方法后执行本方法 ===");
    }
}
```

（3）测试

```java
public class Main {
    public static void main(String[] args) {
        CglibProxy proxy = new CglibProxy();
        Lion proxyInstance = (Lion) proxy.getProxy(Lion.class);
        proxyInstance.eatfood("mutton");
    }
}
```



#### 总结

- 静态代理在编译期就确定了代理类，而动态代理是在程序运行期间动态地创建代理类。
- JDK 动态代理生成的动态代理类继承 Proxy 类。Java 不允许多继承，因此 JDK 动态代理**只能代理接口！**
- JDK 动态代理只能代理接口方法！若接口的实现类中定义了接口中未声明的方法，该方法不能被代理。
- JDK 动态代理创建动态代理对象的过程快，但调用委托类的接口方法慢；CGLib 动态代理创建动态代理对象的过程慢，但调用委托类的接口方法快。



#### 参考资料

[1] [太好了！总算有人把动态代理、CGlib、AOP都说清楚了！](https://cloud.tencent.com/developer/article/1461796)

[2] [JDK动态代理实现原理](https://www.cnblogs.com/zuidongfeng/p/8735241.html)

[3] [JDK动态代理[2]----JDK动态代理的底层实现之Proxy源码分析](https://www.cnblogs.com/liuyun1995/p/8157098.html)