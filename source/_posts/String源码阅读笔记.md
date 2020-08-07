---
title: String源码阅读笔记
comments: true
top: false
date: 2020-05-22 09:51:34
tags:
	- String
categories:
	- Java源码
---

基于 JDK 1.8 的 String 类的源码阅读笔记，重点如下：

- String 类是不可变的，它不对外提供修改字符串的方法，但可以通过反射技术修改保存字符串的字符数组。
- StringBuffer、StringBuilder 类支持可变字符串，前者还是线程安全的类。
- 计算哈希值时采用 31 作为乘子，是考虑到减少哈希冲突和便于计算。

<!--more-->

#### 1. 概述

String 类用来存储字符串，由 final 修饰，因此不能被继承。在 String 内部通过字符数组对象 `value` 保存字符串，且该字符数组也是由 final 修饰的。需要注意的是，String 类对外提供的修改字符串的方法，均会生成一个新的字符串对象。此外，字符串缓存区（StringBuffer 类、StringBuilder 类）是支持可变字符串的。

String 类的常见方法有：检查序列中的各个字符，比较字符串，搜索字符串，字符大小写转换，提取子字符串并创建字符串的副本，等等。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {}
```

- Serializable 接口：序列化标志接口，无任何定义方法。
- Comparable 接口：按序比较单个字符的 ASCII 码，以比较字符串的大小。
- CharSequence 接口：字符值的可读序列。



#### 2. 字段与构造方法

##### 2.1 常量和成员变量

```java
private final char value[]; // 字符数组，用于存储字符串
private int hash; // 缓存字符串的哈希值
```



##### 2.2 构造方法

String 类的构造方法较多，以下仅介绍常用的几种。

```java
public String(String original); // 新建original对象的副本
public String(char value[]); // 根据字符数组新建String对象
public String(StringBuffer buffer); // 根据StringBuffer对象新建String对象
public String(StringBuilder builder); // 根据StringBuilder对象新建String对象
```

（1）

```java
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```

（2）调用工具类 Arrays 的 copyOf 方法拷贝原字符数组，得到副本。

```java
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}
```

（3）因 StringBuffer 是同步的，故对对象 `buffer` 加锁。

```java
public String(StringBuffer buffer) {
    synchronized(buffer) {
        this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
    }
}
```

（4）

```java
public String(StringBuilder builder) {
    this.value = Arrays.copyOf(builder.getValue(), builder.length());
}
```



#### 3. 常用方法

```java
public int hashCode() {} // 获取String对象的哈希值
public boolean equals(Object anObject) {} // 比较两个字符串是否相等，区分大小写
public char[] toCharArray() {} // 将字符串转换成字符数组
public String substring(int beginIndex, int endIndex) {} // 根据首尾索引截断字符串
public String trim() {} // 删除字符串中前部和尾部空格
public int compareTo(String anotherString) {}
public String concat(String str) // 将字符串str拼接到当前字符串尾部
```



##### 3.1 hashCode() 方法

String 类重写基类 Object 的 hashCode 方法，计算哈希值的公式为：
$$
s[0]*31^{(n-1)} + s[1]*31^{(n-2)} + ... + s[n-1]
$$
其中 `n` 为字符串长度，`s[n]` 为相应字符的 ASCII 码值。

**为什么选择 31 作为乘子？**

- 使用 31 作为乘子得到的哈希值分布较为均匀，减少了哈希冲突。
- 当然，使用常数 31, 33, 37, 39 或 41 作为乘子，哈希冲突情况都比较理想。此时涉及的 31 的另一个重要特性：`31 * i = (i<<5) - i`。现代的 JVM 能自动将 `31 * i` 乘法运算优化为移位和减法运算，提高了计算性能。

```java
public int hashCode() {
    int h = hash; // hash默认为0
    if (h == 0 && value.length > 0) {
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

------

##### 3.2 equals(object anObject) 方法

重写基类 Object 的 equals 方法，将引用比较（地址比较）改为值比较。

```java
public boolean equals(Object anObject) {
    // 两个对象的内存地址相同，即同一个对象，直接返回true
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                // 比较两个字符的ASCII码
                // 注意：大小写字母是不等的
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true; // 只有所有字符一一对应，才返回true
        }
    }
    return false;
}
```

------

##### 3.3 toCharArray() 方法

此方法可以将字符串转换成字符数组。使用此方法需要注意两点：

- 不能直接返回 `value` 对象，必须返回副本。虽然 `value` 数组是被 final 修饰的，这仅仅意味着**它指向的内存地址是不可改变的**，但是该地址保存的内容仍是可以被修改的，最终导致 String 的不变性被破坏。
- 获取副本必须使用 System 类的 arrayCopy 方法。根据源码注释，这是因为类的初始化顺序问题。

```java
public char[] toCharArray() {
    char result[] = new char[value.length];
    // 必须使用System类的arrayCopy方法
    System.arraycopy(value, 0, result, 0, value.length);
    return result;
}
```

------

##### 3.4 subString(int beginIndex, int endIndex) 方法

此方法用来截断字符串。在合法的参数范围内，子串应从原字符串的索引 `beginIndex` 处开始，直至索引 `endIndex - 1` 处，子串长度为 `endIndex - beginIndex`。

```java
public String substring(int beginIndex, int endIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    if (endIndex > value.length) {
        throw new StringIndexOutOfBoundsException(endIndex);
    }
    int subLen = endIndex - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    // 若恰好截取整个原字符串，直接返回原字符串对象
    return ((beginIndex == 0) && (endIndex == value.length)) ? this
        : new String(value, beginIndex, subLen);
}
```

------

##### 3.5 trim() 方法

此方法用于消除字符串首、尾部的空格。若原字符串为空，或首字符和尾字符均非空格，则直接返回原字符串对象。

```java
public String trim() {
    int len = value.length;
    int st = 0;
    char[] val = value;    /* avoid getfield opcode */
	// 循环寻找头部的空格
    while ((st < len) && (val[st] <= ' ')) {
        st++;
    }
    // 循环寻找尾部的空格
    while ((st < len) && (val[len - 1] <= ' ')) {
        len--;
    }
    return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
}
```

------

##### 3.6 compareTo(String anotherString) 方法

与 equals 方法有点相似，但不完全相同。当字符串相同时，返回 `0`。

```java
public int compareTo(String anotherString) {
    int len1 = value.length;
    int len2 = anotherString.value.length;
    int lim = Math.min(len1, len2);
    char v1[] = value;
    char v2[] = anotherString.value;

    int k = 0;
    while (k < lim) {
        char c1 = v1[k];
        char c2 = v2[k];
        if (c1 != c2) {
            return c1 - c2;
        }
        k++;
    }
    return len1 - len2;
}
```

------

##### 3.7 concat(String str) 方法

将字符串 `str` 拼接至当前字符串的尾部，底层实现基于字符数组的拷贝。

```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len); // 拼接至字符数组
    return new String(buf, true);
}
```

调用 getChars(char dst[], int dstBegin) 方法

```java
// 从索引dstBegin开始，将当前字符串对象放入进dst字符数组中
void getChars(char dst[], int dstBegin) {
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}
```



#### 4. 问题

##### String 类是不可变的，那么一定是完全不会被修改的吗？

（1）使用 String 类提供的方法尝试修改

```java
public class Main {
    public static void main(String[] args) throws NoSuchFieldException {
        String oldStr = "Hello World";
        String newStr = oldStr.replace(' ', '-');
        System.out.println("old string: "+oldStr.toString());
        System.out.println("new string: "+newStr.toString());
    }
}
```

输出结果为：

```
old string: Hello World
new string: Hello-World
```

调用 String 类提供的方法修改字符串，**返回的是一个新的字符串，原字符串并未改变**。

（2）使用反射技术尝试修改

```java
public class Main {
    public static void main(String[] args) 
        				throws NoSuchFieldException, IllegalAccessException {
        String oldStr = "Hello World";
        Class clazz = oldStr.getClass();
        // 反射获取字段对象
        Field field = clazz.getDeclaredField("value");
        // 获取私有属性的访问权限
        field.setAccessible(true);
        char[] arr = (char[]) field.get(oldStr);
        arr[5] = '-';
        System.out.println(oldStr);
    }
}
```

输出结果为：

```
Hello-World
```

在反射前后 `oldStr` 对象的 hashCode 也保持不变。String 类不对外提供修改原字符串的方法，但通过反射技术仍能够修改原字符串。



#### 参考资料

[1] [科普：为什么 String hashCode 方法选择数字31作为乘子](https://segmentfault.com/a/1190000010799123)

