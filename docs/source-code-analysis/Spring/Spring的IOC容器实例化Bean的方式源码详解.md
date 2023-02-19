[TOC]

# Spring实例化Bean的方式

## 使用方式

### XML配置

#### 1.普通方式：无参构造方式

```java
<bean id="mdcInterceptor" class="com.man.interceptor.MDCInterceptor"/>
```

#### 2.有参构造方式

```java
<bean id="rpcExecutor" class="com.common.rpc.RpcExecutor">
    <constructor-arg index="0" value="${executor.prefix}"/>
    <constructor-arg index="1" value="${app.name}"/>
</bean>
```

#### 3.静态工厂方式

```java
<bean id="springDataEsTransportClient"
      class="com.common.es.base.SpringDataEsTransportClient"
      factory-method="getInstance">
    <constructor-arg index="0" value="${es.spring.data.elasticsearch.cluster-nodes}"/>
</bean>
```

其中getInstance方法是静态方法

#### 4.工厂方式

```java
<bean id="taskThreadPoolExecutor"
      factory-bean="taskThreadPoolFactory" factory-method="getInstance">
</bean>
```

其中taskThreadPoolFactory是单例bean, getInstance方法是实例方法

#### 5.FactoryBean方式

```java
<bean id="masterSqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="multiDataSource"/>
    <property name="configLocation" value="classpath:mybatis-config.xml"/>
</bean>
```

### 注解配置

#### 1.无参构造方式

使用@Component注解：比如@Controller，@Service，@Repository

```java
@Service
public class AutoWireTestService1 {
}
```

#### 2.有参构造方式

```java
@Controller
public class AutowireTestController {
   @Autowired
   public AutowireTestController(AutoWireTestService2 autoWireTestService2) {
      this.autoWireTestService2=autoWireTestService2;
   }
}
```

#### 3.静态工厂方式

```java
    @Bean
   public static AutoWireTestService3 autoWireTestService3() {
      return new AutoWireTestService3();
   }
}
```

#### 4.工厂方式

```java
@Bean
public AutoWireTestService2 autoWireTestService2() {
   return new AutoWireTestService2();
}
```

#### 5.FactoryBean方式

```java
@Bean
public FactoryBean<UserInfo> userBean() {
   return new FactoryBean<UserInfo>() {
      @Override
      public UserInfo getObject(){
         return new UserInfo();
      }

      @Override
      public Class<?> getObjectType() {
         return UserInfo.class;
      }
   };
}
```

总结：XML配置和注解配置除了有参构造的注入有一点点区别（BeanDefinition中要注入的属性存放的位置不同）以外其他完全相同

## 源码分析

### 实例化过程

调用AbstractAutowireCapableBeanFactory#doCreateBean来实例化以及初始化类，本文只分析实例化不分析实例化之后的初始化（包含注入属性，以及后置处理器操作等）

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean.用来存放bean+其他属性
   BeanWrapper instanceWrapper = null;
   // 如果是单例的话，则先把缓存中的同名bean清除
   if (mbd.isSingleton()) {
      // 如果是FactoryBean，则需要先移除未完成的FactoryBean实例的缓存
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
   }
   // 完成通过构造函数初始化bean的操作；
   if (instanceWrapper == null) {
      if(logger.isInfoEnabled()){
         logger.info("通过构造方法反射调用实例化bean，beanName："+beanName);
      }
      // 说明不是 FactoryBean，这里实例化 Bean
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }else{
      if(logger.isInfoEnabled()){
         logger.info("该实例化的bean是FactoryBean，因此从factoryBeanInstanceCache中拿到该bean，beanName为："+beanName);
      }
   }
   。。。。。。

   return exposedObject;
}
```

之后调用AbstractAutowireCapableBeanFactory的createBeanInstance来反射实例化Bean

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
   // Make sure bean class is actually resolved at this point.
   Class<?> beanClass = resolveBeanClass(mbd, beanName);

   // class如果不是public的，则抛出异常。因为没法进行实例化
   if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
   }

   Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
   if (instanceSupplier != null) {
      return obtainFromSupplier(instanceSupplier, beanName);
   }

   // @Bean标注的bean方法,或者使用FactoryBean的factory-method来创建，支持静态工厂和实例工厂，
   if (mbd.getFactoryMethodName() != null) {
      if(logger.isInfoEnabled()){
         logger.info("该bean是@Bean标注的方法，其中beanName为："+beanName);
      }
      return instantiateUsingFactoryMethod(beanName, mbd, args);
   }

   // Shortcut when re-creating the same bean...
   // 无参数情况时，创建bean。调用无参构造方法
   boolean resolved = false;
   boolean autowireNecessary = false;
   if (args == null) {
      synchronized (mbd.constructorArgumentLock) {
         // 初次 resolvedConstructorOrFactoryMethod 为 null
         if (mbd.resolvedConstructorOrFactoryMethod != null) {
            // 2.1 如果resolvedConstructorOrFactoryMethod缓存不为空，则将resolved标记为已解析
            resolved = true;
            // 2.2 根据constructorArgumentsResolved判断是否需要自动注入
            autowireNecessary = mbd.constructorArgumentsResolved;
         }
      }
   }
   // 3.如果已经解析过，则使用resolvedConstructorOrFactoryMethod缓存里解析好的构造函数方法
   if (resolved) {
      if (autowireNecessary) {
         // 通过构造函数初始化
         return autowireConstructor(beanName, mbd, null, null);
      }
      else {
         // 使用默认构造函数初始化
         return instantiateBean(beanName, mbd);
      }
   }

   // Candidate constructors for autowiring?
   // 有参数情况时，创建bean。先利用参数个数，类型等，确定最精确匹配的构造方法。
   // 4.应用后置处理器SmartInstantiationAwareBeanPostProcessor，拿到bean的候选构造函数
   Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
   if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
         mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
      // 5.如果ctors不为空 || mbd的注入方式为AUTOWIRE_CONSTRUCTOR || mdb定义了构造函数的参数值 || args不为空，则执行构造函数自动注入
      if(logger.isInfoEnabled()){
         logger.info("调用有参构造函数来实例化对象，实例化对象的beanName为："+beanName);
      }
      return autowireConstructor(beanName, mbd, ctors, args);
   }

   // No special handling: simply use no-arg constructor.
   // 6.没有特殊处理，则使用默认的构造函数进行bean的实例化
   if(logger.isInfoEnabled()){
      logger.info("调用无参构造函数来实例化对象，实例化对象的beanName为："+beanName);
   }
   return instantiateBean(beanName, mbd);
}
```

- 如果mbd.getFactoryMethodName()不为null，则使用工厂方法或者静态工厂方法来实例化Bean；
- 调用AutowiredAnnotationBeanPostProcessor的determineCandidateConstructors方法来选出构造方法

```java
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
      throws BeansException {

   if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
            // 2.调用SmartInstantiationAwareBeanPostProcessor的determineCandidateConstructors方法，
            // 该方法可以返回要用于beanClass的候选构造函数
            // 例如：使用@Autowire注解修饰构造函数，
            // 则该构造函数在这边会被AutowiredAnnotationBeanPostProcessor找到
            SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
            if(logger.isInfoEnabled()){
               logger.info("调用SmartInstantiationAwareBeanPostProcessor的实例："+ibp.getClass().getSimpleName()+"的determineCandidateConstructors选出构造方法\n" +
                     "说明：AutowiredAnnotationBeanPostProcessor用于解析构造参数上面加了@Autowire的注解" +
                     "一个构造函数上使用了@Autowired(required = true) 或 @Autowired， 就不允许有其他的构造函数使用 @Autowire；" +
                     "但是允许有多个构造函数同时使用 @Autowired(required = false)");
            }
            Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
            if (ctors != null) {
               // 3.如果ctors不为空，则不再继续执行其他的SmartInstantiationAwareBeanPostProcessor
               return ctors;
            }

         }
      }
   }
   return null;
}
```

选出构造方法

```java
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
			throws BeanCreationException {

		// Let's check for lookup methods here...
		// @Lookup注解检查
		if (!this.lookupMethodsChecked.contains(beanName)) {
			try {
				ReflectionUtils.doWithMethods(beanClass, method -> {
					Lookup lookup = method.getAnnotation(Lookup.class);
					if (lookup != null) {
						Assert.state(this.beanFactory != null, "No BeanFactory available");
						LookupOverride override = new LookupOverride(method, lookup.value());
						try {
							RootBeanDefinition mbd = (RootBeanDefinition)
									this.beanFactory.getMergedBeanDefinition(beanName);
							mbd.getMethodOverrides().addOverride(override);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(beanName,
									"Cannot apply @Lookup to beans without corresponding bean definition");
						}
					}
				});
			}
			catch (IllegalStateException ex) {
				throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
			}
			// 已经检查过的添加到lookupMethodsChecked
			this.lookupMethodsChecked.add(beanName);
		}

		// Quick check on the concurrent map first, with minimal locking.
		// 构造函数解析，首先检查是否存在于缓存中
		Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
		if (candidateConstructors == null) {
			// Fully synchronized resolution now...
			// 加锁进行操作
			synchronized (this.candidateConstructorsCache) {
				// 再次检查缓存，双重检测
				candidateConstructors = this.candidateConstructorsCache.get(beanClass);
				if (candidateConstructors == null) {
					// 存放原始的构造函数（候选者）
					Constructor<?>[] rawCandidates;
					try {
						// 获取beanClass声明的构造函数（如果没有声明，会返回一个默认的无参构造函数）
						rawCandidates = beanClass.getDeclaredConstructors();
					}
					catch (Throwable ex) {
						throw new BeanCreationException(beanName,
								"Resolution of declared constructors on bean Class [" + beanClass.getName() +
								"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
					}
					// 存放使用了@Autowire注解的构造函数
					List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
					// 存放使用了@Autowire注解，并且require=true的构造函数
					Constructor<?> requiredConstructor = null;
					// 存放默认的构造函数
					Constructor<?> defaultConstructor = null;
					Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
					int nonSyntheticConstructors = 0;
					// 遍历原始的构造函数候选者
					for (Constructor<?> candidate : rawCandidates) {
						if (!candidate.isSynthetic()) {
							nonSyntheticConstructors++;
						}
						else if (primaryConstructor != null) {
							continue;
						}
						// 获取候选者的注解属性
						//查找是否有@Autowired注解
						AnnotationAttributes ann = findAutowiredAnnotation(candidate);
						if (ann == null) {
							// 如果没有从候选者找到注解，则尝试解析beanClass的原始类（针对CGLIB代理）
							//如果没有@Autowired注解，查找父类的构造方法有没有@Autowired注解
							Class<?> userClass = ClassUtils.getUserClass(beanClass);
							if (userClass != beanClass) {
								try {
									Constructor<?> superCtor =
											userClass.getDeclaredConstructor(candidate.getParameterTypes());
									ann = findAutowiredAnnotation(superCtor);
								}
								catch (NoSuchMethodException ex) {
									// Simply proceed, no equivalent superclass constructor found...
								}
							}
						}
						// 如果该候选者使用了@Autowire注解
						if (ann != null) {
							if (requiredConstructor != null) {
								// 之前已经存在使用@Autowired(required = true)的构造函数，则不能存在其他使用@Autowire注解的构造函数，否则抛异常
								// Autowired注解的构造方法不止一个，那上一次处理的候选构造方法
								// 已经设置到requiredConstructor 中，那么第二个@Autowired注解的
								// 候选构造方法处理的时候就会抛异常
								throw new BeanCreationException(beanName,
										"Invalid autowire-marked constructor: " + candidate +
										". Found constructor with 'required' Autowired annotation already: " +
										requiredConstructor);
							}
							// 获取注解的require属性值
							boolean required = determineRequiredStatus(ann);
							// 说的简单点：在一个 bean 中，只要有构造函数使用了，“@Autowired(required = true)” 或 “@Autowired”，
							// 就不允许有其他的构造函数使用 “@Autowire”；
							// 但是允许有多个构造函数同时使用 “@Autowired(required = false)”
							if (required) {
								if (!candidates.isEmpty()) {
									// 如果当前候选者是@Autowired(required = true)，则之前不能存在其他使用@Autowire(required = false)注解的构造函数，否则抛异常
									throw new BeanCreationException(beanName,
											"Invalid autowire-marked constructors: " + candidates +
											". Found constructor with 'required' Autowired annotation: " +
											candidate);
								}
								// 如果该候选者使用的注解的required属性为true，赋值给requiredConstructor
								// 第一个处理的有@autowired处理的构造方法设置requiredConstructor ，并设置到candidates中
								requiredConstructor = candidate;
							}
							// 将使用了@Autowire注解的候选者添加到candidates
							candidates.add(candidate);
						}
						//当构造方法没有@Autowired注解且参数个数为0，选为defaultConstructor
						else if (candidate.getParameterCount() == 0) {
							// 如果没有使用注解，并且没有参数，则为默认的构造函数
							defaultConstructor = candidate;
						}
					}
					// 如果存在使用了@Autowire(required = false)注解的构造函数
					if (!candidates.isEmpty()) {
						// Add default constructor to list of optional constructors, as fallback.
						// 但是没有使用了@Autowire注解并且required属性为true的构造函数
						if (requiredConstructor == null) {
							if (defaultConstructor != null) {
								// 如果存在默认的构造函数，则将默认的构造函数添加到candidates
								candidates.add(defaultConstructor);
							}
							else if (candidates.size() == 1 && logger.isWarnEnabled()) {
								logger.warn("Inconsistent constructor declaration on bean with name '" + beanName +
										"': single autowire-marked constructor flagged as optional - " +
										"this constructor is effectively required since there is no " +
										"default constructor to fall back to: " + candidates.get(0));
							}
						}
						// 将所有的candidates当作候选者
						//候选方法不为空的时候进入此处，此时就一个@Autowired注解的构造方法
						candidateConstructors = candidates.toArray(new Constructor<?>[0]);
					}
					else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
						// 如果candidates为空 && beanClass只有一个声明的构造函数（非默认构造函数），则将该声明的构造函数作为候选者
						candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
					}
					else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
							defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
					}
					else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor};
					}
					else {
						// 否则返回一个空的Constructor对象
						candidateConstructors = new Constructor<?>[0];
					}
					// 将beanClass的构造函数解析结果放到缓存
					// 缓存选定的候选构造方法，供原型模式和scope模式第二次实例化时使用
					this.candidateConstructorsCache.put(beanClass, candidateConstructors);
				}
			}
		}
		// 返回解析的构造函数
		return (candidateConstructors.length > 0 ? candidateConstructors : null);
	}
```

选出构造方法的逻辑：

1.在一个 bean 中，只要有构造函数使用了，@Autowired(required = true) 或 @Autowired，就不允许有其他的构造函数使用 @Autowire或者@Autowired(required = true)；
2.但是允许有多个构造函数同时使用 @Autowired(required = false)，这时返回多个使用@Autowired(required = false)标注的构造函数。

### 1.无参构造方式

如果没有选出构造方法，则使用默认的无参构造创建Bean

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
   try {
      Object beanInstance;
      final BeanFactory parent = this;
      if (System.getSecurityManager() != null) {
         beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
               getInstantiationStrategy().instantiate(mbd, beanName, parent),
               getAccessControlContext());
      }
      else {
         // 调用bean初始化策InstantiationStrategy略初始化bean
         beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
      }
      // 包装一下，返回
      BeanWrapper bw = new BeanWrapperImpl(beanInstance);
      initBeanWrapper(bw);
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
   }
}
```

调用SimpleInstantiationStrategy的instantiate方法进行实例化Bean

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
   // Don't override the class with CGLIB if no overrides.
   // 如果不存在方法覆写，那就使用 java 反射进行实例化，否则使用 CGLIB
   // 方法覆写 请参见附录"方法注入"中对 lookup-method 和 replaced-method 的介绍
   if (!bd.hasMethodOverrides()) {
      Constructor<?> constructorToUse;
      synchronized (bd.constructorArgumentLock) {
         // 获取构造方法或factory-method
         constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
         if (constructorToUse == null) {
            // BeanDefinition中如果没有Constructor或者factory-method，则直接使用默认无参构造方法。
            final Class<?> clazz = bd.getBeanClass();
            if (clazz.isInterface()) {
               throw new BeanInstantiationException(clazz, "Specified class is an interface");
            }
            try {
               if (System.getSecurityManager() != null) {
                  constructorToUse = AccessController.doPrivileged(
                        (PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
               }
               else {
                  constructorToUse = clazz.getDeclaredConstructor();
               }
               bd.resolvedConstructorOrFactoryMethod = constructorToUse;
            }
            catch (Throwable ex) {
               throw new BeanInstantiationException(clazz, "No default constructor found", ex);
            }
         }
      }
      // 最终在BeanUtils中完成bean的初始化操作
      return BeanUtils.instantiateClass(constructorToUse);
   }
   else {
      // Must generate CGLIB subclass.
      // 存在方法覆写，利用 CGLIB 来完成实例化，需要依赖于 CGLIB 生成子类，这里就不展开了。
      return instantiateWithMethodInjection(bd, beanName, owner);
   }
}
```

```java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
   Assert.notNull(ctor, "Constructor must not be null");
   try {
      ReflectionUtils.makeAccessible(ctor);
      // 通过反射机制初始化bean
      return (KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
            KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
   }
   。。。。。。
}
```

小结：没有选出来构造方法，使用InstantiationStrategy策略（默认的无参构造）来反射创建Bean实例

### 2.有参构造方式

```java
protected BeanWrapper autowireConstructor(
      String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {

   return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```

下面获取构造方法的入参以及需要使用构造方法（适用于多个使用@Autowired(required = false)标注的场景），并反射创建Bean实例

```java
public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
      @Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

   BeanWrapperImpl bw = new BeanWrapperImpl();
   this.beanFactory.initBeanWrapper(bw);

   // spring决定采用哪个构造方法来实例化bean
   // 代码执行到这里说明spring已经决定要采用一个特殊构造方法来实例化bean
   // 最终用于实例化的构造函数
   Constructor<?> constructorToUse = null;
   // 调用反射来实例化对象时，需要具体的参数ArgumentsHolder对需要的具体的参数的包装
   ArgumentsHolder argsHolderToUse = null;
   // 最终用于实例化的构造函数参数
   Object[] argsToUse = null;
   // 解析出要用于实例化的构造函数参数
   if (explicitArgs != null) {
      // 如果explicitArgs不为空，则构造函数的参数直接使用explicitArgs
      // 通过getBean方法调用时，显式指定了参数，则explicitArgs就不为null
      argsToUse = explicitArgs;
   }
   else {
      // 尝试从缓存中获取已经解析过的构造函数参数
      Object[] argsToResolve = null;
      synchronized (mbd.constructorArgumentLock) {
         // 拿到缓存中已解析的构造函数或工厂方法
         constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
         // 如果constructorToUse不为空 && mbd标记了构造函数参数已解析
         if (constructorToUse != null && mbd.constructorArgumentsResolved) {
            // Found a cached constructor...
            // 从缓存中获取已解析的构造函数参数
            argsToUse = mbd.resolvedConstructorArguments;
            if (argsToUse == null) {
               // 1.2.4 如果resolvedConstructorArguments为空，则从缓存中获取准备用于解析的构造函数参数，
               // constructorArgumentsResolved为true时，resolvedConstructorArguments和
               // preparedConstructorArguments必然有一个缓存了构造函数的参数
               argsToResolve = mbd.preparedConstructorArguments;
            }
         }
      }
      if (argsToResolve != null) {
         // 如果argsToResolve不为空，则对构造函数参数进行解析，
         // 如给定方法的构造函数 A(int,int)则通过此方法后就会把配置中的("1","1")转换为(1,1)
         argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
      }
   }


   // 如果构造函数没有被缓存，则通过配置文件获取
   if (constructorToUse == null) {
      // Need to resolve the constructor.
      // 检查是否需要自动装配：chosenCtors不为空 || autowireMode为AUTOWIRE_CONSTRUCTOR
      // 例子：当chosenCtors不为空时，代表有构造函数通过@Autowire修饰，因此需要自动装配
      boolean autowiring = (chosenCtors != null ||
            mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
      ConstructorArgumentValues resolvedValues = null;

      // 最小参数个数
      int minNrOfArgs;
      if (explicitArgs != null) {
         // explicitArgs不为空，则使用explicitArgs的length作为minNrOfArgs的值
         minNrOfArgs = explicitArgs.length;
      }
      else {
         // 获得mbd的构造函数的参数值（indexedArgumentValues：带index的参数值；genericArgumentValues：通用的参数值）
         ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
         // 创建ConstructorArgumentValues对象resolvedValues，用于承载解析后的构造函数参数的值
         resolvedValues = new ConstructorArgumentValues();
         // 解析mbd的构造函数的参数，并返回参数个数
         minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
         // 注：这边解析mbd中的构造函数参数值，主要是处理我们通过xml方式定义的构造函数注入的参数，
         // 但是如果我们是通过@Autowire注解直接修饰构造函数，则mbd是没有这些参数值的
      }

      // 确认构造函数的候选者
      // Take specified constructors, if any.
      // 如果入参chosenCtors不为空，则将chosenCtors的构造函数作为候选者
      Constructor<?>[] candidates = chosenCtors;
      if (candidates == null) {
         Class<?> beanClass = mbd.getBeanClass();
         try {
            // 如果入参chosenCtors为空，则获取beanClass的构造函数
            // （mbd是否允许访问非公共构造函数和方法 ? 所有声明的构造函数：公共构造函数）
            candidates = (mbd.isNonPublicAccessAllowed() ?
                  beanClass.getDeclaredConstructors() : beanClass.getConstructors());
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Resolution of declared constructors on bean Class [" + beanClass.getName() +
                  "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
         }
      }
      // 对给定的构造函数排序：先按方法修饰符排序：public排非public前面，再按构造函数参数个数排序：参数多的排前面
      AutowireUtils.sortConstructors(candidates);
      // 最小匹配权重，权重越小，越接近我们要找的目标构造函数
      int minTypeDiffWeight = Integer.MAX_VALUE;
      Set<Constructor<?>> ambiguousConstructors = null;
      LinkedList<UnsatisfiedDependencyException> causes = null;

      // 遍历所有构造函数候选者，找出符合条件的构造函数
      for (Constructor<?> candidate : candidates) {
         // 拿到当前遍历的构造函数的参数类型数组
         Class<?>[] paramTypes = candidate.getParameterTypes();

         if (constructorToUse != null && argsToUse.length > paramTypes.length) {
            // Already found greedy constructor that can be satisfied ->
            // do not look any further, there are only less greedy constructors left.
            // 如果已经找到满足的构造函数 && 目标构造函数需要的参数个数大于当前遍历的构造函数的参数个数则终止，
            // 因为遍历的构造函数已经排过序，后面不会有更合适的候选者了
            break;
         }
         if (paramTypes.length < minNrOfArgs) {
            // 如果当前遍历到的构造函数的参数个数小于我们所需的参数个数，则直接跳过该构造函数
            continue;
         }

         ArgumentsHolder argsHolder;
         if (resolvedValues != null) {
            // 存在参数则根据参数值来匹配参数类型
            try {
               // resolvedValues不为空，
               // 获取当前遍历的构造函数的参数名称
               // 解析使用ConstructorProperties注解的构造函数参数
               String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
               if (paramNames == null) {
                  // 获取参数名称解析
                  ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                  if (pnd != null) {
                     // 使用参数名称解析器获取当前遍历的构造函数的参数名称
                     paramNames = pnd.getParameterNames(candidate);
                  }
               }
               // 创建一个参数数组以调用构造函数或工厂方法，
               // 主要是通过参数类型和参数名解析构造函数或工厂方法所需的参数
               // （如果参数是其他bean，则会解析依赖的bean）
               String str=Arrays.toString(paramNames);
               if(logger.isInfoEnabled()){
                  logger.info("根据要注入参数的类型和要注入参数的名称，从bdMap中选出要注入的类，其中参数名称为："+str+"\n" +
                        "说明：如果要注入的类没有创建，则先创建，可能出现循环依赖问题。");
               }
               argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
                     getUserDeclaredConstructor(candidate), autowiring);
            }
            catch (UnsatisfiedDependencyException ex) {
               // 参数匹配失败，则抛出异常
               if (logger.isTraceEnabled()) {
                  logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
               }
               // Swallow and try next constructor.
               if (causes == null) {
                  causes = new LinkedList<>();
               }
               causes.add(ex);
               continue;
            }
         }
         else {
            // resolvedValues为空，则explicitArgs不为空，即给出了显式参数
            // Explicit arguments given -> arguments length must match exactly.
            // 如果当前遍历的构造函数参数个数与explicitArgs长度不相同，则跳过该构造函数
            if (paramTypes.length != explicitArgs.length) {
               continue;
            }
            // 使用显式给出的参数构造ArgumentsHolder
            argsHolder = new ArgumentsHolder(explicitArgs);
         }

         // 根据mbd的解析构造函数模式（true: 宽松模式(默认)，false：严格模式），
         // 将argsHolder的参数和paramTypes进行比较，计算paramTypes的类型差异权重值
         int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
               argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
         // Choose this constructor if it represents the closest match.
         // 类型差异权重值越小,则说明构造函数越匹配，则选择此构造函数
         if (typeDiffWeight < minTypeDiffWeight) {
            // 将要使用的参数都替换成差异权重值更小的
            constructorToUse = candidate;
            argsHolderToUse = argsHolder;
            argsToUse = argsHolder.arguments;
            minTypeDiffWeight = typeDiffWeight;
            // 如果出现权重值更小的候选者，则将ambiguousConstructors清空，允许之前存在权重值相同的候选者
            ambiguousConstructors = null;
         }
         // 如果存在两个候选者的权重值相同，并且是当前遍历过权重值最小的
         else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
            // 将这两个候选者都添加到ambiguousConstructors
            if (ambiguousConstructors == null) {
               ambiguousConstructors = new LinkedHashSet<>();
               ambiguousConstructors.add(constructorToUse);
            }
            ambiguousConstructors.add(candidate);
         }
      }


      if (constructorToUse == null) {
         // 如果最终没有找到匹配的构造函数，则进行异常处理
         if (causes != null) {
            UnsatisfiedDependencyException ex = causes.removeLast();
            for (Exception cause : causes) {
               this.beanFactory.onSuppressedException(cause);
            }
            throw ex;
         }
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Could not resolve matching constructor " +
               "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
      }
      else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
         // 如果找到了匹配的构造函数，但是存在多个（ambiguousConstructors不为空） && 解析构造函数的模式为严格模式（默认非严格模式），则抛出异常
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Ambiguous constructor matches found in bean '" + beanName + "' " +
               "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
               ambiguousConstructors);
      }

      if (explicitArgs == null) {
         // 将解析的构造函数和参数放到缓存
         argsHolderToUse.storeCache(mbd, constructorToUse);
      }
   }

   try {
      final InstantiationStrategy strategy = beanFactory.getInstantiationStrategy();
      Object beanInstance;

      // 根据实例化策略以及得到的构造函数及构造函数参数实例化bean
      if (System.getSecurityManager() != null) {
         final Constructor<?> ctorToUse = constructorToUse;
         final Object[] argumentsToUse = argsToUse;
         beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
               strategy.instantiate(mbd, beanName, beanFactory, ctorToUse, argumentsToUse),
               beanFactory.getAccessControlContext());
      }
      else {
         if(logger.isInfoEnabled()){
            logger.info("从多个构造方法中，最终选出来的构造方法为："+constructorToUse+"，参数为："+Arrays.toString(argsToUse));
         }
         beanInstance = strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
      }

      // 将构造的实例加入BeanWrapper中，并返回
      bw.setBeanInstance(beanInstance);
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean instantiation via constructor failed", ex);
   }
}
```

步骤：

1. 对给定的构造函数排序：先按方法修饰符排序：public排非public前面，再按构造函数参数个数排序：参数多的排前面
2. 获取要注入的参数（其中XML配置和注解配置的区别就在这里）
3. 根据构造方法的参数类型差异权重值，获取最终要使用的构造方法（适用于多个构造函数同时使用 @Autowired(required = false)的场景）
4. 使用InstantiationStrategy策略来实例化Bean

分析获取要注入的参数的逻辑

```java
private ArgumentsHolder createArgumentArray(
      String beanName, RootBeanDefinition mbd, @Nullable ConstructorArgumentValues resolvedValues,
      BeanWrapper bw, Class<?>[] paramTypes, @Nullable String[] paramNames, Executable executable,
      boolean autowiring) throws UnsatisfiedDependencyException {

   TypeConverter customConverter = this.beanFactory.getCustomTypeConverter();
   // 获取类型转换器
   TypeConverter converter = (customConverter != null ? customConverter : bw);

   // 新建一个ArgumentsHolder来存放匹配到的参数
   ArgumentsHolder args = new ArgumentsHolder(paramTypes.length);
   Set<ConstructorArgumentValues.ValueHolder> usedValueHolders = new HashSet<>(paramTypes.length);
   Set<String> autowiredBeanNames = new LinkedHashSet<>(4);

   // 遍历参数类型数组
   for (int paramIndex = 0; paramIndex < paramTypes.length; paramIndex++) {
      // 拿到当前遍历的参数类型
      Class<?> paramType = paramTypes[paramIndex];
      // 拿到当前遍历的参数名
      String paramName = (paramNames != null ? paramNames[paramIndex] : "");
      // Try to find matching constructor argument value, either indexed or generic.
      // 查找当前遍历的参数，是否在mdb对应的bean的构造函数参数中存在index、类型和名称匹配的
      ConstructorArgumentValues.ValueHolder valueHolder = null;
      if (resolvedValues != null) {
         valueHolder = resolvedValues.getArgumentValue(paramIndex, paramType, paramName, usedValueHolders);
         // If we couldn't find a direct match and are not supposed to autowire,
         // let's try the next generic, untyped argument value as fallback:
         // it could match after type conversion (for example, String -> int).
         // 如果我们找不到直接匹配并且不应该自动装配，那么让我们尝试下一个通用的无类型参数值作为降级方法：它可以在类型转换后匹配（例如，String - > int）。
         if (valueHolder == null && (!autowiring || paramTypes.length == resolvedValues.getArgumentCount())) {
            valueHolder = resolvedValues.getGenericArgumentValue(null, null, usedValueHolders);
         }
      }
      // XML的有参构造走的是这里
      if (valueHolder != null) {
         // valueHolder不为空，存在匹配的参数
         // We found a potential match - let's give it a try.
         // Do not consider the same value definition multiple times!
         // 将valueHolder添加到usedValueHolders
         usedValueHolders.add(valueHolder);
         // 原始属性值
         Object originalValue = valueHolder.getValue();
         // 转换后的属性值
         Object convertedValue;
         if (valueHolder.isConverted()) {
            // 如果valueHolder已经转换过
            // 则直接获取转换后的值
            convertedValue = valueHolder.getConvertedValue();
            // 将convertedValue作为args在paramIndex位置的预备参数
            args.preparedArguments[paramIndex] = convertedValue;
         }
         else {
            MethodParameter methodParam = MethodParameter.forExecutable(executable, paramIndex);
            try {
               convertedValue = converter.convertIfNecessary(originalValue, paramType, methodParam);
            }
            catch (TypeMismatchException ex) {
               throw new UnsatisfiedDependencyException(
                     mbd.getResourceDescription(), beanName, new InjectionPoint(methodParam),
                     "Could not convert argument value of type [" +
                           ObjectUtils.nullSafeClassName(valueHolder.getValue()) +
                           "] to required type [" + paramType.getName() + "]: " + ex.getMessage());
            }
            Object sourceHolder = valueHolder.getSource();
            if (sourceHolder instanceof ConstructorArgumentValues.ValueHolder) {
               Object sourceValue = ((ConstructorArgumentValues.ValueHolder) sourceHolder).getValue();
               args.resolveNecessary = true;
               args.preparedArguments[paramIndex] = sourceValue;
            }
         }
         // 将convertedValue作为args在paramIndex位置的参数
         args.arguments[paramIndex] = convertedValue;
         // 将originalValue作为args在paramIndex位置的原始参数
         args.rawArguments[paramIndex] = originalValue;
      }
      else {
         // 注解的有参构造走的是这里
         // valueHolder为空，不存在匹配的参数
         // 将方法（此处为构造函数）和参数索引封装成MethodParameter
         MethodParameter methodParam = MethodParameter.forExecutable(executable, paramIndex);
         // No explicit match found: we're either supposed to autowire or
         // have to fail creating an argument array for the given constructor.

         // 找不到明确的匹配，并且不是自动装配，则抛出异常
         if (!autowiring) {
            throw new UnsatisfiedDependencyException(
                  mbd.getResourceDescription(), beanName, new InjectionPoint(methodParam),
                  "Ambiguous argument values for parameter of type [" + paramType.getName() +
                  "] - did you specify the correct bean references as arguments?");
         }
         try {
            if(logger.isInfoEnabled()){
               logger.info("要注入的参数类型为："+methodParam.getParameterType().getSimpleName());
            }
            // 如果是自动装配，则调用用于解析自动装配参数的方法，返回的结果为依赖的bean实例对象
            // 例如：@Autowire修饰构造函数，自动注入构造函数中的参数bean就是在这边处理
            Object autowiredArgument =
                  resolveAutowiredArgument(methodParam, beanName, autowiredBeanNames, converter);
            // 将通过自动装配解析出来的参数赋值给args
            args.rawArguments[paramIndex] = autowiredArgument;
            args.arguments[paramIndex] = autowiredArgument;
            args.preparedArguments[paramIndex] = new AutowiredArgumentMarker();
            args.resolveNecessary = true;
         }
         catch (BeansException ex) {
            // 如果自动装配解析失败，则会抛出异常
            throw new UnsatisfiedDependencyException(
                  mbd.getResourceDescription(), beanName, new InjectionPoint(methodParam), ex);
         }
      }
   }

   // 如果依赖了其他的bean，则注册依赖关系
   for (String autowiredBeanName : autowiredBeanNames) {
      this.beanFactory.registerDependentBean(autowiredBeanName, beanName);
      if (logger.isDebugEnabled()) {
         logger.debug("在构造函数中自动注入 by type from bean name '" + beanName +
               "' via " + (executable instanceof Constructor ? "constructor" : "factory method") +
               " to bean named '" + autowiredBeanName + "'");
      }
   }

   return args;
}
```

其中XML的有参构造valueHolder不为null，注解的有参构造的valueHolder为null。

- XML的有参构造使用转换器进行转换（和SpringMVC的Http请求解析一样）
- 注解的有参构造调用DefaultListableBeanFactory#resolveDependency，先根据类型（如果有泛型，则进行泛型类型匹配）从Spring容器中获取匹配的Bean，后根据@Qualifier>@Primary>@Priority>属性名称匹配的顺序选出要注入的Bean。

使用注解的有参构造获取要注入的属性

```java
protected Object resolveAutowiredArgument(MethodParameter param, String beanName,
      @Nullable Set<String> autowiredBeanNames, TypeConverter typeConverter) {

   // 如果参数类型为InjectionPoint
   if (InjectionPoint.class.isAssignableFrom(param.getParameterType())) {
      // 拿到当前的InjectionPoint（存储了当前正在解析依赖的方法参数信息，DependencyDescriptor）
      InjectionPoint injectionPoint = currentInjectionPoint.get();
      if (injectionPoint == null) {
         // 当前injectionPoint为空，则抛出异常：目前没有可用的InjectionPoint
         throw new IllegalStateException("No current InjectionPoint available for " + param);
      }
      return injectionPoint;
   }
   // 解析指定依赖，DependencyDescriptor：将MethodParameter的方法参数索引信息封装成DependencyDescriptor
   return this.beanFactory.resolveDependency(
         new DependencyDescriptor(param, true), beanName, autowiredBeanNames, typeConverter);
}
```

最终使用调用DefaultListableBeanFactory#resolveDependency方法，先根据类型（如果有泛型，则进行泛型类型匹配）从Spring容器中获取匹配的Bean，后根据@Qualifier>@Primary>@Priority>属性名称匹配的顺序选出要注入的Bean。

其中resolveDependency的源码分析参见todo

继续分析使用InstantiationStrategy策略来实例化Bean，调用SimpleInstantiationStrategy的instantiate方法进行实例化Bean

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
      final Constructor<?> ctor, @Nullable Object... args) {

   if (!bd.hasMethodOverrides()) {
      if (System.getSecurityManager() != null) {
         // use own privileged to change accessibility (when security is on)
         AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            ReflectionUtils.makeAccessible(ctor);
            return null;
         });
      }
      return (args != null ? BeanUtils.instantiateClass(ctor, args) : BeanUtils.instantiateClass(ctor));
   }
   else {
      return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
   }
}
```

```java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
   Assert.notNull(ctor, "Constructor must not be null");
   try {
      ReflectionUtils.makeAccessible(ctor);
      // 通过反射机制初始化bean
      return (KotlinDetector.isKotlinType(ctor.getDeclaringClass()) ?
            KotlinDelegate.instantiateClass(ctor, args) : ctor.newInstance(args));
   }
}
```

小结：

1. 对给定的构造函数排序：先按方法修饰符排序：public排非public前面，再按构造函数参数个数排序：参数多的排前面
2. 获取要注入的参数（其中XML配置和注解配置的区别就在这里）
3. 根据构造方法的参数类型差异权重值，获取最终要使用的构造方法（适用于多个构造函数同时使用 @Autowired(required = false)的场景）
4. 使用InstantiationStrategy策略来实例化Bean

### 3.静态工厂方式

在AbstractAutowireCapableBeanFactory的createBeanInstance方法中来创建实例Bean，分析静态工厂方法以及工厂方法

```java
// @Bean标注的bean方法,或者使用FactoryBean的factory-method来创建，支持静态工厂和实例工厂，
if (mbd.getFactoryMethodName() != null) {
   if(logger.isInfoEnabled()){
      logger.info("该bean是@Bean标注的方法，其中beanName为："+beanName);
   }
   return instantiateUsingFactoryMethod(beanName, mbd, args);
}
```

继续分析instantiateUsingFactoryMethod方法来

```java
protected BeanWrapper instantiateUsingFactoryMethod(
      String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

   return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
}
```



```java
public BeanWrapper instantiateUsingFactoryMethod(
      String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

   BeanWrapperImpl bw = new BeanWrapperImpl();
   this.beanFactory.initBeanWrapper(bw);

   Object factoryBean;
   Class<?> factoryClass;
   boolean isStatic;
   // 通过beanDefinition获取到factoryBeanName ，实际就是@Bean注解的方法所在的configuration类
   String factoryBeanName = mbd.getFactoryBeanName();
   // 工厂方法走这里
   if (factoryBeanName != null) {
      if (factoryBeanName.equals(beanName)) {
         throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
               "factory-bean reference points back to the same bean definition");
      }
      // 获取配置类的实例
      if(logger.isInfoEnabled()){
         logger.info("获取@Bean标注的方法所在的配置类："+factoryBeanName+"未在单例池中的话则先将其加入到单例池中");
      }
      factoryBean = this.beanFactory.getBean(factoryBeanName);
      if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
         throw new ImplicitlyAppearedSingletonException();
      }
      factoryClass = factoryBean.getClass();
      isStatic = false;
   }
   else {
      // 静态工厂方法
      if (!mbd.hasBeanClass()) {
         throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
               "bean definition declares neither a bean class nor a factory-bean reference");
      }
      factoryBean = null;
      factoryClass = mbd.getBeanClass();
      isStatic = true;
   }

   Method factoryMethodToUse = null;
   ArgumentsHolder argsHolderToUse = null;
   Object[] argsToUse = null;
   //如果在调用getBean方法时有传参，那就用传的参作为@Bean注解的方法（工厂方法）的参数，
   // 一般懒加载的bean才会传参，启动过程就实例化的实际上都没有传参
   if (explicitArgs != null) {
      argsToUse = explicitArgs;
   }
   else {
      Object[] argsToResolve = null;
      synchronized (mbd.constructorArgumentLock) {
         factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
         // 不为空表示已经使用过工厂方法，现在是再次使用工厂方法，
         //  一般原型模式和Scope模式采用的上，直接使用该工厂方法和缓存的参数
         if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
            // Found a cached factory method...
            argsToUse = mbd.resolvedConstructorArguments;
            if (argsToUse == null) {
               argsToResolve = mbd.preparedConstructorArguments;
            }
         }
      }
      if (argsToResolve != null) {
         argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve);
      }
   }

   //  调用getBean方法没有传参，或者也是第一次使用工厂方法
   if (factoryMethodToUse == null || argsToUse == null) {
      // Need to determine the factory method...
      // Try all methods with this name to see if they match the given arguments.
      factoryClass = ClassUtils.getUserClass(factoryClass);

      // 获取配置类的所有候选方法
      Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
      List<Method> candidateList = new ArrayList<>();
      for (Method candidate : rawCandidates) {
         // 查找到与工厂方法同名的候选方法,没有@Bean的同名方法不被考虑
         if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
            candidateList.add(candidate);
         }
      }
      // 有多个与工厂方法同名的候选方法时，进行排序。public的方法会往前排，然后参数个数多的方法往前排
      //具体排序代码--->
      // org.springframework.beans.factory.support.AutowireUtils#sortFactoryMethods

      Method[] candidates = candidateList.toArray(new Method[0]);
      AutowireUtils.sortFactoryMethods(candidates);

      ConstructorArgumentValues resolvedValues = null;
      boolean autowiring = (mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
      int minTypeDiffWeight = Integer.MAX_VALUE;
      Set<Method> ambiguousFactoryMethods = null;

      int minNrOfArgs;
      // 如果调用getBean方法时有传参，那么工厂方法最少参数个数要等于传参个数
      if (explicitArgs != null) {
         minNrOfArgs = explicitArgs.length;
      }
      else {
         // We don't have arguments passed in programmatically, so we need to resolve the
         // arguments specified in the constructor arguments held in the bean definition.
         if (mbd.hasConstructorArgumentValues()) {
            ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            resolvedValues = new ConstructorArgumentValues();
            minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
         }
         else {
            minNrOfArgs = 0;
         }
      }

      LinkedList<UnsatisfiedDependencyException> causes = null;

      // 遍历同名候选方法
      for (Method candidate : candidates) {
         //   获取候选方法的参数列表
         Class<?>[] paramTypes = candidate.getParameterTypes();

         if (paramTypes.length >= minNrOfArgs) {
            ArgumentsHolder argsHolder;

            //在调用getBean方法时传的参数不为空，则工厂方法的参数个数需要与传入的参数个数严格一致
            if (explicitArgs != null) {
               // Explicit arguments given -> arguments length must match exactly.
               if (paramTypes.length != explicitArgs.length) {
                  continue;
               }
               argsHolder = new ArgumentsHolder(explicitArgs);
            }
            else {
               // Resolved constructor arguments: type conversion and/or autowiring necessary.
               try {
                  String[] paramNames = null;
                  ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                  if (pnd != null) {
                     paramNames = pnd.getParameterNames(candidate);
                  }
                  //当传入的参数为空，需要根据工厂方法的参数类型注入相应的bean。
                  String str=Arrays.toString(paramNames);
                  if(logger.isInfoEnabled()){
                     logger.info("根据要注入参数的类型和要注入参数的名称，从bdMap中选出要注入的类，其中参数名称为："+str+"\n" +
                           "说明：如果要注入的类没有创建，则先创建，可能出现循环依赖问题。");
                  }
                  argsHolder = createArgumentArray(
                        beanName, mbd, resolvedValues, bw, paramTypes, paramNames, candidate, autowiring);
               }
               catch (UnsatisfiedDependencyException ex) {
                  if (logger.isTraceEnabled()) {
                     logger.trace("Ignoring factory method [" + candidate + "] of bean '" + beanName + "': " + ex);
                  }
                  // Swallow and try next overloaded factory method.
                  if (causes == null) {
                     causes = new LinkedList<>();
                  }
                  causes.add(ex);
                  continue;
               }
            }

            /**计算工厂方法的权重，分严格模式和宽松模式，计算方式可以看本文最后的附录
             严格模式会校验子类（注入参数）继承了多少层父类（方法参数）层数越多权重越大，越不匹配
             ，宽松模式，只要是注入参数类型是方法参数类型的子类就行。
             默认宽松模式 在argsHolders中会有arguments和rawArguments，；
             例如在注入bean时，如果有经历过createArgumentArray方法中的TypeConverter
             （如有有定义并且注册到beanFactory中）的，arguments和rawArguments的值是不一样的
             如果没有经过转换，两者是一样的。通过getBean传入的参数两者通常都是一样的
             所以都是先将工厂方法的参数类型与arguments的比较，不同则赋予最大权重值，
             相同则与rawArguments比较，与rawArguments中的相同，就会赋最大权重值-1024，
             不相同，则赋最大权重值-512，经过类型转换一定会执行最大权重值-512的操作。
             权重值越大，该工厂方法越不匹配。总的来说就是传入的参数或者注入的参数类型
             与工厂方法参数类型的比对。**/

            int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                  argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
            // Choose this factory method if it represents the closest match.
            if (typeDiffWeight < minTypeDiffWeight) {
               /**  当权重小时，重新设置factoryMethodToUse 和argsHolderToUse ，argsToUse ，
                并把当前权重值设置为最小权重值，等待遍历的下一个候选工厂方法比对，
                并且将ambiguousFactoryMethods （表示有含糊同样权重的候选方法）设置为空**/
               factoryMethodToUse = candidate;
               argsHolderToUse = argsHolder;
               argsToUse = argsHolder.arguments;
               minTypeDiffWeight = typeDiffWeight;
               ambiguousFactoryMethods = null;
            }
            // Find out about ambiguity: In case of the same type difference weight
            // for methods with the same number of parameters, collect such candidates
            // and eventually raise an ambiguity exception.
            // However, only perform that check in non-lenient constructor resolution mode,
            // and explicitly ignore overridden methods (with the same parameter signature).
            /**  当遍历到下一个候选方法的时候，已经设置了factoryMethodToUse 且权重值
             与上一次的最小权重值相等时，ambiguousFactoryMethods填值，这个ambiguousFactoryMethods不为空
             表示有两个候选方法的最小权重相等，spring无法匹配出最适合的工厂方法，
             如果再继续往下遍历候选器，有更小的权重值，那ambiguousFactoryMethods会
             再次被设置为空**/
            else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
                  !mbd.isLenientConstructorResolution() &&
                  paramTypes.length == factoryMethodToUse.getParameterCount() &&
                  !Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
               if (ambiguousFactoryMethods == null) {
                  ambiguousFactoryMethods = new LinkedHashSet<>();
                  ambiguousFactoryMethods.add(factoryMethodToUse);
               }
               ambiguousFactoryMethods.add(candidate);
            }
         }
      }

      if (factoryMethodToUse == null) {
         if (causes != null) {
            UnsatisfiedDependencyException ex = causes.removeLast();
            for (Exception cause : causes) {
               this.beanFactory.onSuppressedException(cause);
            }
            throw ex;
         }
         List<String> argTypes = new ArrayList<>(minNrOfArgs);
         if (explicitArgs != null) {
            for (Object arg : explicitArgs) {
               argTypes.add(arg != null ? arg.getClass().getSimpleName() : "null");
            }
         }
         else if (resolvedValues != null) {
            Set<ValueHolder> valueHolders = new LinkedHashSet<>(resolvedValues.getArgumentCount());
            valueHolders.addAll(resolvedValues.getIndexedArgumentValues().values());
            valueHolders.addAll(resolvedValues.getGenericArgumentValues());
            for (ValueHolder value : valueHolders) {
               String argType = (value.getType() != null ? ClassUtils.getShortName(value.getType()) :
                     (value.getValue() != null ? value.getValue().getClass().getSimpleName() : "null"));
               argTypes.add(argType);
            }
         }
         String argDesc = StringUtils.collectionToCommaDelimitedString(argTypes);
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "No matching factory method found: " +
               (mbd.getFactoryBeanName() != null ?
                  "factory bean '" + mbd.getFactoryBeanName() + "'; " : "") +
               "factory method '" + mbd.getFactoryMethodName() + "(" + argDesc + ")'. " +
               "Check that a method with the specified name " +
               (minNrOfArgs > 0 ? "and arguments " : "") +
               "exists and that it is " +
               (isStatic ? "static" : "non-static") + ".");
      }
      //返回类型不能为void
      else if (void.class == factoryMethodToUse.getReturnType()) {
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Invalid factory method '" + mbd.getFactoryMethodName() +
               "': needs to have a non-void return type!");
      }
      //存在含糊的两个工厂方法，不知选哪个
      else if (ambiguousFactoryMethods != null) {
         throw new BeanCreationException(mbd.getResourceDescription(), beanName,
               "Ambiguous factory method matches found in bean '" + beanName + "' " +
               "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
               ambiguousFactoryMethods);
      }

      if (explicitArgs == null && argsHolderToUse != null) {
         argsHolderToUse.storeCache(mbd, factoryMethodToUse);
      }
   }

   try {
      Object beanInstance;

      if (System.getSecurityManager() != null) {
         final Object fb = factoryBean;
         final Method factoryMethod = factoryMethodToUse;
         final Object[] args = argsToUse;
         beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
               beanFactory.getInstantiationStrategy().instantiate(mbd, beanName, beanFactory, fb, factoryMethod, args),
               beanFactory.getAccessControlContext());
      }
      else {
         if(logger.isInfoEnabled()){
            logger.info("选出来的beanMethod来反射得到该实例，beanMethod为："+factoryMethodToUse+"，参数为："+Arrays.toString(argsToUse));
         }
         beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
               mbd, beanName, this.beanFactory, factoryBean, factoryMethodToUse, argsToUse);
      }

      //   到达这里，恭喜，可以完成实例化了
      bw.setBeanInstance(beanInstance);
      return bw;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Bean instantiation via factory method failed", ex);
   }
}
```

1. 根据factoryBeanName是否为null判断是工厂方法还是静态工厂方法，其中工厂方法需要先实例化工厂类
2. 拿到工厂类中所有的与factoryMethodName相同的且有@Bean标注的方法
3. 调用createArgumentArray获取需要注入的参数
4. 根据@Bean方法参数类型差异权重值，获取最终要使用的@Bean方法（适用于多个构造函数同时使用 @Autowired(required = false)的场景）
5. 使用InstantiationStrategy策略来实例化Bean

分析使用InstantiationStrategy策略来实例化Bean

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
      @Nullable Object factoryBean, final Method factoryMethod, @Nullable Object... args) {

   try {
      if (System.getSecurityManager() != null) {
         AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            ReflectionUtils.makeAccessible(factoryMethod);
            return null;
         });
      }
      else {
         ReflectionUtils.makeAccessible(factoryMethod);
      }

      Method priorInvokedFactoryMethod = currentlyInvokedFactoryMethod.get();
      try {
         currentlyInvokedFactoryMethod.set(factoryMethod);
         // 反射调用
         Object result = factoryMethod.invoke(factoryBean, args);
         if (result == null) {
            result = new NullBean();
         }
         return result;
      }
      finally {
         if (priorInvokedFactoryMethod != null) {
            currentlyInvokedFactoryMethod.set(priorInvokedFactoryMethod);
         }
         else {
            currentlyInvokedFactoryMethod.remove();
         }
      }
   }
   。。。。。。
}
```

### 4.工厂方式

工厂方法实例化Bean与静态工厂方式流程几乎一致，就是根据factoryBeanName是否为nul，l判断是工厂方法还是静态工厂方法，其中工厂方法需要先实例化工厂类

### 5.FactoryBean方式

实例化时机：

通常在第一个需要注入属性的类的时候，原因：注入属性的时候需要调用DefaultListableBeanFactory#resolveDependency来获取要注入的类，在判断类型匹配的时候实例化实现FactoryBean接口的Bean，之后调用FactoryBean的getObjectType方法来进行类型比较。

实例化的时候也是调用AbstractAutowireCapableBeanFactory的createBeanInstance方法实例化Bean，在走一遍上面4个步骤中的一个。

```java
private FactoryBean<?> getSingletonFactoryBeanForTypeCheck(String beanName, RootBeanDefinition mbd) {
   synchronized (getSingletonMutex()) {
      BeanWrapper bw = this.factoryBeanInstanceCache.get(beanName);
      if (bw != null) {
         if(logger.isInfoEnabled()){
            logger.info("从factoryBeanInstanceCache得到该实例，beanName为："+beanName);
         }
         return (FactoryBean<?>) bw.getWrappedInstance();
      }
      Object beanInstance = getSingleton(beanName, false);
      if (beanInstance instanceof FactoryBean) {
         return (FactoryBean<?>) beanInstance;
      }
      if (isSingletonCurrentlyInCreation(beanName) ||
            (mbd.getFactoryBeanName() != null && isSingletonCurrentlyInCreation(mbd.getFactoryBeanName()))) {
         if(logger.isInfoEnabled()){
            logger.info("该类正在创建直接返回，类名为："+beanName);
         }
         return null;
      }

      Object instance;
      try {
         // Mark this bean as currently in creation, even if just partially.
         beforeSingletonCreation(beanName);
         // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
         instance = resolveBeforeInstantiation(beanName, mbd);
         if (instance == null) {
            // 根据工厂方法、静态工厂方法、有参构造、无参构造来实例化Bean
            bw = createBeanInstance(beanName, mbd, null);
            instance = bw.getWrappedInstance();
         }
      }
      finally {
         // Finished partial creation of this bean.
         afterSingletonCreation(beanName);
      }

      FactoryBean<?> fb = getFactoryBean(beanName, instance);
      if (bw != null) {
         if(logger.isInfoEnabled()){
            logger.info("向factoryBeanInstanceCache加入该实例，其beanName为："+beanName);
         }
         this.factoryBeanInstanceCache.put(beanName, bw);
      }
      return fb;
   }
}
```

1. 先尝试从缓存中获取
2. 调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法，达到创建Bean提前结束的效果（没用过）
3. 调用createBeanInstance进行实例化Bean
4. 把FactoryBean放到缓存中去，留着后续初始化的时候使用

小结：FactoryBean方式的创建Bean的方式就是根据工厂方法、静态工厂方法、有参构造、无参构造中的一种，不同的是FactoryBean的实例化方式较早，因为注入属性的时候要进行类型判断

## 总结

1. 从源码角度来说XML配置和注解配置实例化Bean的时候是没有区别的，区别的是初始化的时候（注入属性等等）
2. 无参构造最简单，不需要实例化入参，直接使用策略模式反射实例化Bean
3. 有参构造需要先选出入参，然后根据差异值来确定要使用的构造方法，再使用策略模式反射实例化Bean
4. 工厂方式以及静态工厂方式，拿到工厂类中所有的与factoryMethodName相同的且有@Bean标注的方法，获取需要注入的参数，然后根据参数类型差异获取最终要使用的@Bean方法，再使用策略模式反射实例化Bean
5. FactoryBean方式，只是实例化的时机较早（因为要进行类型比较），之后使用以上4中方式中的一种进行实例化
6. 总体来说SimpleInstantiationStrategy的instantiate一共有三种策略来实例化Bean：无参方式、有参方式、工厂方式。