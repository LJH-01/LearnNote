[TOC]



# 你真的了解Java泛型参数？Spring ResolvableType更好的处理泛型

## Type

### **简介**

![img](https://storage.jd.com/shendengbucket1/2021-09-22-18-20-w5lpOvxxSxQv11F.png)

Type是Java 编程语言中所有类型的公共高级接口（官方解释），也就是Java中所有类型的“爹”；其中，“所有类型”的描述尤为值得关注。它并不是我们平常工作中经常使用的 int、String、List、Map等数据类型，而是从Java语言角度来说，对基本类型、引用类型向上的抽象；

Type体系中类型的包括：Class类型(原始类型，基本类型)、参数化类型(ParameterizedType)、数组类型(GenericArrayType)、类型变量(TypeVariable)、通配符类型（WildcardType）;

l 原始类型，不仅仅包含我们平常所指的类，还包括枚举、数组、注解等；

l 基本类型，也就是我们所说的java的基本类型，即int,float,double等

l 参数化类型，就是我们平常所用到的泛型List<String>、Map<String,Integer>这种；

l 类型变量，就是我们在定义泛型时使用到的T,U,K这些，例如Person<T extends Human>，这里的T就是类型变量

l 数组类型，并不是我们工作中所使用的数组String[] 、byte[]，而是参数化类型或者类型变量的数据，即T[] ，或者List<String>[];





### **Type及其子接口的来历**

泛型出现之前的类型

没有泛型的时候，只有原始类型。此时，所有的原始类型都通过字节码文件类Class类进行抽象。Class类的一个具体对象就代表一个指定的原始类型。

泛型出现之后的类型

泛型出现之后，扩充了数据类型。从只有原始类型扩充了参数化类型、类型变量类型、限定符类型 、泛型数组类型。

与泛型有关的类型不能和原始类型统一到Class的原因

产生泛型擦除的原因

原始类型和新产生的类型都应该统一成各自的字节码文件类型对象。但是由于泛型不是最初Java中的成分。如果真的加入了泛型，涉及到JVM指令集的修改，这是非常致命的。

Java中如何引入泛型

为了使用泛型又不真正引入泛型，Java采用泛型擦除机制来引入泛型。Java中的泛型仅仅是给编译器javac使用的，确保数据的安全性和免去强制类型转换的麻烦。但是，一旦编译完成，所有的和泛型有关的类型全部擦除。

Class不能表达与泛型有关的类型，因此，与泛型有关的参数化类型、类型变量类型、限定符类型 、泛型数组类型这些类型编译后全部被打回原形，在字节码文件中全部都是泛型被擦除后的原始类型，并不存在和自身类型对应的字节码文件。所以和泛型相关的新扩充进来的类型不能被统一到Class类中。

与泛型有关的类型在Java中的表示，为了通过反射操作这些类型以迎合实际开发的需要，Java就新增了ParameterizedType, TypeVariable, GenericArrayType, WildcardType几种类型来代表不能被归一到Class类中的类型但是又和原始类型齐名的类型。

引入Type的原因

为了程序的扩展性，最终引入了Type接口作为Class和ParameterizedType, TypeVariable, GenericArrayType, WildcardType这几种类型的总的父接口。这样可以用Type类型的参数来接受以上五种子类的实参或者返回值类型就是Type类型的参数。统一了与泛型有关的类型和原始类型Class

Type接口中没有方法的原因

从上面看到，Type的出现仅仅起到了通过多态来达到程序扩展性提高的作用，没有其他的作用。因此Type接口的源码中没有任何方法。

### **GenericArrayType**

泛型数组,组成数组的元素中有范型则实现了该接口; 它的组成元素是ParameterizedType或TypeVariable类型,它只有一个方法:

Type getGenericComponentType(): 返回数组的组成对象

### **ParameterizedType**

ParameterizedType

具体的泛型类型, 如Map<String, String>

有如下方法:

Type getRawType(): 返回承载该泛型信息的对象, 如上面那个Map<String, String>承载范型信息的对象是Map

Type[] getActualTypeArguments(): 返回实际泛型类型列表, 如上面那个Map<String, String>实际范型列表中有两个元素, 都是String

### **TypeVariable**

TypeVariable

泛型变量, 泛型信息在编译时会被转换为一个特定的类型, 而TypeVariable就是用来反映在JVM编译该泛型前的信息.

TypeVariable就是<T>、<C extends Collection>中的变量T、C本身; 它有如下方法:

Type[] getBounds(): 获取类型变量的上边界, 若未明确声明上边界则默认为Object

getGenericDeclaration(): 获取声明该类型变量的类型

String getName(): 获取在源码中定义时的名字

注意:

类型变量在定义的时候只能使用extends进行(多)边界限定, 不能用super;

为什么边界是一个数组? 因为类型变量可以通过&进行多个上边界限定，因此上边界有多个

### **WildcardType**

WildcardType

该接口表示通配符泛型, 比如? extends Number 和 ? super Integer 它有如下方法:

Type[] getUpperBounds(): 获取范型变量的上界

Type[] getLowerBounds(): 获取范型变量的下界

注意:现阶段通配符只接受一个上边界或下边界, 返回数组是为了以后的扩展, 实际上现在返回的数组的大小是1

例子：

```java
public class TypeTest<K extends Comparable & Serializable> {

    /**
     * GenericArrayType
     * 泛型数组,组成数组的元素中有范型则实现了该接口; 它的组成元素是ParameterizedType或TypeVariable类型;
     * Type getGenericComponentType(): 返回数组的组成对象;
     * @see java.util.List<java.lang.String>
     */
    List<String>[] lists;

    /**
     * ParameterizedType
     * 具体的泛型类型
     * Type getRawType(): 返回承载该泛型信息的对象;
     * @see java.util.Map
     * Type[] getActualTypeArguments(): 返回实际泛型类型列表;
     * @see java.lang.String
     * @see java.lang.Integer
     */
    Map<String, Integer> map;

    /**
     * TypeVariable
     * 泛型变量, 泛型信息在编译时会被转换为一个特定的类型, 而TypeVariable就是用来反映在JVM编译该泛型前的信息.
     * Type[] getBounds(): 获取类型变量的上边界, 若未明确声明上边界则默认为Object;
     * @see java.lang.Comparable
     * @see java.io.Serializable
     * getGenericDeclaration(): 获取声明该类型变量的类型;
     * @see com.ljh.type.test2.TypeTest
     * String getName(): 获取在源码中定义时的名字;
     * @see K
     * 注意:
     * 类型变量在定义的时候只能使用extends进行(多)边界限定, 不能用super;
     * 为什么边界是一个数组? 因为类型变量可以通过&进行多个上边界限定，因此上边界有多个
     */
    K key;

    /**
     * WildcardType
     * 该接口表示通配符泛型, 比如? extends Number 和 ? super Integer 它有如下方法:
     * Type[] getUpperBounds(): 获取范型变量的上界;
     * @see java.lang.Number
     * Type[] getLowerBounds(): 获取范型变量的下界;
     * @see java.lang.String
     * 注意:
     * 现阶段通配符只接受一个上边界或下边界, 返回数组是为了以后的扩展, 实际上现在返回的数组的大小是1
     */
    private List<? extends Number> numbers;
    private List<? super String> s;

    public static void main(String[] args) {
        // 尝试获取数组lists的填充数据类型
        Class clz = TypeTest.class;
        try {

            System.out.println("=====testGenericArrayType=====");
            testGenericArrayType(clz);

            System.out.println("=====testParameterizedType=====");
            testParameterizedType(clz);

            System.out.println("=====testTypeVariable=====");
            testTypeVariable(clz);

            System.out.println("=====testWildcardType=====");
            testWildcardType(clz);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }

    private static void testWildcardType(Class clz) throws NoSuchFieldException {
        Field nf = clz.getDeclaredField("numbers");
        Field af = clz.getDeclaredField("s");

        //获取泛型参数类型
        ParameterizedType nftype = (ParameterizedType) nf.getGenericType();
        ParameterizedType aftype = (ParameterizedType) af.getGenericType();
        //获取泛型通配符类型
        WildcardType nfwType = (WildcardType) nftype.getActualTypeArguments()[0];
        System.out.println(nfwType.getUpperBounds()[0]);
        // System.out.println(nfwType.getLowerBounds()[0]);//运行报错，extends限定的是上界，没有下界

        WildcardType afwType = (WildcardType) aftype.getActualTypeArguments()[0];
        // System.out.println(afwType.getUpperBounds()[0]);//运行报错，super限定的是下界，没有上界
        System.out.println(afwType.getLowerBounds()[0]);
    }

    private static void testTypeVariable(Class clz) throws NoSuchFieldException {
        Field fk = clz.getDeclaredField("key");

        TypeVariable typeVariableK = (TypeVariable) fk.getGenericType();

        // 获取声明该属性的类型
        // class com.ljh.type.test2.TypeTest
        GenericDeclaration kDecla = typeVariableK.getGenericDeclaration();
        System.out.println(kDecla);

        // 获取该属性在源码中定义时的属性类型的名字
        // K
        String name = typeVariableK.getName();
        System.out.println(name);

        // 获取类型变量的上边界, 若未明确声明上边界则默认为Object
        // interface java.lang.Comparable
        // interface java.io.Serializable
        Type[] kbounds = typeVariableK.getBounds();
        for (Type kbound : kbounds) {
            System.out.println(kbound);
        }
    }

    private static void testParameterizedType(Class clz) throws NoSuchFieldException {
        Field field = clz.getDeclaredField("map");
        // java.util.Map<java.lang.String, java.lang.Integer>
        ParameterizedType parameterizedType = (ParameterizedType) field.getGenericType();
        System.out.println(parameterizedType);

        // 获取承载泛型的对象的数据类型
        // interface java.util.Map
        Type rawType = parameterizedType.getRawType();
        System.out.println(rawType);

        // 获取泛型的实际类型
        // class java.lang.String
        // java.lang.String
        // class java.lang.Integer
        // java.lang.Integer
        Type[] typeArguments = parameterizedType.getActualTypeArguments();
        for (Type typeArgument : typeArguments) {
            System.out.println(typeArgument);
            System.out.println(typeArgument.getTypeName());
        }
    }

    private static void testGenericArrayType(Class clz) throws NoSuchFieldException {
        Field field = clz.getDeclaredField("lists");
        // 获取填充数组的数据类型
        GenericArrayType type = (GenericArrayType) field.getGenericType();

        // 泛型数组的声明类型，包含泛型信息
        // java.util.List<java.lang.String>
        Type arraytype = type.getGenericComponentType();
        System.out.println(arraytype);

        // 获取填充数组的数据类型，如果数据类型是泛型，那么获取的是承载泛型的对象的数据类型
        // interface java.util.List
        ParameterizedType parameterizedType = (ParameterizedType) arraytype;
        System.out.println(parameterizedType.getRawType());

        // 获取泛型的实际类型
        // class java.lang.String
        Type[] typeArguments = parameterizedType.getActualTypeArguments();
        for (Type typeArgument : typeArguments) {
            System.out.println(typeArgument);
        }
    }
}
```



## **ResolvableType**

在学习了Java的Type体系后，我们会发现，依赖于整个Type体系去处理泛型代码非常的繁琐，并且不易于理解。基于这种情况，Spring开发了一个`ResolvableType`类，这个类对整个Type体系做了系统的封装。

ResolvableType为所有的java类型提供了统一的数据结构以及API，换句话说，一个ResolvableType对象就对应着一种java类型。我们可以通过ResolvableType对象获取类型携带的信息（举例如下）：

  1.getSuperType()：获取直接父类型

  2.getInterfaces()：获取接口类型

  3.getGeneric(int...)：获取类型携带的泛型类型

  4.resolve()：Type对象到Class对象的转换

ResolvableType的构造方法全部为私有的，我们不能直接new，只能使用其提供的静态方法进行类型获取：

 1.forField(Field)：获取指定字段的类型

 2.forMethodParameter(Method, int)：获取指定方法的指定形参的类型

 3.forMethodReturnType(Method)：获取指定方法的返回值的类型

 4.forClass(Class)：直接封装指定的类型

```java
class Parent<T,Y>{
	String string;
}
class Son1 extends Parent<String,Integer>{

}
class Son2 extends Parent<String, BigDecimal>{

}
public class SpringResolvableTypeGenericClass<K extends Comparable & Serializable, V> extends Parent{

	private List<String> listString;
	private List<String>[] listLists;
	private List<? extends Number> numbers;
	private Map<String, Long> maps;

	private Parent<String,Integer> parent;

	K key;
	V value;

	public Map<String, Long> getMaps() {
		return maps;
	}

	/**
	 * 总结一句话就是使用起来非常的简单方便，更多超级复杂的可以参考spring 源码中的测试用例：ResolvableTypeTests
	 * 其实这些的使用都是在Java的基础上进行使用的哦！
	 * Type是Java 编程语言中所有类型的公共高级接口（官方解释），也就是Java中所有类型的“爹”；其中，“所有类型”的描述尤为值得关注。它并不是我们平常工作中经常使用的 int、String、List、Map等数据类型，
	 * 而是从Java语言角度来说，对基本类型、引用类型向上的抽象；
	 * Type体系中类型的包括：原始类型(Class)、参数化类型(ParameterizedType)、数组类型(GenericArrayType)、类型变量(TypeVariable)、基本类型(Class);
	 * 原始类型，不仅仅包含我们平常所指的类，还包括枚举、数组、注解等；
	 * 参数化类型，就是我们平常所用到的泛型List、Map；
	 * 数组类型，并不是我们工作中所使用的数组String[] 、byte[]，而是带有泛型的数组，即T[] ；
	 * 基本类型，也就是我们所说的java的基本类型，即int,float,double等
	 *
	 * @param args
	 */
	/**
	 * ResolvableType为所有的java类型提供了统一的数据结构以及API ，换句话说，一个ResolvableType对象就对应着一种java类型。
	 *
	 * 我们可以通过ResolvableType对象获取类型携带的信息，举例如下：
	 *
	 * getSuperType()：获取直接父类型，返回ResolvableType
	 * getInterfaces()：获取接口类型， 返回ResolvableType[]数组
	 * getGeneric(int…)：获取类型携带的泛型类型，返回ResolvableType[]数组
	 * resolve()：Type对象到Class对象的转换
	 * 另外，ResolvableType的构造方法全部为私有的，我们不能直接new，只能使用其提供的静态方法进行类型获取：
	 *
	 * forField(Field)：获取指定字段的类型
	 * forMethodParameter(Method, int)：获取指定方法的指定形参的类型
	 * forMethodReturnType(Method)：获取指定方法的返回值的类型
	 * forClass(Class)：直接封装指定的类型
	 * @param args
	 */
	public static void main(String[] args) {
		ReflectionUtils.findField(SpringResolvableTypeGenericClass.class, "string", null);
		// doTestFindListStr();
		// doTestFindlistLists();
		doTestFindNumbers();
		doTestFindK();
		doTestFindMaps();
		doTestFindReturn();
	}


	public static void doTestFindListStr() {
		ResolvableType listStringResolvableType = ResolvableType.forField(ReflectionUtils.findField(
				SpringResolvableTypeGenericClass.class, "listString"));

		// 获取第0个位置的参数泛型
		// class java.lang.String
		Class<?> resolve = listStringResolvableType.getGeneric(0).resolve();
		System.out.println(resolve);

	}

	static void doTestFindlistLists() {
		ResolvableType listListsResolvableType = ResolvableType.forField(ReflectionUtils.findField(
				SpringResolvableTypeGenericClass.class, "listLists"));

		// 获取第0个位置的参数泛型
		// interface java.util.List
		Class<?> resolve = listListsResolvableType.getGeneric(0).resolve();
		System.out.println(resolve);

		// class java.lang.String
		resolve = listListsResolvableType.getGeneric(0).getGeneric(0).resolve();
		System.out.println(resolve);
		// class java.lang.String
		resolve = listListsResolvableType.getGeneric(0, 0).resolve();
		System.out.println(resolve);

		ResolvableType[] resolvableTypes = listListsResolvableType.getGenerics();
		System.out.println("begin 遍历");
		// interface java.util.List
		for (ResolvableType resolvableType : resolvableTypes) {
			resolve = resolvableType.resolve();
			System.out.println(resolve);
		}
		System.out.println("end 遍历");
	}

	static void doTestFindNumbers() {
		ResolvableType listResolvableType = ResolvableType.forField(ReflectionUtils.findField(
				SpringResolvableTypeGenericClass.class, "numbers"));

		// 获取第0个位置的参数泛型
		// class java.lang.Number
		Class<?> resolve = listResolvableType.getGeneric(0).resolve();
		System.out.println(resolve);


		ResolvableType[] resolvableTypes = listResolvableType.getGenerics();
		System.out.println("begin 遍历");
		// class java.lang.Number
		for (ResolvableType resolvableType : resolvableTypes) {
			resolve = resolvableType.resolve();
			System.out.println(resolve);
		}
		System.out.println("end 遍历");
	}

	/**
	 * private Map<String, Long> maps;
	 */
	public static void doTestFindMaps() {
		ResolvableType mapsResolvableType = ResolvableType.forField(ReflectionUtils.findField(
				SpringResolvableTypeGenericClass.class, "maps"));

		System.out.println("begin 遍历");
		ResolvableType[] resolvableTypes = mapsResolvableType.getGenerics();
		Class<?> resolve = null;
		// class java.lang.String
		// class java.lang.Long
		for (ResolvableType resolvableType : resolvableTypes) {
			resolve = resolvableType.resolve();
			System.out.println(resolve);
		}
		System.out.println("end 遍历");
	}

	public static void doTestFindK() {
		ResolvableType KResolvableType = ResolvableType.forField(ReflectionUtils.findField(
				SpringResolvableTypeGenericClass.class, "key"));

		// 获取第0个位置的参数泛型
		// null
		Class<?> resolve = KResolvableType.getGeneric(0).resolve();
		System.out.println(resolve);

		ResolvableType[] resolvableTypes = KResolvableType.getGenerics();
		System.out.println("begin 遍历");
		// null
		for (ResolvableType resolvableType : resolvableTypes) {
			resolve = resolvableType.resolve();
			System.out.println(resolve);
		}
		System.out.println("end 遍历");
	}

	/**
	 * Map<String, Long>
	 */
	public static void doTestFindReturn() {
		ResolvableType resolvableType = ResolvableType.forMethodReturnType(ReflectionUtils.findMethod(SpringResolvableTypeGenericClass.class, "getMaps"));

		System.out.println("begin 遍历");
		ResolvableType[] resolvableTypes = resolvableType.getGenerics();
		Class<?> resolve = null;
		// class java.lang.String
		// class java.lang.Long
		for (ResolvableType resolvableTypeItem : resolvableTypes) {
			resolve = resolvableTypeItem.resolve();
			System.out.println(resolve);
		}
		System.out.println("end 遍历");


	}
}

class SpringTypeTest {

	/**
	 * 测试继承获取泛型
	 * @param
	 */
	class Parent<T, S>{

	}

	interface IParent1<T> {

	}

	interface IParent2<T> {

	}

	public class Children<K extends String, V extends Collection> extends Parent<String, Boolean> implements IParent1<Long>, IParent2<Double> {

	}

	public static void main(String[] args) {
		System.out.println("------------------------jdk原生方式-获取泛型父类---------------------------------------");
		// jdk原生方式获取Children类继承的父类的类型
		Type genericSuperclassType = Children.class.getGenericSuperclass();
		// com.calebzhao.test.SpringTypeTest$Parent
		System.out.println(genericSuperclassType);

		if (genericSuperclassType instanceof ParameterizedType) {
			Type[] actualTypeArguments = ((ParameterizedType) genericSuperclassType)
					.getActualTypeArguments();
			for (Type argumentType : actualTypeArguments) {
				System.out.println("父类ParameterizedType.getActualTypeArguments:" + argumentType);
			}
		}

		System.out.println("------------------------jdk原生方式-获取泛型父接口---------------------------------------");


		// jdk原生方式获取Children类实现的接口的类型
		Type[] genericInterfacesTypes = Children.class.getGenericInterfaces();
		// [com.calebzhao.test.SpringTypeTest$IParent1, com.calebzhao.test.SpringTypeTest$IParent2]
		System.out.println(Arrays.toString(genericInterfacesTypes));

		for (Type interfaceType : genericInterfacesTypes) {
			if (interfaceType instanceof ParameterizedType) {
				Type[] actualTypeArguments = ((ParameterizedType) interfaceType)
						.getActualTypeArguments();
				for (Type argumentType : actualTypeArguments) {
					System.out.println("父接口ParameterizedType.getActualTypeArguments:" + argumentType);
				}
			}
		}


		System.out.println("------------------------分割线 spring ResolvableType---------------------------------------");
		ResolvableType childrenResolvableType = ResolvableType.forClass(Children.class);
		System.out.println("children type：" + childrenResolvableType.getType());
		System.out.println("children raw type：" + childrenResolvableType.getRawClass());
		System.out.println("children generics：" + Arrays.toString(childrenResolvableType.getGenerics()));

		System.out.println("-----super ResolvableType-------");
		ResolvableType superResolvableType = childrenResolvableType.getSuperType();
		System.out.println("super generics：" + Arrays.toString(superResolvableType.getGenerics()));
		System.out.println("super type：" + superResolvableType.getType());
		System.out.println("super raw class：" + superResolvableType.getRawClass());
		System.out.println("super getComponentType：" +  superResolvableType.getComponentType());
		System.out.println("super getSource：" +  superResolvableType.getSource());
		System.out.println("super：" + Arrays.toString(superResolvableType.getInterfaces()));

		System.out.println("\n-----interface ResolvableType-------");
		ResolvableType[] interfaceResolvableTypes = childrenResolvableType.getInterfaces();
		IntStream.range(0, interfaceResolvableTypes.length).forEach(index ->{
			ResolvableType interfaceResolvableType = interfaceResolvableTypes[index];

			System.out.println("\n -------第" + index + "个接口------------");
			System.out.println("interface generics：" + Arrays.toString(interfaceResolvableType.getGenerics()));
			System.out.println("interface type：" + interfaceResolvableType.getType());
			System.out.println("interface raw class：" + interfaceResolvableType.getRawClass());
			System.out.println("interface getComponentType：" +  interfaceResolvableType.getComponentType());
			System.out.println("interface getSource：" +  interfaceResolvableType.getSource());
			System.out.println("interface：" + Arrays.toString(interfaceResolvableType.getInterfaces()));
		});
	}
}
```

## Spring中使用处理泛型

Spring中ApplicationListener<E extends ApplicationEvent>使用泛型与事件类型匹配确定调用哪个监听器以及使用@Autowired、@Resource进行属性注入时使用泛型匹配注入的bean。