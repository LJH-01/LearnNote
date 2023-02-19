[TOC]



# Spring的IOC容器初始化Bean源码详解

## 从IOC容器中获取Bean



详细分析AbstractBeanFactory的doGetBean逻辑

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

   // 如果这个 name 是 FactoryBean 的beanName (&+beanName),就删除& ,
   // 返回beanName ,传入的name也可以是别名,也需要做转换
   // 注意 beanName 和 name 变量的区别,beanName是经过处理的,
   // 经过处理的beanName就直接对应singletonObjects中的key
   // 以及解决别名问题
   final String beanName = transformedBeanName(name);
   Object bean;

   // Eagerly check singleton cache for manually registered singletons.
   // 根据beanName尝试从singletonObjects获取Bean
   // 获取不到则再尝试从earlySingletonObjects,singletonFactories 从获取Bean
   // 这段代码和解决循环依赖有关，当属性注入是
   Object sharedInstance = getSingleton(beanName);
   // 第一次进入sharedInstance肯定为null
   if (sharedInstance != null && args == null) {
      // 如果beanName的实例存在于缓存中
      if (logger.isDebugEnabled()) {
         if (isSingletonCurrentlyInCreation(beanName)) {
            logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                  "' 该bean还没有完全实例化 - 用于解决循环依赖");
         }
         else {
            logger.debug("返回已经创建的 instance of singleton bean '" + beanName + "'说明：此处的beanName可能是@bean标注的方法所在的配置类的名");
         }
      }
      // 如果sharedInstance不为null,也就是非第一次进入
      // 为什么要调用 getObjectForBeanInstance 方法,
      // 判断当前Bean是不是FactoryBean,如果是,那么要不要调用getObject方法
      // 因为传入的name变量如果是(&+beanName),beanName变量就是(beanName),
      // 也就是说,程序在这里要返回FactoryBean
      // 如果传入的name变量(beanName),beanName变量也是(beanName),
      // 之前获取的sharedInstance可能是FactoryBean,需要通过sharedInstance来获取对应的Bean
      // 如果传入的name变量(beanName),beanName变量也是(beanName),
      // 获取的sharedInstance就是对应的Bean的话,就直接返回Bean
      // 返回beanName对应的实例对象（主要用于FactoryBean的特殊处理，普通Bean会直接返回sharedInstance本身）
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
      // scope为prototype的循环依赖校验：如果beanName已经正在创建Bean实例中，而此时我们又要再一次创建beanName的实例，则代表出现了循环依赖，需要抛出异常。
      // 例子：如果存在A中有B的属性，B中有A的属性，那么当依赖注入的时候，就会产生当A还未创建完的时候因为对于B的创建再次返回创建A，造成循环依赖
      // 判断是否循环依赖
      if (isPrototypeCurrentlyInCreation(beanName)) {
         throw new BeanCurrentlyInCreationException(beanName);
      }

      // Check if bean definition exists in this factory.
      // 获取父BeanFactory,一般情况下,父BeanFactory为null,
      // 如果存在父BeanFactory,就先去父级容器去查找
      BeanFactory parentBeanFactory = getParentBeanFactory();
      // 如果parentBeanFactory存在，并且beanName在当前BeanFactory不存在Bean定义，则尝试从parentBeanFactory中获取bean实例
      if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
         // Not found -> check parent.
         // 将别名解析成真正的beanName
         String nameToLookup = originalBeanName(name);
         if (parentBeanFactory instanceof AbstractBeanFactory) {
            return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                  nameToLookup, requiredType, args, typeCheckOnly);
         }
         // 尝试在parentBeanFactory中获取bean对象实例
         else if (args != null) {
            // Delegation to parent with explicit args.
            return (T) parentBeanFactory.getBean(nameToLookup, args);
         }
         else {
            // No args -> delegate to standard getBean method.
            return parentBeanFactory.getBean(nameToLookup, requiredType);
         }
      }

      // 创建的Bean是否需要进行类型验证,一般情况下都不需要
      // 如果不是仅仅做类型检测，而是创建bean实例，这里要将beanName放到alreadyCreated缓存
      if (!typeCheckOnly) {
         // 标记 bean 已经被创建
         markBeanAsCreated(beanName);
      }

      try {
         // 获取其父类Bean定义,子类合并父类公共属性
         // 根据beanName重新获取MergedBeanDefinition（步骤6将MergedBeanDefinition删除了，这边获取一个新的）
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         // 检查MergedBeanDefinition
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
         // 获取当前Bean依赖的Bean的名称 ,@DependsOn
         // 拿到当前bean依赖的bean名称集合，在实例化自己之前，需要先实例化自己依赖的bean
         String[] dependsOn = mbd.getDependsOn();
         if (dependsOn != null) {
            // 遍历当前bean依赖的bean名称集合
            for (String dep : dependsOn) {
               // 检查dep是否依赖于beanName，即检查是否存在循环依赖
               if (isDependent(beanName, dep)) {
                  // 如果是循环依赖则抛异常
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
               }
               // 如果当前Bean依赖其他Bean,把被依赖Bean注册给当前Bean
               // 将dep和beanName的依赖关系注册到缓存中
               registerDependentBean(dep, beanName);
               try {
                  // 先去创建所依赖的Bean
                  // 获取dep对应的bean实例，如果dep还没有创建bean实例，则创建dep的bean实例
                  getBean(dep);
               }
               catch (NoSuchBeanDefinitionException ex) {
                  throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
               }
            }
         }

         // Create bean instance.
         // 针对不同的scope进行bean的创建
         if (mbd.isSingleton()) {
            // 创建单例Bean
            // scope为singleton的bean创建（新建了一个ObjectFactory，并且重写了getObject方法）
            sharedInstance = getSingleton(beanName, () -> {
               try {
                  // 创建Bean实例
                  return createBean(beanName, mbd, args);
               }
               catch (BeansException ex) {
                  // Explicitly remove instance from singleton cache: It might have been put there
                  // eagerly by the creation process, to allow for circular reference resolution.
                  // Also remove any beans that received a temporary reference to the bean.
                  destroySingleton(beanName);
                  throw ex;
               }
            });
            // 如果sharedInstance不为null,也就是非第一次进入
            // 为什么要调用 getObjectForBeanInstance 方法,
            // 判断当前Bean是不是FactoryBean,如果是,那么要不要调用getObject方法
            // 因为传入的name变量如果是(&+beanName),beanName变量就是(beanName),
            // 也就是说,程序在这里要返回FactoryBean
            // 如果传入的name变量(beanName),beanName变量也是(beanName),
            // 之前获取的sharedInstance可能是FactoryBean,需要通过sharedInstance来获取对应的Bean
            // 如果传入的name变量(beanName),beanName变量也是(beanName),
            // 获取的sharedInstance就是对应的Bean的话,就直接返回Bean
            // 返回beanName对应的实例对象
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

         // 创建prototype Bean,每次都会创建一个新的对象
         // scope为prototype的bean创建
         else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            // scope为prototype的bean创建
            Object prototypeInstance = null;
            try {
               // 回调beforePrototypeCreation方法，注册当前创建的原型对象
               // 创建实例前的操作（将beanName保存到prototypesCurrentlyInCreation缓存中）
               beforePrototypeCreation(beanName);
               // 创建Bean实例
               prototypeInstance = createBean(beanName, mbd, args);
            }
            finally {
               // 回调 afterPrototypeCreation 方法，告诉容器该Bean的原型对象不再创建
               // 创建实例后的操作（将创建完的beanName从prototypesCurrentlyInCreation缓存中移除）
               afterPrototypeCreation(beanName);
            }
            // 返回beanName对应的实例对象
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         else {
            // 如果既不是单例Bean,也不是prototype,则获取其Scope
            // 其他scope的bean创建，可能是request之类的
            // 根据scopeName，从缓存拿到scope实例
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            if (scope == null) {
               throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
            }
            try {
               // 仅适用于WebApplicationContext环境,
               // request：请求作⽤域
               // session：session作⽤域
               // application：全局作⽤域创建对象
               Object scopedInstance = scope.get(beanName, () -> {
                  // 创建实例前的操作（将beanName保存到prototypesCurrentlyInCreation缓存中）
                  beforePrototypeCreation(beanName);
                  try {
                     // 创建bean实例
                     return createBean(beanName, mbd, args);
                  }
                  finally {
                     // 创建实例后的操作（将创建完的beanName从prototypesCurrentlyInCreation缓存中移除）
                     afterPrototypeCreation(beanName);
                  }
               });
               // 返回beanName对应的实例对象
               bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // Check if required type matches the type of the actual bean instance.
   // 对创建的Bean进行类型检查
   // 检查所需类型是否与实际的bean对象的类型匹配
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         // 尝试将创建的bean转换为requiredType指明的类型
         // 类型不对，则尝试转换bean类型
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
      }
      catch (TypeMismatchException ex) {
         if (logger.isDebugEnabled()) {
            logger.debug("Failed to convert bean '" + name + "' to required type '" +
                  ClassUtils.getQualifiedName(requiredType) + "'", ex);
         }
         throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
      }
   }
   // 返回创建出来的bean实例对象
   return (T) bean;
}
```

1. 根据beanName尝试从singletonObjects获取Bean，和解决循环依赖有关
2. 非单例作用域的bean，存在循环引用则抛异常
3. 根据beanName先从父容器中查找
4. 如果要创建的bean含有@DependsOn注解，先加载@DependsOn指定的Bean
5. 根据bean的作用域采用不同的策略来创建
6. FactoryBean返回的Bean实例特殊处理

分析根据beanName尝试从singletonObjects获取Bean

```java
public Object getSingleton(String beanName) {
   return getSingleton(beanName, true);
}
```



```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   //singletonObjects 就是Spring内部用来存放单例Bean的对象池,key为beanName，value为Bean
   Object singletonObject = this.singletonObjects.get(beanName);
   // singletonsCurrentlyInCreation 存放了当前正在创建的bean的BeanName
   // 如果单例对象缓存中没有，并且该beanName对应的单例bean正在创建中
   if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      synchronized (this.singletonObjects) {
         // earlySingletonObjects 是早期单例Bean的缓存池,此时Bean已经被创建(newInstance),但是还没有完成初始化
         // key为beanName，value为Bean
         // earlySingletonObjects为了循环依赖的要注入的属性的校验
         // 从早期单例对象缓存中获取单例对象（之所称成为早期单例对象，是因为earlySingletonObjects里
         // 的对象的都是通过提前曝光的ObjectFactory创建出来的，还未进行属性填充等操作）
         singletonObject = this.earlySingletonObjects.get(beanName);
         // 是否允许早期依赖
         // 如果在早期单例对象缓存中也没有，并且允许创建早期单例对象引用
         if (singletonObject == null && allowEarlyReference) {
            // singletonFactories 单例工厂的缓存,key为beanName,value 为ObjectFactory
            // 从单例工厂缓存中获取beanName的单例工厂
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
               // 获取早期Bean
               // 如果存在单例对象工厂，则通过工厂创建一个单例对象
               singletonObject = singletonFactory.getObject();
               // 将早期Bean放到earlySingletonObjects中
               // 将通过单例对象工厂创建的单例对象，放到早期单例对象缓存中
               this.earlySingletonObjects.put(beanName, singletonObject);
               // 移除该beanName对应的单例对象工厂，因为该单例工厂已经创建了一个实例对象，并且放到earlySingletonObjects缓存了，
               // 因此，后续获取beanName的单例对象，可以通过earlySingletonObjects缓存拿到，不需要在用到该单例工厂
               this.singletonFactories.remove(beanName);
            }
         }
      }
   }
   return singletonObject;
}
```

其中singletonFactories的赋值逻辑是在Bean实例化之后（Bean的实例化参考: todo）

```java
// 完成将实例化的bean提前暴露出去
// 这样就解决了单例bean非构造函数的循环引用问题；
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
   if (logger.isDebugEnabled()) {
      logger.debug("Eagerly caching bean '" + beanName +
            "' to allow for resolving potential circular references");
   }
   // 提前曝光beanName的ObjectFactory，用于解决循环引用
   // 应用后置处理器SmartInstantiationAwareBeanPostProcessor，允许返回指定bean的早期引用，若没有则直接返回bean
   if(logger.isInfoEnabled()){
      logger.info("提前曝光beanName："+beanName+"的ObjectFactory，用于解决循环引用");
   }
   addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```



存在三级缓存来处理循环引用的问题：

- 第一层缓存（singletonObjects）：单例对象缓存池，已经实例化并且属性赋值，这里的对象是成熟对象
- 第二层缓存（earlySingletonObjects）：单例对象缓存池，已经实例化但尚未属性赋值，这里的对象是半成品对象
- 第三层缓存（singletonFactories）: 单例工厂的缓存

循环依赖不能解决的两种场景：

- 非单例的作用域的bean的循环依赖。因为spring不会缓存非单例的作用域的bean，而spring中循环依赖的解决正是通过缓存来实现的。
- 构造方法引起的循环依赖。因为在调用构造方法之前还未将其放入三级缓存之中，所以后续的依赖调用构造方法的时候并不能从三级缓存中获取到依赖的Bean，因此不能解决。



## 根据Bean的作用域创建Bean

| 作用域      | 使用方式                                                | 描述                                                         |
| ----------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| singleton   | 默认方式                                                | 单例作用域，在整个Spring IoC容器仅存在一个Bean实例，Bean以单例方式存在 |
| prototype   | @Scope(ConfigurableListableBeanFactory.SCOPE_PROTOTYPE) | 原型作用域，每次从容器中调用Bean时，都返回一个新的实例。即每次调用getBean时，相当于执行newXxxBean方法 |
| request     | @Scope(WebApplicationContext.SCOPE_REQUEST)             | 请求作用域，每次Http请求都会创建一个新的Bean                 |
| session     | @Scope(WebApplicationContext.SCOPE_SESSION)             | 会话作用域，同一个Http Session共享一个Bean，不同Session使用不同的Bean |
| application | @Scope(WebApplicationContext.SCOPE_APPLICATION)         | 全局作用域，在一个Http Servlet Context中，定义一个Bean.      |

分析根据bean的作用域采用不同的策略来创建

### 单例Bean的创建



```java
if (mbd.isSingleton()) {
   // 创建单例Bean
   // scope为singleton的bean创建（新建了一个ObjectFactory，并且重写了getObject方法）
   sharedInstance = getSingleton(beanName, () -> {
      try {
         // 创建Bean实例
         return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
         // Explicitly remove instance from singleton cache: It might have been put there
         // eagerly by the creation process, to allow for circular reference resolution.
         // Also remove any beans that received a temporary reference to the bean.
         destroySingleton(beanName);
         throw ex;
      }
   });
   // 如果sharedInstance不为null,也就是非第一次进入
   // 为什么要调用 getObjectForBeanInstance 方法,
   // 判断当前Bean是不是FactoryBean,如果是,那么要不要调用getObject方法
   // 因为传入的name变量如果是(&+beanName),beanName变量就是(beanName),
   // 也就是说,程序在这里要返回FactoryBean
   // 如果传入的name变量(beanName),beanName变量也是(beanName),
   // 之前获取的sharedInstance可能是FactoryBean,需要通过sharedInstance来获取对应的Bean
   // 如果传入的name变量(beanName),beanName变量也是(beanName),
   // 获取的sharedInstance就是对应的Bean的话,就直接返回Bean
   // 返回beanName对应的实例对象
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

1. 从单例池中获取Bean，处理获取则创建单例Bean
2. FactoryBean返回的Bean实例特殊处理

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
   Assert.notNull(beanName, "Bean name must not be null");
   synchronized (this.singletonObjects) {
      //判断单例Bean是否已经存在,如果存在,则直接返回
      Object singletonObject = this.singletonObjects.get(beanName);
      if (singletonObject == null) {
         // beanName对应的bean实例不存在于缓存中，则进行Bean的创建
         if (this.singletonsCurrentlyInDestruction) {
            // 当bean工厂的单例处于destruction状态时，不允许进行单例bean创建，抛出异常
            throw new BeanCreationNotAllowedException(beanName,
                  "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                  "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
         }
         // 创建单例之前调用该方法,将此Bean标记为正在创建中,用来检测循环依赖
         // 向singletonsCurrentlyInCreation中添加该beanName
         if(logger.isInfoEnabled()){
            logger.info("标记["+beanName+"]为正在处理，" +
                  "说明：用于解决循环依赖");
         }
         beforeSingletonCreation(beanName);
         boolean newSingleton = false;
         // suppressedExceptions用于记录异常相关信息
         boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
         if (recordSuppressedExceptions) {
            this.suppressedExceptions = new LinkedHashSet<>();
         }
         try {
            // 执行singletonFactory的getObject方法获取bean实例
            singletonObject = singletonFactory.getObject();
            // 标记为新的单例对象
            newSingleton = true;
         }
         catch (IllegalStateException ex) {
            // Has the singleton object implicitly appeared in the meantime ->
            // if yes, proceed with it since the exception indicates that state.
            singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
               throw ex;
            }
         }
         catch (BeanCreationException ex) {
            if (recordSuppressedExceptions) {
               for (Exception suppressedException : this.suppressedExceptions) {
                  ex.addRelatedCause(suppressedException);
               }
            }
            throw ex;
         }
         finally {
            if (recordSuppressedExceptions) {
               this.suppressedExceptions = null;
            }
            // 创建单例之后调用该方法,将单例标记为不在创建中
            // 将beanName从singletonsCurrentlyInCreation中移除
            if(logger.isInfoEnabled()){
               logger.info("标记["+beanName+"]为已经处理，" +
                     "说明：用于解决循环依赖");
            }
            afterSingletonCreation(beanName);

         }
         if (newSingleton) {
            // 加入到单例池容器中
            if(logger.isInfoEnabled()){
               logger.info("将["+beanName+"]加入到单例池singletonObjects中");
            }
            addSingleton(beanName, singletonObject);

         }
      }
      // 返回创建出来的单例对象
      return singletonObject;
   }
}
```

1. 将Bean放入到Set中，标记单例Bean正在创建，循环依赖用
2. 调用AbstractBeanFactory#createBean真正实例化以及初始化Bean
3. 将Bean从Set中移除，标记单例Bean创建完毕
4. 将Bean放入到单例池中

### 原型Bean的创建

```java
else if (mbd.isPrototype()) {
   // It's a prototype -> create a new instance.
   // scope为prototype的bean创建
   Object prototypeInstance = null;
   try {
      // 回调beforePrototypeCreation方法，注册当前创建的原型对象
      // 创建实例前的操作（将beanName保存到prototypesCurrentlyInCreation缓存中）
      beforePrototypeCreation(beanName);
      // 创建Bean实例
      prototypeInstance = createBean(beanName, mbd, args);
   }
   finally {
      // 回调 afterPrototypeCreation 方法，告诉容器该Bean的原型对象不再创建
      // 创建实例后的操作（将创建完的beanName从prototypesCurrentlyInCreation缓存中移除）
      afterPrototypeCreation(beanName);
   }
   // 返回beanName对应的实例对象
   bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
```

1. 标记非单例Bean正在创建
2. 调用AbstractBeanFactory#createBean真正实例化以及初始化Bean
3. 移除标记非单例Bean正在创建
4. FactoryBean返回的Bean实例特殊处理

### WebApplicationContext环境的request、session、application类型的Bean的创建

```java
String scopeName = mbd.getScope();
final Scope scope = this.scopes.get(scopeName);
if (scope == null) {
   throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
}
try {
   // 仅适用于WebApplicationContext环境,
   // request：请求作⽤域
   // session：session作⽤域
   // application：全局作⽤域创建对象
   Object scopedInstance = scope.get(beanName, () -> {
      // 创建实例前的操作（将beanName保存到prototypesCurrentlyInCreation缓存中）
      beforePrototypeCreation(beanName);
      try {
         // 创建bean实例
         return createBean(beanName, mbd, args);
      }
      finally {
         // 创建实例后的操作（将创建完的beanName从prototypesCurrentlyInCreation缓存中移除）
         afterPrototypeCreation(beanName);
      }
   });
   // 返回beanName对应的实例对象
   bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
}
```

1. 根据作用域获取对应的Scope
2. 从Scope中根据beanName获取Bean，没有获取到就创建Bean，并将bean放入到对应的作用域中
3. 标记非单例Bean正在创建
4. 调用AbstractBeanFactory#createBean真正实例化以及初始化Bean
5. 移除标记非单例Bean正在创建
6. FactoryBean返回的Bean实例特殊处理

以应用维度的作用域来举例从从Scope中根据beanName获取Bean的逻辑

```java
public Object get(String name, ObjectFactory<?> objectFactory) {
   Object scopedObject = this.servletContext.getAttribute(name);
   if (scopedObject == null) {
      scopedObject = objectFactory.getObject();
      this.servletContext.setAttribute(name, scopedObject);
   }
   return scopedObject;
}
```





## 将BeanDefinition转化为Bean实例

继续分析调用AbstractBeanFactory#createBean真正实例化以及初始化Bean的逻辑

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
      throws BeanCreationException {

   if (logger.isDebugEnabled()) {
      logger.debug("开始创建 instance of bean '" + beanName + "'");
   }
   RootBeanDefinition mbdToUse = mbd;

   // Make sure bean class is actually resolved at this point, and
   // clone the bean definition in case of a dynamically resolved Class
   // which cannot be stored in the shared merged bean definition.
   // 为指定的bean定义解析bean类，将bean类名称解析为Class引用
   //（如果需要,并将解析后的Class存储在bean定义中以备将来使用)
   // 也就是通过类加载去加载这个Class
   // 判断需要创建的Bean是否可以实例化，这个类是否可以通过类装载器来载入
   Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
   if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
      // 如果resolvedClass存在，并且mdb的beanClass类型不是Class，
      // 并且mdb的beanClass不为空（则代表beanClass存的是Class的name）,
      // 则使用mdb深拷贝一个新的RootBeanDefinition副本，
      // 并且将解析的Class赋值给拷贝的RootBeanDefinition副本的beanClass属性，
      // 该拷贝副本取代mdb用于后续的操作
      mbdToUse = new RootBeanDefinition(mbd);
      mbdToUse.setBeanClass(resolvedClass);
   }

   // Prepare method overrides.
   try {
      // 校验和准备 Bean 中的方法覆盖
      mbdToUse.prepareMethodOverrides();
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
            beanName, "Validation of method overrides failed", ex);
   }

   try {
      // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      // 执行BeanPostProcessors ,
      // 给 BeanPostProcessors 一个机会直接返回代理对象来代替Bean实例
      // 在Bean还没有开始实例化之前执行
      // InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation方法,
      // 这个方法可能会直接返回Bean
      // 如果这里直接返回了Bean,那么这里返回的Bean可能是被经过处理的Bean(可能是代理对象)
      // 在这里需要注意Spring方法中的两个单词:
      // Instantiation 和 Initialization , 实例化和初始化, 先实例化,然后初始化
      // 内部执行了
      // InstantiationAwareBeanPostProcessor的一个相关方法，
      //    分别是：
      //    beanPostProcessorsBeforeInstantiation：
      //          该方法在Bean实例化之前调用，给BeanPostProcessor一个机会去创建代理对象来代理Bean
      //          也就是说，经过调用InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation方法之后，
      //          生成的Bean可能已经不是我们的原生Bean，而是一个代理对象。
      // 实例化前的处理，给InstantiationAwareBeanPostProcessor一个机会返回代理对象来替代真正的bean实例，达到“短路”效果
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         // 如果bean不为null,则直接返回,不在做后续处理
         return bean;
      }
   }
   catch (Throwable ex) {
      throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
            "BeanPostProcessor before instantiation of bean failed", ex);
   }

   try {
      // 创建Bean,真正执行bean的创建
      Object beanInstance = doCreateBean(beanName, mbdToUse, args);
      if (logger.isDebugEnabled()) {
         logger.debug("Finished creating instance of bean '" + beanName + "'");
      }
      return beanInstance;
   }
   catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
      // A previously detected exception with proper bean creation context already,
      // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
   }
}
```

1. 调用resolveBeforeInstantiation来替代真正的bean实例，其中调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法，在AnnotationAwareAspectJAutoProxyCreator的postProcessBeforeInstantiation方法中获取所有的AOP的advisor以及事务的advisor
2. 调用doCreateBean来真正实例化以及初始化Bean

分析resolveBeforeInstantiation方法

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
   Object bean = null;
   if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
      // Make sure bean class is actually resolved at this point.
      // mbd不是合成的，并且BeanFactory中存在InstantiationAwareBeanPostProcessor
      if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
         // 解析beanName对应的Bean实例的类型
         Class<?> targetType = determineTargetType(beanName, mbd);
         if (targetType != null) {
            // 实例化前的后置处理器应用（处理InstantiationAwareBeanPostProcessor）
            bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
            if (bean != null) {
               // 如果返回的bean不为空，会跳过Spring默认的实例化过程，
               // 所以只能在这里调用BeanPostProcessor实现类的postProcessAfterInitialization方法
               if(logger.isInfoEnabled()){
                  logger.info("--但是返回的实例不为空，则调用postProcessorsAfterInitialization方法后，直接返回，创建对象操作结束");
               }
               bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
            }
         }
      }
      // 如果bean不为空，则将beforeInstantiationResolved赋值为true，代表在实例化之前已经解析
      mbd.beforeInstantiationResolved = (bean != null);
   }
   return bean;
}
```

```java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
   for (BeanPostProcessor bp : getBeanPostProcessors()) {
      // 应用InstantiationAwareBeanPostProcessor后置处理器，允许postProcessBeforeInstantiation方法返回bean对象的代理
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
         // 执行postProcessBeforeInstantiation方法，在Bean实例化前操作，
         // 该方法可以返回一个构造完成的Bean实例，从而不会继续执行创建Bean实例的“正规的流程”，达到“短路”的效果。
         InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
         // 除了AnnotationAwareAspectJAutoProxyCreator,其他都为空
         if(logger.isInfoEnabled()){
            logger.info("对beanName为："+beanName+" 的bd调用InstantiationAwareBeanPostProcessor的实例"+bp.getClass().getSimpleName()+"的postProcessBeforeInstantiation方法");
         }
         Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
         if (result != null) {
            // 如果result不为空，也就是有后置处理器返回了bean实例对象，则会跳过Spring默认的实例化过程
            if(logger.isInfoEnabled()){
               logger.info("--但是"+bp.getClass().getSimpleName()+"的返回值不为空，直接返回");
            }
            return result;
         }
      }
   }
   return null;
}
```

调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法，在AnnotationAwareAspectJAutoProxyCreator的postProcessBeforeInstantiation方法中获取所有的AOP的advisor以及事务的advisor详见todo

继续分析doCreateBean来真正实例化以及初始化Bean

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
   // 拿到创建好的Bean实例
   final Object bean = instanceWrapper.getWrappedInstance();
   // 拿到Bean实例的类型
   Class<?> beanType = instanceWrapper.getWrappedClass();
   if (beanType != NullBean.class) {
      mbd.resolvedTargetType = beanType;
   }

   // Allow post-processors to modify the merged bean definition.
   synchronized (mbd.postProcessingLock) {
      if (!mbd.postProcessed) {
         try {
            // 应用后置处理器MergedBeanDefinitionPostProcessor，允许修改MergedBeanDefinition，
            // Autowired注解正是通过此方法实现注入类型的预解析
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
         }
         catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                  "Post-processing of merged bean definition failed", ex);
         }
         mbd.postProcessed = true;
      }
   }

   // Eagerly cache singletons to be able to resolve circular references
   // even when triggered by lifecycle interfaces like BeanFactoryAware.
   // 完成将实例化的bean提前暴露出去
   // 这样就解决了单例bean非构造函数的循环引用问题；
   boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
         isSingletonCurrentlyInCreation(beanName));
   if (earlySingletonExposure) {
      if (logger.isDebugEnabled()) {
         logger.debug("Eagerly caching bean '" + beanName +
               "' to allow for resolving potential circular references");
      }
      // 提前曝光beanName的ObjectFactory，用于解决循环引用
      // 应用后置处理器SmartInstantiationAwareBeanPostProcessor，允许返回指定bean的早期引用，若没有则直接返回bean
      if(logger.isInfoEnabled()){
         logger.info("提前曝光beanName："+beanName+"的ObjectFactory，用于解决循环引用");
      }
      addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
      // 初始化bean的各种注入或者setXX参数
      // 调用 InstantiationAwareBeanPostProcessor 的 postProcessAfterInstantiation 方法
      // 对bean进行属性填充；其中，可能存在依赖于其他bean的属性，则会递归初始化依赖的bean实例
      if(logger.isInfoEnabled()){
         logger.info("给该类填充属性，该类名为："+beanName);
      }
      populateBean(beanName, mbd, instanceWrapper);
      // 属性注入完成后，这一步其实就是处理各种回调了
      // 调用注入类的init-method方法
      // 同时在执行init-method之前会
      // 调用applyBeanPostProcessorsBeforeInitialization
      // 完成bean使用前的处理操作，
      // 调用applyBeanPostProcessorsAfterInitialization
      // 完成bean初始化后的操作；
      if(logger.isInfoEnabled()){
         logger.info("执行init-method,@PostConstruct，applyBeanPostProcessorsBeforeInitialization,applyBeanPostProcessorsAfterInitialization等方法");
      }
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   catch (Throwable ex) {
      if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
         throw (BeanCreationException) ex;
      }
      else {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
      }
   }

   if (earlySingletonExposure) {
      // 如果允许提前曝光实例，则进行循环依赖检查
      Object earlySingletonReference = getSingleton(beanName, false);
      // earlySingletonReference只有在当前解析的bean存在循环依赖的情况下才会不为空
      if (earlySingletonReference != null) {
         if (exposedObject == bean) {
            // 如果exposedObject没有在initializeBean方法中被增强，则不影响之前的循环引用
            exposedObject = earlySingletonReference;
         }
         else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            // 如果exposedObject在initializeBean方法中被增强 && 不允许在循环引用的情况下使用注入原始bean实例
            // && 当前bean有被其他bean依赖
            // 拿到依赖当前bean的所有bean的beanName数组
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
               // 尝试移除这些bean的实例，因为这些bean依赖的bean已经被增强了，他们依赖的bean相当于脏数据
               if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                  // 移除失败的添加到 actualDependentBeans
                  actualDependentBeans.add(dependentBean);
               }
            }
            if (!actualDependentBeans.isEmpty()) {
               // 如果存在移除失败的，则抛出异常，因为存在bean依赖了“脏数据”
               throw new BeanCurrentlyInCreationException(beanName,
                     "Bean with name '" + beanName + "' has been injected into other beans [" +
                     StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                     "] in its raw version as part of a circular reference, but has eventually been " +
                     "wrapped. This means that said other beans do not use the final version of the " +
                     "bean. This is often the result of over-eager type matching - consider using " +
                     "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
         }
      }
   }

   // Register bean as disposable.
   try {
      // 注册bean为可销毁的bean，bean销毁时，会回调destroy-method
      // 注册用于销毁的bean，执行销毁操作的有三种：自定义destroy方法、DisposableBean接口、DestructionAwareBeanPostProcessor
      registerDisposableBeanIfNecessary(beanName, bean, mbd);
   }
   catch (BeanDefinitionValidationException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
   }

   return exposedObject;
}
```

1. 根据不同的策略来实例化Bean
2. 调用MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition方法，其中预解析@Autowire、@Value与预解析@PostConstruct、@PreDestroy注解
3. 将实例化的bean提前暴露出去，暴露到singletonFactories中，用于解决循环依赖
4. 注入属性，其中有参构造XML注入属性与注解注入属性的区别就是这里
5. 初始化Bean，其中有很多逻辑后续分析
6. 返回创建的Bean

### 实例化Bean

参考[todo]()

### 预解析注解

调用MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition方法，其中预解析@Autowire、@Value与预解析@PostConstruct、@PreDestroy注解

 AutowiredAnnotationBeanPostProcessor的postProcessMergedBeanDefinition

```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
   // 在指定Bean中查找使用@Autowired、@Value注解的元数据
   InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
   // 检查元数据中的注解信息
   metadata.checkConfigMembers(beanDefinition);
}
```

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
   // Fall back to class name as cache key, for backwards compatibility with custom callers.
   // 设置cacheKey的值（beanName 或者 className）
   String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
   // Quick check on the concurrent map first, with minimal locking.
   // 检查beanName对应的InjectionMetadata是否已经存在于缓存中
   InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
   // 检查InjectionMetadata是否需要刷新（为空或者class变了）
   if (InjectionMetadata.needsRefresh(metadata, clazz)) {
      synchronized (this.injectionMetadataCache) {
         // 加锁后，再次从缓存中获取beanName对应的InjectionMetadata
         metadata = this.injectionMetadataCache.get(cacheKey);
         // 加锁后，再次检查InjectionMetadata是否需要刷新
         if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            if (metadata != null) {
               // 如果需要刷新，并且metadata不为空，则先移除
               metadata.clear(pvs);
            }
            // 解析@Autowired、@Value注解的信息，生成元数据（包含clazz和clazz里解析到的注入的元素，
            // 这里的元素包括AutowiredFieldElement和AutowiredMethodElement）
            metadata = buildAutowiringMetadata(clazz);
            // 将解析的元数据放到injectionMetadataCache缓存，以备复用，每一个类只解析一次
            this.injectionMetadataCache.put(cacheKey, metadata);
         }
      }
   }
   return metadata;
}
```

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
   // 用于存放所有解析到的注入的元素的变量
   List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
   Class<?> targetClass = clazz;

   // 循环遍历
   do {
      // 定义存放当前循环的Class注入的元素(有序)
      final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

      // 如果targetClass的属性上有@Autowired、@Value注解，则用工具类获取注解信息
      ReflectionUtils.doWithLocalFields(targetClass, field -> {
         // 获取field上的@Autowired、@Value注解信息
         AnnotationAttributes ann = findAutowiredAnnotation(field);
         if (ann != null) {
            // 校验field是否被static修饰，如果是则直接返回，因为@Autowired、@Value注解不支持static修饰的field
            if (Modifier.isStatic(field.getModifiers())) {
               if (logger.isWarnEnabled()) {
                  logger.warn("Autowired annotation is not supported on static fields: " + field);
               }
               return;
            }
            // 获取@Autowired注解的required的属性值（required：值为true时，如果没有找到bean时，自动装配应该失败；false则不会）
            boolean required = determineRequiredStatus(ann);
            // 将field、required封装成AutowiredFieldElement，添加到currElements
            currElements.add(new AutowiredFieldElement(field, required));
         }
      });

      // 如果targetClass的方法上有@Autowired、@Value注解，则用工具类获取注解信息
      ReflectionUtils.doWithLocalMethods(targetClass, method -> {
         // 找到桥接方法
         Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
         // 判断方法的可见性，如果不可见则直接返回
         if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
            return;
         }
         // 获取method上的@Autowired、@Value注解信息
         AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
         if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
            // 校验method是否被static修饰，如果是则直接返回，因为@Autowired、@Value注解不支持static修饰的method
            if (Modifier.isStatic(method.getModifiers())) {
               if (logger.isWarnEnabled()) {
                  logger.warn("Autowired annotation is not supported on static methods: " + method);
               }
               return;
            }
            if (method.getParameterCount() == 0) {
               if (logger.isWarnEnabled()) {
                  logger.warn("Autowired annotation should only be used on methods with parameters: " +
                        method);
               }
            }
            // 获取@Autowired注解的required的属性值
            boolean required = determineRequiredStatus(ann);
            // 获取method的属性描述器
            PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
            // 将method、required、pd封装成AutowiredMethodElement，添加到currElements
            currElements.add(new AutowiredMethodElement(method, required, pd));
         }
      });

      // 将本次循环获取到的注解信息添加到elements
      elements.addAll(0, currElements);
      //  在解析完targetClass之后，递归解析父类，将所有的@Autowired的属性和方法收集起来，且类的层级越高其属性会被越优先注入
      targetClass = targetClass.getSuperclass();
   }
   // 递归解析targetClass父类(直至父类为Object结束)
   while (targetClass != null && targetClass != Object.class);

   // 将clazz和解析到的注入的元素封装成InjectionMetadata
   return new InjectionMetadata(clazz, elements);
}
```

缓存已解析的@Autowired、@Value注解信息

```java
public void checkConfigMembers(RootBeanDefinition beanDefinition) {
   Set<InjectedElement> checkedElements = new LinkedHashSet<>(this.injectedElements.size());
   // 遍历检查所有要注入的元素
   for (InjectedElement element : this.injectedElements) {
      Member member = element.getMember();
      // 如果beanDefinition的externallyManagedConfigMembers属性不包含该member
      if (!beanDefinition.isExternallyManagedConfigMember(member)) {
         // 将该member添加到beanDefinition的externallyManagedConfigMembers属性
         beanDefinition.registerExternallyManagedConfigMember(member);
         // 并将element添加到checkedElements
         checkedElements.add(element);
         if (logger.isDebugEnabled()) {
            logger.debug("Registered injected element on class [" + this.targetClass.getName() + "]: " + element);
         }
      }
   }
   // 赋值给checkedElements（检查过的元素）
   this.checkedElements = checkedElements;
}
```

 预解析@Autowired、@Value注解完毕

 CommonAnnotationBeanPostProcessor的postProcessMergedBeanDefinition

```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
   // 解析@PostConstruct注解以及@PreDestroy注解;
   LifecycleMetadata metadata = findLifecycleMetadata(beanType);
   // 缓存起来
   metadata.checkConfigMembers(beanDefinition);
}
```

```java
private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
   if (this.lifecycleMetadataCache == null) {
      // Happens after deserialization, during destruction...
      return buildLifecycleMetadata(clazz);
   }
   // Quick check on the concurrent map first, with minimal locking.
   LifecycleMetadata metadata = this.lifecycleMetadataCache.get(clazz);
   if (metadata == null) {
      synchronized (this.lifecycleMetadataCache) {
         metadata = this.lifecycleMetadataCache.get(clazz);
         if (metadata == null) {
            // 解析@PostConstruct注解以及@PreDestroy注解;
            metadata = buildLifecycleMetadata(clazz);
            this.lifecycleMetadataCache.put(clazz, metadata);
         }
         return metadata;
      }
   }
   return metadata;
}
```

```java
private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
   final boolean debug = logger.isDebugEnabled();
   List<LifecycleElement> initMethods = new ArrayList<>();
   List<LifecycleElement> destroyMethods = new ArrayList<>();
   Class<?> targetClass = clazz;

   do {
      final List<LifecycleElement> currInitMethods = new ArrayList<>();
      final List<LifecycleElement> currDestroyMethods = new ArrayList<>();

      ReflectionUtils.doWithLocalMethods(targetClass, method -> {
         if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
            LifecycleElement element = new LifecycleElement(method);
            currInitMethods.add(element);
            if (debug) {
               logger.debug("Found init method on class [" + clazz.getName() + "]: " + method);
            }
         }
         if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
            currDestroyMethods.add(new LifecycleElement(method));
            if (debug) {
               logger.debug("Found destroy method on class [" + clazz.getName() + "]: " + method);
            }
         }
      });

      initMethods.addAll(0, currInitMethods);
      destroyMethods.addAll(currDestroyMethods);
      targetClass = targetClass.getSuperclass();
   }
   while (targetClass != null && targetClass != Object.class);

   return new LifecycleMetadata(clazz, initMethods, destroyMethods);
}
```

缓存处理的@PostConstruct注解以及@PreDestroy注解

```java
public void checkConfigMembers(RootBeanDefinition beanDefinition) {
   Set<LifecycleElement> checkedInitMethods = new LinkedHashSet<>(this.initMethods.size());
   for (LifecycleElement element : this.initMethods) {
      String methodIdentifier = element.getIdentifier();
      if (!beanDefinition.isExternallyManagedInitMethod(methodIdentifier)) {
         beanDefinition.registerExternallyManagedInitMethod(methodIdentifier);
         checkedInitMethods.add(element);
         if (logger.isDebugEnabled()) {
            logger.debug("Registered init method on class [" + this.targetClass.getName() + "]: " + element);
         }
      }
   }
   Set<LifecycleElement> checkedDestroyMethods = new LinkedHashSet<>(this.destroyMethods.size());
   for (LifecycleElement element : this.destroyMethods) {
      String methodIdentifier = element.getIdentifier();
      if (!beanDefinition.isExternallyManagedDestroyMethod(methodIdentifier)) {
         beanDefinition.registerExternallyManagedDestroyMethod(methodIdentifier);
         checkedDestroyMethods.add(element);
         if (logger.isDebugEnabled()) {
            logger.debug("Registered destroy method on class [" + this.targetClass.getName() + "]: " + element);
         }
      }
   }
   this.checkedInitMethods = checkedInitMethods;
   this.checkedDestroyMethods = checkedDestroyMethods;
}
```

预解析@PostConstruct注解以及@PreDestroy注解完毕



### 属性注入

参考todo

### 初始化Bean

继续分析initializeBean方法

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
   if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
         invokeAwareMethods(beanName, bean);
         return null;
      }, getAccessControlContext());
   }
   else {
      // 如果 bean 实现了 BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口，回调
      if(logger.isInfoEnabled()){
         logger.info("调用BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口方法");
      }
      invokeAwareMethods(beanName, bean);
   }

   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      // 执行全部的后置处理器BeanPostProcessor 的 postProcessBeforeInitialization 回调
      // ApplicationContextAwareProcessor 来处理
      //  EnvironmentAware
      //  EmbeddedValueResolverAware
      //  ResourceLoaderAware
      //  ApplicationEventPublisherAware
      //  MessageSourceAware
      //  ApplicationContextAware
      //  ImportAwareBeanPostProcessor来处理ImportAware
      //  CommonAnnotationBeanPostProcessor来处理@PostConstruct
      if(logger.isInfoEnabled()){
         logger.info("执行beanPostProcessorsBeforeInitialization方法,其中处理@PostConstrct标注的方法,和一些实现XXAware接口的类");
      }
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      // 处理 bean 中定义的 init-method，
      // 或者如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
      if(logger.isInfoEnabled()){
         logger.info("执行init-method,如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法");
      }
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   catch (Throwable ex) {
      throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
   }
   if (mbd == null || !mbd.isSynthetic()) {
      // 执行所有的后置处理器BeanPostProcessor 的 postProcessAfterInitialization 回调
      // Spring AOP 会在 IOC 容器创建 bean 实例的最后对 bean 进行处理。其实就是在这一步进行代理增强。
      if(logger.isInfoEnabled()){
         logger.info("执行beanPostProcessorsAfterInitialization方法\n" +
               "AOP在这里利用CGLIB或者JDK的动态代理将要增强后的类放到单例池中");
      }
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }

   return wrappedBean;
}
```

1. 调用BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口方法
2. 调用BeanPostProcessor的postProcessBeforeInitialization方法，根据BeanPostProcessor的顺序todo，先调ApplicationContextAwareProcessor来处理EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware等接口方法；后调CommonAnnotationBeanPostProcessor来处理@PostConstruct
3. 调用InitializingBean的afterPropertiesSet方法
4. 反射调用@Bean(autowire = Autowire.BY_NAME,initMethod = "init")中initMethod指定的方法
5. 调用BeanPostProcessor的postProcessAfterInitialization方法，根据BeanPostProcessor的顺序todo，先调AbstractAutoProxyCreator的postProcessAfterInitialization来生成需要动态代理的类；后调ApplicationListenerDetector的postProcessAfterInitialization方法如果创建的bean是ApplicationListener的实现类且是单例的，则加入到容器的applicationListeners中。



## FactoryBean返回的Bean实例特殊处理

```java
protected Object getObjectForBeanInstance(
      Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

   // Don't let calling code try to dereference the factory if the bean isn't a factory.
   // 如果name以“&”为前缀，但是beanInstance不是FactoryBean，则抛异常
   if (BeanFactoryUtils.isFactoryDereference(name)) {
      if (beanInstance instanceof NullBean) {
         return beanInstance;
      }
      if (!(beanInstance instanceof FactoryBean)) {
         throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
      }
   }

   // Now we have the bean instance, which may be a normal bean or a FactoryBean.
   // If it's a FactoryBean, we use it to create a bean instance, unless the
   // caller actually wants a reference to the factory.
   // 如果beanInstance不是FactoryBean（也就是普通bean），则直接返回beanInstance
   // 如果beanInstance是FactoryBean，并且name以“&”为前缀，则直接返回beanInstance（以“&”为前缀代表想获取的是FactoryBean本身）
   if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
      return beanInstance;
   }

   Object object = null;
   // 走到这边，代表beanInstance是FactoryBean，但name不带有“&”前缀，表示想要获取的是FactoryBean创建的对象实例
   if (mbd == null) {
      // 如果mbd为空，则尝试从factoryBeanObjectCache缓存中获取该FactoryBean创建的对象实例
      object = getCachedObjectForFactoryBean(beanName);
   }
   if (object == null) {
      // Return bean instance from factory.
      // 只有beanInstance是FactoryBean才能走到这边，因此直接强转
      FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
      // Caches object obtained from FactoryBean if it is a singleton.
      if (mbd == null && containsBeanDefinition(beanName)) {
         // 6.mbd为空，但是该bean的BeanDefinition在缓存中存在，则获取该bean的MergedBeanDefinition
         mbd = getMergedLocalBeanDefinition(beanName);
      }
      // mbd是否是合成的（这个字段比较复杂，mbd正常情况都不是合成的，也就是false）
      boolean synthetic = (mbd != null && mbd.isSynthetic());
      // 从FactoryBean获取对象实例
      object = getObjectFromFactoryBean(factory, beanName, !synthetic);
   }
   return object;
}
```

-  如果beanInstance不是FactoryBean（也就是普通bean），则直接返回beanInstance
- 如果beanInstance是FactoryBean，并且name以“&”为前缀，则直接返回beanInstance（以“&”为前缀代表想获取的是FactoryBean本身）
- 如果beanInstance是FactoryBean，但name不带有“&”前缀，则调用FactoryBean的getObject方法获取对象实例，并返回该实例

## 总结

1. 根据beanName尝试从singletonObjects获取Bean，和解决循环依赖有关
2. 非单例作用域的bean，存在循环引用则抛异常
3. 根据beanName先从父容器中查找
4. 如果要创建的bean含有@DependsOn注解，先加载@DependsOn指定的Bean
5. 调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法，在AnnotationAwareAspectJAutoProxyCreator的postProcessBeforeInstantiation方法中获取所有的AOP的advisor以及事务的advisor
6. 根据bean的作用域采用不同的策略来实例化Bean
7. 调用MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition方法，其中预解析@Autowire、@Value与预解析@PostConstruct、@PreDestroy注解
8. 将实例化的bean提前暴露出去，暴露到singletonFactories中，用于解决循环依赖
9. 进行属性注入处理todo
10. 调用BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口方法
11. 调用BeanPostProcessor的postProcessBeforeInitialization方法，根据BeanPostProcessor的顺序todo，先调ApplicationContextAwareProcessor来处理EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware等接口方法；后调CommonAnnotationBeanPostProcessor来处理@PostConstruct
12. 调用InitializingBean的afterPropertiesSet方法
13. 反射调用@Bean(autowire = Autowire.BY_NAME,initMethod = "init")中initMethod指定的方法
14. 调用BeanPostProcessor的postProcessAfterInitialization方法，根据BeanPostProcessor的顺序todo，先调AbstractAutoProxyCreator的postProcessAfterInitialization来生成需要动态代理的类；后调ApplicationListenerDetector的postProcessAfterInitialization方法如果创建的bean是ApplicationListener的实现类且是单例的，则加入到容器的applicationListeners中。
15. FactoryBean返回的Bean实例特殊处理，返回对应的初始化完成的Bean

## 参考

https://blog.51cto.com/panyujie/5551094