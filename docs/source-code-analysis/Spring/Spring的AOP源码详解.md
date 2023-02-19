[TOC]



# Spring的AOP源码详解

本文主要介绍Spring AOP原理解析的切面实现过程（将切面类的所有切面方法根据使用的注解生成对应的增强Advice，并将Advice连同切入点匹配器和切面类等信息一并封装到Advisor，为后续交给代理增强实现做准备的过程）

## 简单使用

```java
@Configuration
@EnableAspectJAutoProxy
public class SpringConfig {
}
```

```java
@Aspect
@Component
@Slf4j
public class LogAspect {

   @Pointcut("@annotation(com.lijunhua.advice.log.Logger)")
   public void pc() {
   }

   @Around("pc()")
   public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
      //1、调用开始...
      String param = JSON.toJSONString(joinPoint.getArgs());
      //开始时间
      long start = System.currentTimeMillis();
      Object result = null;//返回结果 当异常时修改返回结果
      //执行
      try {
         result = joinPoint.proceed();
      } catch (Exception e) {
         log.error("soa method error,param = {}", param);
         throw e;
      }
      //结束时间
      long end = System.currentTimeMillis();
      log.info("soa method end,start time:{}; end time:{}; Run Time:{}(ms), param={}, result = {}", start, end, (end - start), param, JSON.toJSONString(result, SerializerFeature.DisableCircularReferenceDetect));
      return result;
   }
}
```

本文只分析AOP的注解实现原理

## @EnableAspectJAutoProxy解析

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

   /**
    * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
    * to standard Java interface-based proxies. The default is {@code false}.
    */
   boolean proxyTargetClass() default false;

   /**
    * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
    * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
    * Off by default, i.e. no guarantees that {@code AopContext} access will work.
    * @since 4.3.1
    */
   boolean exposeProxy() default false;

}
```

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

   private final Log logger = LogFactory.getLog(getClass());
   /**
    * Register, escalate, and configure the AspectJ auto proxy creator based on the value
    * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
    * {@code @Configuration} class.
    */
   @Override
   public void registerBeanDefinitions(
         AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

      // 就是在往传入的BeanDefinitionRegistry registry中注册BeanDefinition。
      // 注册了BeanDefinition之后，Spring就会去实例化这个Bean，从而达到AspectJAutoProxy作用
      // 添加到Spring上下文中的AnnotationAwareAspectjAutoProxyCreator对象
      //  AnnotationAwareAspectjAutoProxyCreator对象其实就是这个类型InstantiationAwareBeanPostProcessor
      // 在createbean时调用resolveBeforeInstantiation(beanName, mbdToUse)，
      //  会首先调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法，可以在此处返回代理的对象
      if(logger.isInfoEnabled()){
         logger.info("加入AOP处理的类的bd到bdMap中，该类名为：AnnotationAwareAspectJAutoProxyCreator");
      }
      AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

      AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
      if (enableAspectJAutoProxy != null) {
         if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
         }
         if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
         }
      }
   }

}
```

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
   return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
}
```

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
      BeanDefinitionRegistry registry, @Nullable Object source) {

   return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

```java
private static BeanDefinition registerOrEscalateApcAsRequired(
      Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   // 如果注册表中已经存在beanName=org.springframework.aop.config.internalAutoProxyCreator的bean，
   // 则按优先级进行选择。
   // beanName=org.springframework.aop.config.internalAutoProxyCreator，可能存在的beanClass有三种，
   // 按优先级排序如下：
   // InfrastructureAdvisorAutoProxyCreator、
   // AspectJAwareAdvisorAutoProxyCreator、
   // AnnotationAwareAspectJAutoProxyCreator
   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
      // 拿到已经存在的bean定义
      BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      // 如果已经存在的bean的className与当前要注册的bean的className不相同，则按优先级进行选择
      if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
         // 拿到已经存在的bean的优先级
         int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
         // 拿到当前要注册的bean的优先级
         int requiredPriority = findPriorityForClass(cls);
         if (currentPriority < requiredPriority) {
            // 如果当前要注册的bean的优先级大于已经存在的bean的优先级，则将bean的className替换为当前要注册的bean的className，
            apcDefinition.setBeanClassName(cls.getName());
         }
         // 如果小于，则不做处理
      }
      // 如果已经存在的bean的className与当前要注册的bean的className相同，则无需进行任何处理
      return null;
   }

   // 如果注册表中还不存在，则新建一个Bean定义，并添加到注册表中
   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
   beanDefinition.setSource(source);
   // 设置了order为最高优先级
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   // 注册BeanDefinition，beanName为org.springframework.aop.config.internalAutoProxyCreator
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
   return beanDefinition;
}
```



### 小结

1. 利用@Import导入AspectJAutoProxyRegistrar类，该类实现ImportBeanDefinitionRegistrar接口
2. 在解析AspectJAutoProxyRegistrar的registerBeanDefinitions方法是注册BeanDefinition，这里面注册了AnnotationAwareAspectJAutoProxyCreator的BeanDefinition，后续实例化以及初始化该类，在处理AOP的增强逻辑。

## @Aspect的解析

即形成Advisor的过程

### 解析的时机

初次从spring容器中获取第一个bean时，调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法时，调用createBean方法--->resolveBeforeInstantiation方法--->调用容器中所有的InstantiationAwareBeanPostProcessor的实现类的postProcessBeforeInstantiation方法

### 真正解析步骤

最终调到AbstractAutoProxyCreator的postProcessBeforeInstantiation方法

```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
   Object cacheKey = getCacheKey(beanClass, beanName);

   if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
      if (this.advisedBeans.containsKey(cacheKey)) {
         return null;
      }
      // bean的类是aop基础设施类 || 使用@Aspect注解
      // 在shouldSkip中初始化了所有的AOP的advisor以及事务的advisor
      if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
         this.advisedBeans.put(cacheKey, Boolean.FALSE);
         return null;
      }
   }

   // Create proxy here if we have a custom TargetSource.
   // Suppresses unnecessary default instantiation of the target bean:
   // The TargetSource will handle target instances in a custom fashion.
   TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
   if (targetSource != null) {
      if (StringUtils.hasLength(beanName)) {
         this.targetSourcedBeans.add(beanName);
      }
      Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
      Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   return null;
}
```

分析shouldSkip方法

```java
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
   // TODO: Consider optimization by caching the list of the aspect names
   // 找到容器中所有的Advisor，含AOP和Spring事务
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   for (Advisor advisor : candidateAdvisors) {
      // 名称相同
      if (advisor instanceof AspectJPointcutAdvisor &&
            ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
         return true;
      }
   }
   return super.shouldSkip(beanClass, beanName);
}
```

最终调到AnnotationAwareAspectJAutoProxyCreator的findCandidateAdvisors方法

```java
protected List<Advisor> findCandidateAdvisors() {
   // Add all the Spring advisors found according to superclass rules.
   // 添加根据父类规则找到的所有advisor。
   List<Advisor> advisors = super.findCandidateAdvisors();
   // Build Advisors for all AspectJ aspects in the bean factory.
   // 为bean工厂中的所有AspectJ方面构建advisor
   if (this.aspectJAdvisorsBuilder != null) {
      advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
   }
   return advisors;
}
```

1. 查找容器中所有的Advisor接口的实现类，其中事务的Advisor就是在这里实例化以及初始化的
2. 创建AOP需要的Advisor

本文只分析AOP的注解实现，调父类的findCandidateAdvisors见todo Spring的事务源码解析

继续分析buildAspectJAdvisors方法

```java
public List<Advisor> buildAspectJAdvisors() {
   List<String> aspectNames = this.aspectBeanNames;

   // 如果aspectNames为空，则进行解析
   if (aspectNames == null) {
      synchronized (this) {
         aspectNames = this.aspectBeanNames;
         if (aspectNames == null) {
            List<Advisor> advisors = new ArrayList<>();
            aspectNames = new ArrayList<>();
            // 获取所有的beanName
            String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                  this.beanFactory, Object.class, true, false);
            // 循环遍历所有的beanName，找出对应的增强方法
            for (String beanName : beanNames) {
               // 不合法的beanName则跳过，默认返回true，子类可以覆盖实现，AnnotationAwareAspectJAutoProxyCreator
               // 实现了自己的逻辑，支持使用includePatterns进行筛选
               if (!isEligibleBean(beanName)) {
                  continue;
               }
               // We must be careful not to instantiate beans eagerly as in this case they
               // would be cached by the Spring container but would not have been weaved.
               // 获取beanName对应的bean的类型
               Class<?> beanType = this.beanFactory.getType(beanName);
               if (beanType == null) {
                  continue;
               }
               // 如果beanType存在Aspect注解则进行处理
               if (this.advisorFactory.isAspect(beanType)) {
                  if (logger.isInfoEnabled()) {
                     logger.info("扫描到自定义AOP切面, 切面名: "+beanName);
                  }
                  // 将存在Aspect注解的beanName添加到aspectNames列表
                  aspectNames.add(beanName);
                  // 新建切面元数据
                  AspectMetadata amd = new AspectMetadata(beanType, beanName);
                  // 获取per-clause的类型是SINGLETON
                  if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                     // 使用BeanFactory和beanName创建一个BeanFactoryAspectInstanceFactory，主要用来创建切面对象实例
                     MetadataAwareAspectInstanceFactory factory =
                           new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                     // 解析标记AspectJ注解中的增强方法
                     List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                     // 放到缓存中
                     if (this.beanFactory.isSingleton(beanName)) {
                        // 如果beanName是单例则直接将解析的增强方法放到缓存
                        this.advisorsCache.put(beanName, classAdvisors);
                     }
                     else {
                        // 如果不是单例，则将factory放到缓存，之后可以通过factory来解析增强方法
                        this.aspectFactoryCache.put(beanName, factory);
                     }
                     // 将解析的增强器添加到advisors
                     advisors.addAll(classAdvisors);
                  }
                  else {
                     // Per target or per this.
                     // 如果per-clause的类型不是SINGLETON
                     if (this.beanFactory.isSingleton(beanName)) {
                        // 名称为beanName的Bean是单例，但切面实例化模型不是单例，则抛异常
                        throw new IllegalArgumentException("Bean with name '" + beanName +
                              "' is a singleton, but aspect instantiation model is not singleton");
                     }
                     MetadataAwareAspectInstanceFactory factory =
                           new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                     // 将factory放到缓存，之后可以通过factory来解析增强方法
                     this.aspectFactoryCache.put(beanName, factory);
                     // 解析标记AspectJ注解中的增强方法，并添加到advisors中
                     advisors.addAll(this.advisorFactory.getAdvisors(factory));
                  }
               }
            }
            // 将解析出来的切面beanName放到缓存aspectBeanNames
            this.aspectBeanNames = aspectNames;
            // 最后返回解析出来的增强器
            return advisors;
         }
      }
   }

   // 如果aspectNames不为null，则代表已经解析过了，则无需再次解析
   // 如果aspectNames是空列表，则返回一个空列表。空列表也是解析过的，只要不是null都是解析过的。
   if (aspectNames.isEmpty()) {
      return Collections.emptyList();
   }
   // aspectNames不是空列表，则遍历处理
   List<Advisor> advisors = new ArrayList<>();
   for (String aspectName : aspectNames) {
      // 根据aspectName从缓存中获取增强器
      List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
      if (cachedAdvisors != null) {
         // 根据上面的解析，可以知道advisorsCache存的是已经解析好的增强器，直接添加到结果即可
         advisors.addAll(cachedAdvisors);
      }
      else {
         // 如果不存在于advisorsCache缓存，则代表存在于aspectFactoryCache中，
         // 从aspectFactoryCache中拿到缓存的factory，然后解析出增强器，添加到结果中
         MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
         advisors.addAll(this.advisorFactory.getAdvisors(factory));
      }
   }
   // 返回增强器
   return advisors;
}
```

1. 解析标记AspectJ注解中的增强方法（Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class）生成AspectJExpressionPointcut
2. 根据增强方法的注解类型生成对应的AbstractAspectJAdvice的子类AspectJAroundAdvice等等
3. 返回所有的AOP的Advisor，其中Advisor中含有Advice增强方法

分析this.advisorFactory.getAdvisors方法

```java
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
   // 前面我们将beanClass和beanName封装成了aspectInstanceFactory的AspectMetadata属性，
   // 这边可以通过AspectMetadata属性重新获取到当前处理的切面类
   Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   // 获取当前处理的切面类的名字
   String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();

   validate(aspectClass);

   // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
   // so that it will only instantiate once.
   // 使用装饰器包装MetadataAwareAspectInstanceFactory，以便它只实例化一次。
   MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
         new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

   List<Advisor> advisors = new ArrayList<>();
   // 获取切面类中的方法（也就是我们用来进行逻辑增强的方法，被@Around、@After等注解修饰的方法，使用@Pointcut的方法不处理）
   for (Method method : getAdvisorMethods(aspectClass)) {
      // 处理method，获取增强器
      Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
      if (advisor != null) {
         // 如果增强器不为空，则添加到advisors
         advisors.add(advisor);
      }
   }

   // If it's a per target aspect, emit the dummy instantiating aspect.
   if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
      // 如果寻找的增强器不为空而且又配置了增强延迟初始化，那么需要在首位加入同步实例化增强器（用以保证增强使用之前的实例化）
      Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
      advisors.add(0, instantiationAdvisor);
   }

   // Find introduction fields.
   // 获取DeclareParents注解
   for (Field field : aspectClass.getDeclaredFields()) {
      Advisor advisor = getDeclareParentsAdvisor(field);
      if (advisor != null) {
         advisors.add(advisor);
      }
   }

   return advisors;
}
```

调用getAdvisor获取方法上面的增强器

```java
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
      int declarationOrderInAspect, String aspectName) {

   validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

   // AspectJ切点信息的获取（例如：表达式），就是指定注解的表达式信息的获取，
   // 如：@Around("@annotation(com.ljh.advice.log.Logger)")
   AspectJExpressionPointcut expressionPointcut = getPointcut(
         candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
   // 如果expressionPointcut为null，则直接返回null
   if (expressionPointcut == null) {
      return null;
   }

   // 根据切点信息生成增强器
   return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
         this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

1. 解析AspectJAnnotation注解信息
2. 根据AspectJAnnotation注解信息生成Advisor

```java
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
   // 查找并返回给定方法的第一个AspectJ注解（@Before, @Around, @After, @AfterReturning, @AfterThrowing, @Pointcut）
   // 因为我们之前把@Pointcut注解的方法跳过了，所以这边必然不会获取到@Pointcut注解
   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   // 如果方法没有使用AspectJ的注解，则返回null
   if (aspectJAnnotation == null) {
      return null;
   }

   // 使用AspectJExpressionPointcut实例封装获取的信息
   AspectJExpressionPointcut ajexp =
         new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
   // 提取得到的注解中的表达式，
   // 例如：@Around("@annotation(com.ljh.advice.log.Logger)")，
   // 得到：@annotation(com.ljh.advice.log.Logger)
   ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
   if (this.beanFactory != null) {
      ajexp.setBeanFactory(this.beanFactory);
   }
   return ajexp;
}
```

1. 拿到方法上面的AspectJAnnotation，包含：Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class
2. 使用AspectJExpressionPointcut实例封装获取的注解信息

```java
protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
   // 设置要查找的注解类
   for (Class<?> clazz : ASPECTJ_ANNOTATION_CLASSES) {
      // 查找方法上是否存在当前遍历的注解，如果有则返回
      AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) clazz);
      if (foundAnnotation != null) {
         return foundAnnotation;
      }
   }
   return null;
}
```

获取方法上的AspectJAnnotation注解信息

继续分析根据AspectJAnnotation注解信息生成Advisor

```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
      Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

   // 简单的将信息封装在类的实例中
   this.declaredPointcut = declaredPointcut;
   this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
   this.methodName = aspectJAdviceMethod.getName();
   this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
   // aspectJAdviceMethod保存的是我们用来进行逻辑增强的方法（@Around、@After等修饰的方法）
   this.aspectJAdviceMethod = aspectJAdviceMethod;
   this.aspectJAdvisorFactory = aspectJAdvisorFactory;
   this.aspectInstanceFactory = aspectInstanceFactory;
   this.declarationOrder = declarationOrder;
   this.aspectName = aspectName;

   // 是否需要延迟实例化
   if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
      // Static part of the pointcut is a lazy type.
      Pointcut preInstantiationPointcut = Pointcuts.union(
            aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

      // Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
      // If it's not a dynamic pointcut, it may be optimized out
      // by the Spring AOP infrastructure after the first evaluation.
      this.pointcut = new PerTargetInstantiationModelPointcut(
            this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
      this.lazy = true;
   }
   else {
      // A singleton aspect.
      this.pointcut = this.declaredPointcut;
      this.lazy = false;
      // 实例化增强器：根据注解中的信息初始化对应的增强器
      this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
   }
}
```

 实例化增强器

```java
private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
   // 创建 advice
   Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
         this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
   return (advice != null ? advice : EMPTY_ADVICE);
}
```

```java
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
      MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

   // 获取切面类
   Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
   // 校验切面类（重复校验第3次...）
   validate(candidateAspectClass);

   // 查找并返回方法的第一个AspectJ注解
   AspectJAnnotation<?> aspectJAnnotation =
         AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
   if (aspectJAnnotation == null) {
      return null;
   }

   // If we get here, we know we have an AspectJ method.
   // Check that it's an AspectJ-annotated class
   // 如果我们到这里，我们知道我们有一个AspectJ方法。检查切面类是否使用了AspectJ注解
   if (!isAspect(candidateAspectClass)) {
      throw new AopConfigException("Advice must be declared inside an aspect type: " +
            "Offending method '" + candidateAdviceMethod + "' in class [" +
            candidateAspectClass.getName() + "]");
   }

   if (logger.isDebugEnabled()) {
      logger.debug("Found AspectJ method: " + candidateAdviceMethod);
   }

   AbstractAspectJAdvice springAdvice;

   // 根据方法使用的aspectJ注解创建对应的增强器，例如最常见的@Around注解会创建AspectJAroundAdvice
   switch (aspectJAnnotation.getAnnotationType()) {
      case AtPointcut:
         if (logger.isDebugEnabled()) {
            logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
         }
         return null;
      case AtAround:
         springAdvice = new AspectJAroundAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtBefore:
         springAdvice = new AspectJMethodBeforeAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfter:
         springAdvice = new AspectJAfterAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         break;
      case AtAfterReturning:
         springAdvice = new AspectJAfterReturningAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterReturningAnnotation.returning())) {
            springAdvice.setReturningName(afterReturningAnnotation.returning());
         }
         break;
      case AtAfterThrowing:
         springAdvice = new AspectJAfterThrowingAdvice(
               candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
         AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
         if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
            springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
         }
         break;
      default:
         throw new UnsupportedOperationException(
               "Unsupported advice type on method: " + candidateAdviceMethod);
   }

   // Now to configure the advice...
   // 配置增强器
   // 切面类的name，其实就是beanName
   springAdvice.setAspectName(aspectName);
   springAdvice.setDeclarationOrder(declarationOrder);
   // 获取增强方法的参数
   String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
   if (argNames != null) {
      // 如果参数不为空，则赋值给springAdvice
      springAdvice.setArgumentNamesFromStringArray(argNames);
   }
   springAdvice.calculateArgumentBindings();

   // 最后，返回增强器
   return springAdvice;
}
```

生成该注解对应的增强器

### 小结

1. 初次从spring容器中获取第一个bean时，调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法时，调用createBean方法--->resolveBeforeInstantiation方法--->调用容器中所有的InstantiationAwareBeanPostProcessor的实现类的postProcessBeforeInstantiation方法
2. 拿到Spring容器中所有的bean，如果bean存在Aspect注解则进行处理
3. 解析标记AspectJ注解中的增强方法（Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class）生成AspectJExpressionPointcut
4. 根据增强方法的注解类型生成对应的AbstractAspectJAdvice的子类
5. 返回所有的AOP的Advisor，其中Advisor中含有Advice增强方法

## 使用Advisor进行动态代理增强

### 增强的时机

在bean初始化的最后一步，调Spring容器中所有的BeanPostProcessor的postProcessAfterInitialization方法时

### 真正的处理逻辑

调AbstractAutoProxyCreator#postProcessAfterInitialization方法

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (this.earlyProxyReferences.remove(cacheKey) != bean) {
         // 返回代理类（如果需要的话）
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   // 判断当前bean是否在targetSourcedBeans缓存中存在（已经处理过），如果存在，则直接返回当前bean
   if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   // 在advisedBeans缓存中存在，并且value为false，则代表无需处理
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   // bean的类是aop基础设施类 || 使用@Aspect注解
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // Create proxy if we have advice.
   // 返回匹配当前 bean 的所有的 advisor、advice、interceptor
   // getAdvicesAndAdvisorsForBean这个方法将得到所有的可用于拦截当前 bean 的 advisor、advice、interceptor。
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   // 如果存在增强器则创建代理
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      // 创建代理对象：这边SingletonTargetSource的target属性存放的就是我们原来的bean实例
      // （也就是被代理对象），
      // 用于最后增加逻辑执行完毕后，通过反射执行我们真正的方法时使用（method.invoke(bean, args)）
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      // 创建完代理后，将cacheKey -> 代理类的class放到缓存
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   // 标记为无需处理
   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

1. 其中getAdvicesAndAdvisorsForBean是得到所有的可用于拦截当前 bean 的 advisor
2. 利用选出来的advisor使用动态代理创建Bean

分析getAdvicesAndAdvisorsForBean方法

```java
protected Object[] getAdvicesAndAdvisorsForBean(
      Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

   // 找到符合条件的Advisor
   List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
   if (advisors.isEmpty()) {
      // 如果没有符合条件的Advisor，则返回null
      return DO_NOT_PROXY;
   }
   return advisors.toArray();
}
```

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
   // 查找所有的候选Advisor
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   // 从所有候选的Advisor中找出符合条件的
   List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
   // 如果存在AOP的Advisor，在eligibleAdvisors中增加ExposeInvocationInterceptor.ADVISOR
   extendAdvisors(eligibleAdvisors);
   if (!eligibleAdvisors.isEmpty()) {
      // 对符合条件的Advisor进行排序
      eligibleAdvisors = sortAdvisors(eligibleAdvisors);
   }
   return eligibleAdvisors;
}
```

1. 获取Spring容器中所有的Advisor，包含Spring的AOP的Advisor以及事务处理的Advisor。findCandidateAdvisors见上文
2. 从所有的Advisor中找出符合条件的Advisor
3. 如果存在AOP的Advisor，在eligibleAdvisors中第一个增加ExposeInvocationInterceptor.ADVISOR，其中ExposeInvocationInterceptor.ADVISOR用来记录调用的上下文

继续分析findAdvisorsThatCanApply

```java
protected List<Advisor> findAdvisorsThatCanApply(
      List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

   ProxyCreationContext.setCurrentProxiedBeanName(beanName);
   try {
      // 从candidateAdvisors中找出符合条件的Advisor
      return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
   }
   finally {
      ProxyCreationContext.setCurrentProxiedBeanName(null);
   }
}
```



```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
   if (candidateAdvisors.isEmpty()) {
      return candidateAdvisors;
   }
   List<Advisor> eligibleAdvisors = new ArrayList<>();
   for (Advisor candidate : candidateAdvisors) {
      if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
         eligibleAdvisors.add(candidate);
      }
   }
   boolean hasIntroductions = !eligibleAdvisors.isEmpty();
   // 遍历所有的candidateAdvisors
   for (Advisor candidate : candidateAdvisors) {
      // 引介增强已经处理，直接跳过
      if (candidate instanceof IntroductionAdvisor) {
         // already processed
         continue;
      }
      // 正常增强处理，判断当前bean是否可以应用于当前遍历的增强器
      if (canApply(candidate, clazz, hasIntroductions)) {
         eligibleAdvisors.add(candidate);
      }
   }
   return eligibleAdvisors;
}
```

继续分析canApply，判断当前bean是否可以应用于当前遍历的增强器

```java
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
   Assert.notNull(pc, "Pointcut must not be null");
   // 比较切点是否匹配目标类
   if (!pc.getClassFilter().matches(targetClass)) {
      return false;
   }

   MethodMatcher methodMatcher = pc.getMethodMatcher();
   if (methodMatcher == MethodMatcher.TRUE) {
      // No need to iterate the methods if we're matching any method anyway...
      return true;
   }

   IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
   if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
      introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
   }

   Set<Class<?>> classes = new LinkedHashSet<>();
   if (!Proxy.isProxyClass(targetClass)) {
      classes.add(ClassUtils.getUserClass(targetClass));
   }
   classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

   for (Class<?> clazz : classes) {
      Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
      for (Method method : methods) {
         // 比较切点是否匹配目标方法
         if (introductionAwareMethodMatcher != null ?
               introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
               methodMatcher.matches(method, targetClass)) {
            return true;
         }
      }
   }

   return false;
}
```

1. 比较切点是否匹配目标类
2. 比较切点是否匹配目标方法

先分析pc.getClassFilter().matches比较切点是否匹配目标类

```java
public boolean matches(Class<?> targetClass) {
   // 获取PointcutExpression，没有则新建
   PointcutExpression pointcutExpression = obtainPointcutExpression();
   try {
      try {
         // 测试class匹配
         return pointcutExpression.couldMatchJoinPointsInType(targetClass);
      }
      catch (ReflectionWorldException ex) {
         logger.debug("PointcutExpression matching rejected target class - trying fallback expression", ex);
         // Actually this is still a "maybe" - treat the pointcut as dynamic if we don't know enough yet
         PointcutExpression fallbackExpression = getFallbackPointcutExpression(targetClass);
         if (fallbackExpression != null) {
            return fallbackExpression.couldMatchJoinPointsInType(targetClass);
         }
      }
   }
   catch (Throwable ex) {
      logger.debug("PointcutExpression matching rejected target class", ex);
   }
   return false;
}
```

1. 获取PointcutExpression，没有则新建
2. 使用aspectJ的PointcutExpression来判断类是否匹配，这是AspectJ里面的逻辑，不进行分析

分析获取PointcutExpression的逻辑

```java
private PointcutExpression obtainPointcutExpression() {
   if (getExpression() == null) {
      throw new IllegalStateException("Must set property 'expression' before attempting to match");
   }
   if (this.pointcutExpression == null) {
      // 获取PointcutExpression，没有则新建
      this.pointcutClassLoader = determinePointcutClassLoader();
      this.pointcutExpression = buildPointcutExpression(this.pointcutClassLoader);
   }
   return this.pointcutExpression;
}
```

```java
private PointcutExpression buildPointcutExpression(@Nullable ClassLoader classLoader) {
   // 获取切点解析器
   PointcutParser parser = initializePointcutParser(classLoader);
   PointcutParameter[] pointcutParameters = new PointcutParameter[this.pointcutParameterNames.length];
   for (int i = 0; i < pointcutParameters.length; i++) {
      pointcutParameters[i] = parser.createPointcutParameter(
            this.pointcutParameterNames[i], this.pointcutParameterTypes[i]);
   }
   return parser.parsePointcutExpression(replaceBooleanOperators(resolveExpression()),
         this.pointcutDeclarationScope, pointcutParameters);
}
```

1. 获取切点解析器
2. 调用parser.parsePointcutExpression得到PointcutExpression,这是AspectJ里面的逻辑，不进行分析

分析获取切点解析器的逻辑

```java
private PointcutParser initializePointcutParser(@Nullable ClassLoader classLoader) {
   // 获取一个利用指定classloader、支持指定的原语集合的切点解析器。一共有10个
   PointcutParser parser = PointcutParser
         .getPointcutParserSupportingSpecifiedPrimitivesAndUsingSpecifiedClassLoaderForResolution(
               SUPPORTED_PRIMITIVES, classLoader);
   // bean：当调用的方法是指定的bean的方法时生效。(Spring AOP自己扩展支持的)。Spring一共支持11个
   parser.registerPointcutDesignatorHandler(new BeanPointcutDesignatorHandler());
   return parser;
}
```

其中Sping的AspectJExpressionPointcut是

```java
public class AspectJExpressionPointcut extends AbstractExpressionPointcut
      implements ClassFilter, IntroductionAwareMethodMatcher, BeanFactoryAware {

   private static final Set<PointcutPrimitive> SUPPORTED_PRIMITIVES = new HashSet<>();

   // Spring支持的AspectJ的切点语言表达式一共有10个
   static {
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.EXECUTION);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.ARGS);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.REFERENCE);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.THIS);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.TARGET);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.WITHIN);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_ANNOTATION);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_WITHIN);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_ARGS);
      SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_TARGET);
   }
  ......
}
```

所以Spring的AOP支持的表达式类型有11种，10种AspectJ原生的，1种Spring提供的。



继续分析比较切点是否匹配目标方法的逻辑

```java
public boolean matches(Method method, @Nullable Class<?> targetClass, boolean hasIntroductions) {
   // 获取PointcutExpression，没有则新建
   obtainPointcutExpression();
   // 比较切点是否匹配目标方法
   ShadowMatch shadowMatch = getTargetShadowMatch(method, targetClass);

   // Special handling for this, target, @this, @target, @annotation
   // in Spring - we can optimize since we know we have exactly this class,
   // and there will never be matching subclass at runtime.
   // 切点匹配目标方法
   if (shadowMatch.alwaysMatches()) {
      return true;
   }
   // 切点不匹配目标方法
   else if (shadowMatch.neverMatches()) {
      return false;
   }
   else {
      // the maybe case
      if (hasIntroductions) {
         return true;
      }
      // A match test returned maybe - if there are any subtype sensitive variables
      // involved in the test (this, target, at_this, at_target, at_annotation) then
      // we say this is not a match as in Spring there will never be a different
      // runtime subtype.
      RuntimeTestWalker walker = getRuntimeTestWalker(shadowMatch);
      return (!walker.testsSubtypeSensitiveVars() ||
            (targetClass != null && walker.testTargetInstanceOfResidue(targetClass)));
   }
}
```

1. 获取PointcutExpression
2. 比较切点是否匹配目标方法
3. 根据匹配结果进行返回

分析比较切点是否匹配目标方法

```java
private ShadowMatch getTargetShadowMatch(Method method, @Nullable Class<?> targetClass) {
   Method targetMethod = AopUtils.getMostSpecificMethod(method, targetClass);
   if (targetClass != null && targetMethod.getDeclaringClass().isInterface()) {
      // Try to build the most specific interface possible for inherited methods to be
      // considered for sub-interface matches as well, in particular for proxy classes.
      // Note: AspectJ is only going to take Method.getDeclaringClass() into account.
      Set<Class<?>> ifcs = ClassUtils.getAllInterfacesForClassAsSet(targetClass);
      if (ifcs.size() > 1) {
         try {
            Class<?> compositeInterface = ClassUtils.createCompositeInterface(
                  ClassUtils.toClassArray(ifcs), targetClass.getClassLoader());
            targetMethod = ClassUtils.getMostSpecificMethod(targetMethod, compositeInterface);
         }
         catch (IllegalArgumentException ex) {
            // Implemented interfaces probably expose conflicting method signatures...
            // Proceed with original target method.
         }
      }
   }
   // 比较切点是否匹配目标方法
   return getShadowMatch(targetMethod, method);
}
```

```java
private ShadowMatch getShadowMatch(Method targetMethod, Method originalMethod) {
   // Avoid lock contention for known Methods through concurrent access...
   ShadowMatch shadowMatch = this.shadowMatchCache.get(targetMethod);
   if (shadowMatch == null) {
      synchronized (this.shadowMatchCache) {
         // Not found - now check again with full lock...
         PointcutExpression fallbackExpression = null;
         shadowMatch = this.shadowMatchCache.get(targetMethod);
         if (shadowMatch == null) {
            Method methodToMatch = targetMethod;
            try {
               try {
                  // 比较切点是否匹配目标方法
                  shadowMatch = obtainPointcutExpression().matchesMethodExecution(methodToMatch);
               }
               catch (ReflectionWorldException ex) {
                  // Failed to introspect target method, probably because it has been loaded
                  // in a special ClassLoader. Let's try the declaring ClassLoader instead...
                  try {
                     fallbackExpression = getFallbackPointcutExpression(methodToMatch.getDeclaringClass());
                     if (fallbackExpression != null) {
                        shadowMatch = fallbackExpression.matchesMethodExecution(methodToMatch);
                     }
                  }
                  catch (ReflectionWorldException ex2) {
                     fallbackExpression = null;
                  }
               }
               if (targetMethod != originalMethod && (shadowMatch == null ||
                     (shadowMatch.neverMatches() && Proxy.isProxyClass(targetMethod.getDeclaringClass())))) {
                  // Fall back to the plain original method in case of no resolvable match or a
                  // negative match on a proxy class (which doesn't carry any annotations on its
                  // redeclared methods).
                  methodToMatch = originalMethod;
                  try {
                     shadowMatch = obtainPointcutExpression().matchesMethodExecution(methodToMatch);
                  }
                  catch (ReflectionWorldException ex) {
                     // Could neither introspect the target class nor the proxy class ->
                     // let's try the original method's declaring class before we give up...
                     try {
                        fallbackExpression = getFallbackPointcutExpression(methodToMatch.getDeclaringClass());
                        if (fallbackExpression != null) {
                           shadowMatch = fallbackExpression.matchesMethodExecution(methodToMatch);
                        }
                     }
                     catch (ReflectionWorldException ex2) {
                        fallbackExpression = null;
                     }
                  }
               }
            }
            catch (Throwable ex) {
               // Possibly AspectJ 1.8.10 encountering an invalid signature
               logger.debug("PointcutExpression matching rejected target method", ex);
               fallbackExpression = null;
            }
            if (shadowMatch == null) {
               shadowMatch = new ShadowMatchImpl(org.aspectj.util.FuzzyBoolean.NO, null, null, null);
            }
            else if (shadowMatch.maybeMatches() && fallbackExpression != null) {
               shadowMatch = new DefensiveShadowMatch(shadowMatch,
                     fallbackExpression.matchesMethodExecution(methodToMatch));
            }
            this.shadowMatchCache.put(targetMethod, shadowMatch);
         }
      }
   }
   return shadowMatch;
}
```

最终使用AspectJ判断方法是否匹配，这是AspectJ里面的逻辑，不进行分析

### 小结

1. 在bean初始化的最后一步，调Spring容器中所有的BeanPostProcessor的postProcessAfterInitialization方法时，判断该类是否能增强
2. 获取Spring容器中所有的Advisor，包含Spring的AOP的Advisor以及事务处理的Advisor。
3. 从所有的Advisor中找出符合条件的Advisor
4. 比较Advisor是否匹配目标类
5. 比较Advisor是否匹配目标方法
6. 如果返回的Advisors中存在AOP的Advisor，在eligibleAdvisors中第一个增加ExposeInvocationInterceptor.ADVISOR，其中ExposeInvocationInterceptor.ADVISOR用来记录调用的上下文
7. 利用选出来的advisor使用动态代理创建Bean

## 使用动态代理进行增强

todo































