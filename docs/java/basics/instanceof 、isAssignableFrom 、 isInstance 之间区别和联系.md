# instanceof 、isAssignableFrom 、 isInstance 之间区别和联系



1. instanceof：用法：c instanceof Parent ，实例c是否为Parent类的子类或子接口。 
2. isInstance：用法： Face.class.isInstance(c) ， Face类是否为实例c的父类或父接口 ---- 和 instanceof 关键字判断刚刚好相反。
3. isAssignableFrom：用法：Parent.class.isAssignableFrom(Child.class) ，Parent类是否为Child类的父类或父接口。
4. instanceof 、isAssignableFrom 、 isInstance 都是判断Java中类之间的关系（继承类、实现接口）； instanceof 判断是左边是不是右边的儿子 ； isInstance 和 isAssignableFrom 判断的左边是不是右边的爸爸。
5. instanceof 和 isInstance 都是用来判断 类（接口）和实例之间是否存在关系。
6. isAssignableFrom 判断类和类，接口和接口，类和接口 之间是否存在关系。



## 参考

https://blog.csdn.net/HaHa_Sir/article/details/120084092