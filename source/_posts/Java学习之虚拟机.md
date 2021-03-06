---
title: Java学习之虚拟机
comments: true
date: 2019-11-11
tags: 
	- JVM
categories:	
	- 学习笔记
---

通读周志明先生的《深入理解Java虚拟机》一书，学习了Java虚拟机的内存模型，内存管理机制，常见的垃圾回收算法和不同类型的垃圾收集器，以及字节码文件的结构。现整理成笔记，温故而知新！

<!-- more -->

### 一、内存管理机制

#### 1. Java内存区域图示如下：

![](Java学习之虚拟机/JVM内存区域示意.png)

值得注意的是：**Java虚拟机栈（Java Virtual Machine Stacks）**、**堆（Heap）**、**方法区（Method Area）**：

1. 虚拟机栈主要与方法调用相关，存储的局部变量表中保存**对象引用**；

2. 堆：存放所有的对象实例；

3. 方法区：与编译相关，存储虚拟机加载的类信息、**静态变量**、常量等；

4. 运行时常量池：编译生成的字面量和符号引用。

![](Java学习之虚拟机/2.png)



#### 2. HotSpot虚拟机新建对象过程

以常见的HotSpot为例，介绍其新建普通 Java对象的过程。

![](Java学习之虚拟机/HotSpot虚拟机new对象过程.jpg)

（1）在堆中分配内存的两种方式：

- 指针碰撞：内存区域规整（未使用区域与已使用区域分界）时，仅需要移动指针

- 空闲列表：因内存区域杂乱无章，虚拟机维持记录可用内存区域的表



**（2）两种保证分配内存时线程安全的方法：**

- 同步机制，CAS + 失败重试

- **每个线程**单独预分配一块内存区域，称之为**本地线程分配缓冲（TLAB）**



（3）对象的内存布局：

1. 对象头：保存运行时数据，如哈希码、GC分代年龄等；另一部分保存指向类元数据的**类型指针**

2. 实例数据部分

3. 填充位



#### 3. 对象的访问方式

访问对象的途径主要是通过句柄和直接访问：

![](Java学习之虚拟机/通过句柄访问对象.jpg)

> **对象类型数据**是指该实例所属的类
>
> 查找对象的元数据信息未经过对象本身
>
> 优点：垃圾收集需要调整对象所在内存地址时，不影响reference；



![](Java学习之虚拟机/通过指针访问对象.jpg)

> 查找对象的元数据信息须经过对象本身
>
> 优点：访问速度快



#### 4、内存分配策略

具体的内存分配策略取决于 GC组合、JVM参数设置：

1. 对象优先在新生代 Eden 区分配，该区域不足时，发起 Minor GC；

2. 占用大内存的对象直接进入老年代；

3. 寿命长的对象进入老年代（根据对象年龄计数器来判断）；

4. Survivor 空间中相同年龄的对象达到一半空间时，年龄大于或等于该年龄的对象可进入老年代；

5. 老年代为 Minor GC提供空间分配担保。



### 二、GC工作区域

在 JVM中，虚拟机栈、本地方法栈和程序计数器属于线程私有，随线程结束而释放内存。JVM中的GC关注的是**方法区**和**堆**。

#### 1. 回收堆

##### 1.1 如何判断对象内存可回收？

**可达性分析算法**：根搜索方法，通过一些“GC Roots”对象作为起点，从这些节点开始往下搜索，搜索通过的路径成为引用链（ReferenceChain）。

![](Java学习之虚拟机/可达性分析算法.jpg)



##### 1.2 Java中可作为GC Roots的四种对象：

- 虚拟机栈的本地变量表中引用的对象

- 方法区类静态属性引用的对象

- 方法区常量引用的对象

- 本地方法栈Native方法引用的对象



##### 1.3 经可达性分析后，可回收对象就一定会被回收？

经可达性分析后，被定为可回收的对象须经历**两次标记**方能被GC回收内存。【重点：finalize()方法】



##### 1.4 JVM为什么不采用经典的**引用计数**算法来判断对象是否可回收？

引用计数算法实现简单，但面临一个重要缺陷：当两个对象互相引用（循环引用）时，相应的内存区域无法被回收，导致**内存泄漏**

![](Java学习之虚拟机/循环引用.jpg)



##### 1.5 引用分类（是否回收被引用对象所在的内存区域）

1. 强引用：new出来的对象引用关系，不会回收；

2. 软引用：有用、非必需，在内存溢出前进行二次回收；

3. 弱引用：生存至下一次GC工作前，一旦GC开始回收内存，则直接回收其对象；

4. 虚引用：无法通过虚引用取得对象，仅能在此对象被回收时通知系统。



#### 2、回收方法区

方法区待回收的内容主要为：废弃常量 和 无用的类。

（1）废弃常量：当常量池中的常量不被任何对象引用时，则可以清理出常量池。

（2）无用的类：无用的类须满足下述三个条件，

- Java堆中该类所有的实例均已被回收；

- 该类的加载器ClassLoader已被回收；

- 无法通过反射机制访问该类的方法。



### 三、回收算法

JVM 针对 Java堆 采用的回收算法主要分为：**标记-清除算法、复制算法和标记-整理算法**；采用的策略为：分代收集算法，即新生代采用复制算法，老年代采用标记-清除算法和标记-整理算法

#### 1. 算法

##### 1.1 标记-清除（Mark-Sweep）算法

在标记阶段，由**根节点开始经可达性分析**，标记所有从根节点开始的可达对象，因此未被标记的对象就是未被引用的垃圾对象；

在清除阶段，清除所有未被标记的对象。

- *特点：将循环引用标记为不可达，解决了引用计数算法无法处理循环引用的问题.*

- *缺点：1、效率低；2、易导致内存区域不规整（即大量不连续的内存碎片）。*

![](Java学习之虚拟机/标记-清除算法.png)



##### 1.2 复制（Copying）算法

内存对半分，先使用一半，需要清理时先将存活对象转移至保留内存区域，随后全部回收刚使用过的一半内存。

**适用于新生代**中对象存活率低、可回收对象比例高的场景。

- *优点：1、无内存碎片；2、效率高。*
- *缺点：1、实际使用内存仅占一半，浪费资源；2、当对象存活率高时，需要较为频繁的复制操作，效率降低。*

![](Java学习之虚拟机/复制算法.png)



##### 1.3 标记-整理（Mark-Compact）算法

针对老年代中对象存活率高，若使用复制算法，则效率低。

标记-整理算法（或标记-压缩算法）核心在于：标记完成后，将“分散”的存活对象移动至“端集中”区域，随后清理其端边界以外的内存。

- *优点：1、适用于对象存活率高的场景；2、无内存碎片。*

- *缺点：标记、整理过程代价高昂。*

![](Java学习之虚拟机/标记-整理算法.png)



##### 1.4 分代收集（Generational Collection）算法

将 Java堆分为**新生代**和**老年代**

- 新生代：对象存活率低，垃圾多；

- 老年代：对象存活率高，垃圾少。

基于上述算法的优缺点，分代收集的策略是：*对新生代采用复制算法，对老年代采用标记-清除或标记-整理算法* 。



在新生代中，分为：较大的**Eden空间**和两块较小的**Survivor空间**。

![](Java学习之虚拟机/新生代具体算法实现.png)

1. 在新生代中，先使用Eden和一块Survivor区域，另一块Survivor区域保留，其空间比例一般为8：1：1；

2. 回收时，Eden和Survivor(From)中**存活时间长和大对象直接进入老年代**，其余存活对象进入Survivor(To)区域；

3. 当Survivor(To)区域空间不足时，利用**分配担保机制**，非存活时间长或大对象的存活对象同样被放入老年代中；

4. 清理Eden和Survivor(From)。



#### 2. HotSpot 的算法实现

（1）在可达性分析算法中，如何快速、精确地找出（枚举）根节点（GC Roots）？

精确：GC进行时停止所有 Java线程，确保分析时引用关系不再变化；

速度：OopMap（Ordinary Object Pointer Map）普通对象指针的Map数据结构，保存对象引用信息。



（2）Safe Point & Safe Region

> 安全点：GC中断线程的“位置”



### 四、垃圾收集器

重要概念：

（1）在垃圾收集器中，并发（Concurrent）与并行（Parallel）的区别：

- 并发：用户进程与垃圾收集进程并发执行。单CPU环境，只能交替执行；多CPU环境，可同时执行。

  > *CMS 在并发标记、清理阶段*

- 并行：用户进程暂停，在多CPU环境中，多条垃圾收集线程可同时执行。

在垃圾收集器中，并发与并行的重要区别在于**前者不用暂停用户进程，后者必须暂停用户进程**，这与常说的线程并发和并行的概念还是有点不同！单核和多核环境只决定了线程能否同时执行。



（2）吞吐量与GC停顿时间的关系

- 吞吐量越高，表示CPU利用效率越高

- GC停顿时间缩短是以牺牲吞吐量和新生代空间大小换来的

因此，追求更短的 GC停顿时间，适合交互频繁的场景，代价是 CPU利用率下降；追求更高的 CPU利用率，适合计算量大的场景，代价是 GC停顿时间延长。*



（3）Minor GC 和 Full GC

- Minor GC：发生在新生代的垃圾收集动作，GC 频繁且速度快；

- Major GC / Full GC：发生在老年代，速度比 Minor GC 慢。



![](Java学习之虚拟机/垃圾收集器.jpg)



#### 1. Serial 收集器

（1）单线程：单条GC线程完成收集，且必须暂停其它所有用户正常工作线程（中断其它的 Java 线程）；

（2）对于单CPU环境，Serial 没有线程交互开销，最高的单线程收集效率；

（3）Client 模式下默认的新生代收集器。

![](Java学习之虚拟机/Serial.jpg)

#### 2. ParNew 收集器

（1）多线程：多条GC线程（多核时才可能同时执行），且必须暂停其它所有用户正常工作线程；

（2）除多线程外，ParNew 其余实现与 Serial 相同；

（3）对于单CPU环境，ParNew 存在线程交互开销，效率低于 Serial；

（4）对于多核CPU环境，ParNew 因多线程并行，效率一般高于 Serial；

（5）Server模式下首选的新生代收集器。

![](Java学习之虚拟机/ParNew.jpg)

#### 3. Parallel Scavenge 收集器

（1）与 ParNew 在基本实现上相似；

（2）吞吐量优先：在GC停顿时间和CPU效率间平衡选择，不同于 CMS 等收集器追求尽可能短的 GC 停顿时		  间；

（3）GC Ergonomics：自适应调节策略。

![](Java学习之虚拟机/ParallelScavenge.jpg)

#### 4. Serial Old 收集器

（1）Serial 收集器的 老年代 版

（2）标记 - 整理 算法

![](Java学习之虚拟机/SerialOld.jpg)

#### 5. Parallel Old 收集器

（1）Parallel Scavenge 收集器的 老年代 版

（2）标记 - 整理 算法

（3）Parallel Scavenge + Parallel Old 组合：吞吐量优先，适合注重吞吐量和CPU资源敏感的场景

![](Java学习之虚拟机/ParallelOld.jpg)

#### 6. CMS 收集器

（1）标记 - 清理 算法

（2）突出点：CMS ( Concurrent Mark Sweep ) 收集器在运行期间，**不需要一直中断其它线程**，因此能实现**最短的 GC 停顿时间**

（3）分为以下四个步骤：

- 初始标记

- 并发标记

- 重新标记

- 并发清理

其中，并发标记 / 并发清理期间，不暂停其它 CPU 核上的线程，耗时较长；初始标记 / 重新标记期间，需要暂停其它所有线程，耗时较短。

（4）CMS 的不足：

- 对CPU资源敏感，使吞吐量下降，CPU利用率降低

- 并发清理阶段其它用户线程产生的“浮动垃圾”须等到下次GC

- 潜在的 Concurrent Mode Failure 问题

- 清理算法产生大量空间碎片

![](Java学习之虚拟机/CMS.jpg)



#### 7. G1( Garbage-First ) 收集器

基本原理：Region + 区域回收优先级

(1) 不再使用 新生代 / 老年代 划分，而是使用 **Region** 划分堆内存区域；

(2) G1追踪不同的 Region，并维护 **优先列表**，由此决定优先GC的区域；

(3) 不同的Region之间的对象引用关系保存在 Remembered Set中（新生代与老年代之间亦是）。

*优点：*

- 更适合多 CPU的环境；

- 可独自运作，且对寿命不同的对象也采取不同的策略；

- 无内存碎片；

- **可预测的暂停**。

![](Java学习之虚拟机/g1.jpg)

运行过程如下：

1. 初始标记：标记GC Roots直接关联的对象；

2. 并发标记：可达性分析，耗时长、与其它 Java线程并发；

3. 最终标记：Remembered Set Logs中数据（记录并发标记阶段的引用变化）与Remembered Set合并；

4. 筛选回收：有选择地确定GC Region。



### 五、虚拟机类加载

***虚拟机将编译得到的字节码文件加载到内存，经连接、解析等过程，得到可直接使用的 Java 类型的过程，称为类加载***。

***程序运行时期进行***



#### 1. 加载过程

类的生命周期如下图

![](Java学习之虚拟机/类的生命周期.jpg)

##### 1.1 加载

加载Class文件过程三个动作：

1. 通过**非数组类**的全限定名获取二进制字节流，但具体的获取途径并未规定（可以从ZIP包、网络、JSP文件等）；

   > *数组类直接由虚拟机加载，不经过类加载器*

2. 将字节流所代表的静态存储结构转换为方法区的运行时数据结构；

3. 在内存中生成类的访问入口对象 java.lang.Class。

   > *HotSpot 虚拟机将 Class 对象存放在方法区*



##### 1.2 验证

![](Java学习之虚拟机/验证阶段流程.jpg)

> 符号引用验证发生于解析阶段，说明**类加载过程各项“工作”是按先后顺序开始，但不一定是前一阶段结束才开始后一阶段，不同阶段的“工作”可并行**



##### 1.3 准备

在方法区为**类静态变量**分配内存并初始化为零值，**final**修饰的**类静态常量**赋给定的初值



##### 1.4 解析

（1）符号引用：存放于常量池中的常量，包括：

1. 类和接口的全限定名；

2. 字段名和描述符；

3. 方法名和描述符。

（2）解析过程：将符号引用解析、翻译，匹配对应的物理内存地址，即直接引用。



##### 1.5 初始化

初始化阶段，执行类构造器的 ` <clinit>() ` 方法

> 类变量赋值语句 && 静态代码块



#### 2、双亲委派模型

双亲委派模型的要点：

1. 子类加载器先请求父类加载器尝试加载；

2. 父类加载器无法加载时，子类加载器再尝试加载；

3. **所有的加载请求首先被启动类加载器处理**。

![](Java学习之虚拟机/双亲委派模型.jpg)



### 六、并发

#### 1. Java内存模型

> 定义了**线程共享的变量**从虚拟机内存中被读取或写入的底层实现
>
> 线程共享变量：实例字段、静态字段、构成数组对象的元素



#### 1.1 主内存与工作内存

- 工作内存中保存主内存中（共享）变量的拷贝副本；

- 工作内存中还保存了**线程私有的变量**；

- 线程对（共享）变量的读写操作必须经过工作内存才可与主内存交互；

- 线程间（共享）变量值的传递须借助主内存。

![](Java学习之虚拟机/内存模型图.jpg)



#### 1.2 内存间交互

|  操作  |     作用对象     |                     功能                      |
| :----: | :--------------: | :-------------------------------------------: |
|  lock  |  主内存中的变量  |              锁定状态，线程独占               |
| unlock |  主内存中的变量  |                 解除锁定状态                  |
|  read  |  主内存中的变量  |        将变量值传输至对应的工作内存中         |
|  load  | 工作内存中的变量 | read 操作后，将变量值放入工作内存的变量副本中 |
|  use   | 工作内存中的变量 |                  使用变量值                   |
| assign | 工作内存中的变量 |                     赋值                      |
| store  | 工作内存中的变量 |            将变量值传输至主内存中             |
| write  |  主内存中的变量  |    store 操作后，变量值存入主内存的变量中     |



#### 1.3 原子性、可见性和有序性

（1）原子性

1. read、load、use、assign、store、write 均为原子性的变量操作，实现**基本数据类型的访问和读写操作是原子性的**；
2. lock 和 unlock 操作，更大范围内实现原子性（**synchronized** 关键字）。



（2）可见性

1. **volatile** 关键字，**最轻量级的同步机制**，其修饰的变量对所有线程是可见的；
2. final 关键字；
3. **synchronized** 关键字，同步代码块。



（3）有序性

1. *线程内表现为串行的语义* ：在本线程内观察，操作是有序的；
2. *指令重排* & *工作内存与主内存同步延迟* ： 从另一个线程观察，被观察线程内的操作是无序的。



#### 2. Java线程

> 线程将进程的**资源分配**和**执行调度**分开，线程共享进程资源（内存地址、文件I/O等），线程独立被CPU调用、执行。

实现线程的三种方式：内核线程实现、用户线程实现、用户线程加轻量级进程的混合实现。



##### 2.1 内核线程实现

*内核线程（Kernel-Level Thread, KLT）*：由操作系统内核完成线程切换、调度、任务映射等操作，用户程序通过 *轻量级进程 （Light Weight Process, LWP)* 接口使用内核线程。

![](Java学习之虚拟机/轻量级线程与内核线程一对一的关系.jpg)

> 特点：系统调用代价高



##### 2.2 用户线程实现

*用户线程（User Thread, UT）* ：建立在用户空间的线程库，用户线程的建立、同步、调度和销毁在用户态中完成。

![](Java学习之虚拟机/进程与用户线程一对多的关系.jpg)

> 特点：用户态操作，代价小；实现过程异常复杂。



##### 2.3 用户线程加轻量级进程的混合实现

> 轻量级进程是用户线程与内核线程之间的桥梁

![](Java学习之虚拟机/用户线程与进程之间多对多的关系.jpg)



### 参考资料

[1] 《深入理解Java虚拟机》周志明 著

[2] https://blog.csdn.net/qq_41701956/article/details/81664921

[3] https://blog.csdn.net/wen7280/article/details/54428387

[4] https://www.jianshu.com/p/114bf4d9e59e

[5] https://www.jianshu.com/p/50d5c88b272dc