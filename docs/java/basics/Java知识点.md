# Java知识点



## instanceof 、isAssignableFrom 、 isInstance 之间区别和联系



1. instanceof：用法：c instanceof Parent ，实例c是否为Parent类的子类或子接口。 
2. isInstance：用法： Face.class.isInstance(c) ， Face类是否为实例c的父类或父接口 ---- 和 instanceof 关键字判断刚刚好相反。
3. isAssignableFrom：用法：Parent.class.isAssignableFrom(Child.class) ，Parent类是否为Child类的父类或父接口。
4. instanceof 、isAssignableFrom 、 isInstance 都是判断Java中类之间的关系（继承类、实现接口）； instanceof 判断是左边是不是右边的儿子 ； isInstance 和 isAssignableFrom 判断的左边是不是右边的爸爸。
5. instanceof 和 isInstance 都是用来判断 类（接口）和实例之间是否存在关系。
6. isAssignableFrom 判断类和类，接口和接口，类和接口 之间是否存在关系。

## HotSpot中字符串常量池保存哪里？永久代？方法区还是堆区?

运行时常量池（Runtime Constant Pool）是虚拟机规范中是方法区的一部分，在加载类和结构到虚拟机后，就会创建对应的运行时常量池；而字符串常量池是这个过程中常量字符串的存放位置。所以从这个角度，字符串常量池属于虚拟机规范中的方法区，它是一个**逻辑上的概念**；而堆区，永久代以及元空间是实际的存放位置。

不同的虚拟机对虚拟机的规范（比如方法区）是不一样的，只有 HotSpot 才有永久代的概念。

HotSpot也是发展的，由于[一些问题](http://openjdk.java.net/jeps/122)的存在，HotSpot考虑逐渐去永久代，对于不同版本的JDK，**实际的存储位置**是有差异的，具体看如下表格：

| JDK版本      | 是否有永久代，字符串常量池放在哪里？                         | 方法区逻辑上规范，由哪些实际的部分实现的？                   |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| jdk1.6及之前 | 有永久代，运行时常量池（包括字符串常量池），静态变量存放在永久代上 | 这个时期方法区在HotSpot中是由永久代来实现的，以至于**这个时期说方法区就是指永久代** |
| jdk1.7       | 有永久代，但已经逐步“去永久代”，字符串常量池、静态变量移除，保存在堆中； | 这个时期方法区在HotSpot中由**永久代**（类型信息、字段、方法、常量）和**堆**（字符串常量池、静态变量）共同实现 |
| jdk1.8及之后 | 取消永久代，类型信息、字段、方法、常量保存在本地内存的元空间，但字符串常量池、静态变量仍在堆中 | 这个时期方法区在HotSpot中由本地内存的**元空间**（类型信息、字段、方法、常量）和**堆**（字符串常量池、静态变量）共同实现 |































## 参考

https://blog.csdn.net/HaHa_Sir/article/details/120084092

https://pdai.tech/md/java/basic/java-basic-lan-basic.html#string-intern