[TOC]



# Java知识点



## instanceof 、isAssignableFrom 、 isInstance 之间区别和联系



1. instanceof：用法：c instanceof Parent ，实例c是否为Parent类的子类或子接口。 
2. isInstance：用法： Face.class.isInstance(c) ， Face类是否为实例c的父类或父接口 ---- 和 instanceof 关键字判断刚刚好相反。
3. isAssignableFrom：用法：Parent.class.isAssignableFrom(Child.class) ，Parent类是否为Child类的父类或父接口。
4. instanceof 、isAssignableFrom 、 isInstance 都是判断Java中类之间的关系（继承类、实现接口）； instanceof 判断是左边是不是右边的儿子 ； isInstance 和 isAssignableFrom 判断的左边是不是右边的爸爸。
5. instanceof 和 isInstance 都是用来判断 类（接口）和实例之间是否存在关系。
6. isAssignableFrom 判断类和类，接口和接口，类和接口 之间是否存在关系。

## HotSpot中字符串常量池保存哪里？永久代？方法区还是堆区?

### 1.8Java内存区域

![image-20230518103655019](assets/image-20230518103655019.png)

运行时常量池（Runtime Constant Pool）是虚拟机规范中是方法区的一部分，在加载类和结构到虚拟机后，就会创建对应的运行时常量池；而字符串常量池是这个过程中常量字符串的存放位置。所以从这个角度，字符串常量池属于虚拟机规范中的方法区，它是一个**逻辑上的概念**；而堆区，永久代以及元空间是实际的存放位置。

不同的虚拟机对虚拟机的规范（比如方法区）是不一样的，只有 HotSpot 才有永久代的概念。

HotSpot也是发展的，由于[一些问题](http://openjdk.java.net/jeps/122)的存在，HotSpot考虑逐渐去永久代，对于不同版本的JDK，**实际的存储位置**是有差异的，具体看如下表格：

![image-20230518103821336](assets/image-20230518103821336.png)

| JDK版本      | 是否有永久代，字符串常量池放在哪里？                         | 方法区逻辑上规范，由哪些实际的部分实现的？                   |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| jdk1.6及之前 | 有永久代，运行时常量池（包括字符串常量池），静态变量存放在永久代上 | 这个时期方法区在HotSpot中是由永久代来实现的，以至于**这个时期说方法区就是指永久代** |
| jdk1.7       | 有永久代，但已经逐步“去永久代”，字符串常量池、静态变量移除，保存在堆中； | 这个时期方法区在HotSpot中由**永久代**（类型信息、字段、方法、常量）和**堆**（字符串常量池、静态变量）共同实现 |
| jdk1.8及之后 | 取消永久代，类型信息、字段、方法、常量保存在本地内存的元空间，但字符串常量池、静态变量仍在堆中 | 这个时期方法区在HotSpot中由本地内存的**元空间**（类型信息、字段、方法、常量）和**堆**（字符串常量池、静态变量）共同实现 |



## 多线程

### 线程中断

1. java.lang.Thread#interrupt方法：实例方法，中断该线程，实际上只是给线程设置一个中断标志。线程如果正在运行仍会继续运行；如使用了sleep,同步锁的wait,socket中的receiver,accept等方法时，会抛出InterruptException异常
2. java.lang.Thread#interrupted方法：类方法，返回一个boolean并清除中断状态，第二次再调用时中断状态已经被清除，将返回一个false。
ReentrantLock 中 parkAndCheckInterrupt() 为会么会调用Thread.interrupted()？
3. LockSupport.park()，可通过两种方式被唤醒，LockSupport.unpark() 或者 interrupt()，若有中断标志，park就不阻塞线程，所以JDK中被唤醒之后都会调用Thread#interrupted方法来清楚中断状态。




## 实例对象-Class实例对象-元数据的区别联系

1. **元数据**：JVM**加载**一个类的时候会创建一个**instanceKlass**，用来表示这个**类的元数据**，包括常量池、字段、方法等。存放在**方法区**。
2. **实例对象** ：在**new一个对象**时，jvm创建**instanceOopDesc**，来表示**这个对象**，存放在**堆区**，其引用，存放在栈区；平时说的Java 对象实例就是instanceOopDesc，它用来表示对象的实例信息；
3. **Class实例对象**：HotSpot并没有把instanceKlass直接暴露给Java，而会另外创建对应的instanceOopDesc来表示java.lang.Class对象，并将后者称为前者的“Java镜像”，**instanceKlass持有指向对应的Class对象-即instanceOopDesc的引用**。

**instanceOopDesc**包含三部分：

1. 对象头，也叫Mark Word，主要存储对象运行时记录信息，如hashcode, GC分代年龄，锁状态标志，线程ID，时间戳等;
2. 元数据指针，即指向方法区的**instanceKlass**实例
3. 实例数据;
4. 另外，如果是数组对象，还多了一个数组长度

**Person类的实例(Oop)-->Person类的元数据（instanceKlass）-->Person类的Class对象（前提：同一个类加载器）**

- 每一个类的实例可以有多个，每个实例对象对应的Oop中对**instanceKlass**的引用都是同一个。
- 每一个类的元数据instanceKlass只有一个
- 每一个类的Class对象（Oop）也只有一个
- 唯一的Klass与唯一的Class对象一一对应，互相保存着对彼此的引用。

这些**引用关系**支持着**获取class对象三种方式的实现**

- Class.forName("ClassName")：通过类的元数据中的Class对象引用获得Class对象
-  object.getClass()：通过实例对象中保存的对类的元数据的引用获取类的元数据（instanceKlass），通过instanceKlass中对Class对象的引用获取Class对象（instanceOopDesc）
- 类名.class：通过类的元数据中的Class对象引用获得Class对象



## Synchronized锁优化

![image-20230520001123057](assets/image-20230520001123057.png)




## Java的增强技术

如下图，从软件的开发周期来看，可织入埋点的时机主要有 3 个阶段：编译期、编译后和运行期。

![image-20230613172245043](assets/image-20230613172245043.png)

### 编译期

这里的编译期指将Java源文件编译为class字节码的过程。Java编译器提供了基于 JSR 269 规范[1]的注解处理器机制，通过操作AST （抽象语法树，Abstract Syntax Tree，下同）实现逻辑的织入。业内有不少基于此机制的应用，比如Lombok 、MapStruct 、JPA 等；此机制的优点是因为在编译期执行，可以将问题前置，没有多余依赖，因此做出来的工具使用起来比较方便。缺点也很明显，要熟练操作 AST并不是想的那么简单，不理解前后关联的流程写出来的代码不够稳定，因此要花大量时间熟悉编译器底层原理。当然这个过程对使用者来讲是没有感知的。

### 编译后

编译后是指编译成 class 字节码之后，通过字节码进行增强的过程。此阶段插桩需要适配不同的构建工具：Maven、Gradle、Ant、Ivy等，也需要使用方增加额外的构建配置，因此存在开发量大和使用不够方便的问题，首先要排除掉此选项。可能只有极少数场景下才会需要在此阶段插桩。

### 运行期

运行期是指在程序启动后，在运行时进行增强的过程，这个阶段有 3 种方式可以织入逻辑，按照启动顺序，可以分为：静态 Agent、AOP 和动态 Agent。

#### 静态 Agent

JVM 启动时使用 -javaagent 载入指定 jar 包，调用 MANIFEST.MF 文件里的 Premain-Class 类的 premain 方法触发织入逻辑。是技术中间件最常使用的方式，借助字节码工具完成相关工作。应用此机制的中间件有很多，比如：京东内部的链路监控 pfinder、外部开源的 skywalking 的探针、阿里的 TTL 等等。这种方式优点是整体比较成熟，缺点主要是兼容性问题，要测试不同的 JDK 版本代价较大，出现问题只能在线上发现。同时如果不是专业的中间件团队，还是存在一定的技术门槛，维护成本比较高；

#### Spring AOP

Spring AOP大家都不陌生，通过 Spring 代理机制，可以在方法调用前后织入逻辑。AOP 最大的优点是使用简单，同样存在不少缺点：

1. 需要两次反射，一次是增强里面的反射，一次是需要执行方法的反射
2. 私有方法、静态方法、final class和方法等场景无法走切面

#### 动态 Agent

动态加载jar包，调用MANIFEST.MF文件中声明的Agent-Class类的agentmain方法触发织入逻辑。这种方式主要用来线上动态调试，使用此机制的中间件也有很多，比如：Btrace、Arthas等，此方式不适合常驻内存使用，因此要排除掉。

### 最终方案

选择通过上面的分析梳理可知，要实现重复代码的抽象有 3 种方式：基于JSR 269 的插桩、基于 Java Agent 的字节码增强、基于Spring AOP的自定义切面。接下来进一步的对比：



![image-20230613172427353](assets/image-20230613172427353.png)



如上表所示，从实现成本上来看，AOP 最简单，但这个方案不能覆盖所有场景，存在一定的局限性，不符合我们追求极致的调性，因此首先排除。Java Agent 能达到的效果与 JSR 269 相同，但是启动参数里需要增加 -javaagent 配置，有少量的运维工作，同时还有 JDK 兼容性的坑需要趟，对非中间件团队来说，这种方式从长久看会带来负担，因此也要排除。

基于 JSR 269 的插桩方式，对Java编译器工作流程的理解和 AST 的操作会带来实现上的复杂性，前期投入比较大，但是组件一旦成型，会带来一劳永逸的解决方案，可以很自信的讲，插桩实现的组件是监控埋点场景里的银弹（事实证明了这点，不然也不敢这么吹）。













## 参考

https://blog.csdn.net/HaHa_Sir/article/details/120084092

https://pdai.tech/md/java/basic/java-basic-lan-basic.html#string-intern

https://blog.csdn.net/weixin_44250483/article/details/121338111

https://blog.csdn.net/qq_45795744/article/details/123493673

https://zhuanlan.zhihu.com/p/548844443