[TOC]



# Spring事务源码详解

## 简单使用

```java
@Configuration
@EnableTransactionManagement
public class SpringConfig {
}
```

## @EnableTransactionManagement解析

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {

   /**
    * Indicate whether subclass-based (CGLIB) proxies are to be created ({@code true}) as
    * opposed to standard Java interface-based proxies ({@code false}). The default is
    * {@code false}. <strong>Applicable only if {@link #mode()} is set to
    * {@link AdviceMode#PROXY}</strong>.
    * <p>Note that setting this attribute to {@code true} will affect <em>all</em>
    * Spring-managed beans requiring proxying, not just those marked with
    * {@code @Transactional}. For example, other beans marked with Spring's
    * {@code @Async} annotation will be upgraded to subclass proxying at the same
    * time. This approach has no negative impact in practice unless one is explicitly
    * expecting one type of proxy vs another, e.g. in tests.
    */
   boolean proxyTargetClass() default false;

   /**
    * Indicate how transactional advice should be applied.
    * <p><b>The default is {@link AdviceMode#PROXY}.</b>
    * Please note that proxy mode allows for interception of calls through the proxy
    * only. Local calls within the same class cannot get intercepted that way; an
    * {@link Transactional} annotation on such a method within a local call will be
    * ignored since Spring's interceptor does not even kick in for such a runtime
    * scenario. For a more advanced mode of interception, consider switching this to
    * {@link AdviceMode#ASPECTJ}.
    */
   AdviceMode mode() default AdviceMode.PROXY;

   /**
    * Indicate the ordering of the execution of the transaction advisor
    * when multiple advices are applied at a specific joinpoint.
    * <p>The default is {@link Ordered#LOWEST_PRECEDENCE}.
    */
   int order() default Ordered.LOWEST_PRECEDENCE;

}
```

默认代理模式是AdviceMode.PROXY即JDK动态代理



之后在BeanDefinition加载的时候，调用ImportSelector的selectImports来导入beanName

```java
public final String[] selectImports(AnnotationMetadata importingClassMetadata) {
		// 得到 EnableTransactionManagement 类
		Class<?> annType = GenericTypeResolver.resolveTypeArgument(getClass(), AdviceModeImportSelector.class);
		Assert.state(annType != null, "Unresolvable type argument for AdviceModeImportSelector");

		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(importingClassMetadata, annType);
		if (attributes == null) {
			throw new IllegalArgumentException(String.format(
					"@%s is not present on importing class '%s' as expected",
					annType.getSimpleName(), importingClassMetadata.getClassName()));
		}
		// 获得 EnableTransactionManagement.mode 的值默认是AdviceMode.PROXY即JDK动态代理
		AdviceMode adviceMode = attributes.getEnum(getAdviceModeAttributeName());
		String[] imports = selectImports(adviceMode);
		if (imports == null) {
			throw new IllegalArgumentException("Unknown AdviceMode: " + adviceMode);
		}
		return imports;
	}
```

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

   /**
    * Returns {@link ProxyTransactionManagementConfiguration} or
    * {@code AspectJTransactionManagementConfiguration} for {@code PROXY}
    * and {@code ASPECTJ} values of {@link EnableTransactionManagement#mode()},
    * respectively.
    */
   @Override
   protected String[] selectImports(AdviceMode adviceMode) {
      switch (adviceMode) {
         // 默认使用JDK proxy-based advice
         case PROXY:
            return new String[] {AutoProxyRegistrar.class.getName(),
                  ProxyTransactionManagementConfiguration.class.getName()};
         case ASPECTJ:
            return new String[] {
                  TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME};
         default:
            return null;
      }
   }

}
```

返回beanName：AutoProxyRegistrar.class.getName(), ProxyTransactionManagementConfiguration.class.getName()

```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

   @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
      BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
      advisor.setTransactionAttributeSource(transactionAttributeSource());
      advisor.setAdvice(transactionInterceptor());
      if (this.enableTx != null) {
         advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
      }
      return advisor;
   }

   @Bean
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public TransactionAttributeSource transactionAttributeSource() {
      return new AnnotationTransactionAttributeSource();
   }

   @Bean
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public TransactionInterceptor transactionInterceptor() {
      TransactionInterceptor interceptor = new TransactionInterceptor();
      interceptor.setTransactionAttributeSource(transactionAttributeSource());
      if (this.txManager != null) {
         interceptor.setTransactionManager(this.txManager);
      }
      return interceptor;
   }

}
```

在ProxyTransactionManagementConfiguration中利用工厂模式创建BeanFactoryTransactionAttributeSourceAdvisor、TransactionInterceptor、TransactionAttributeSource

### 小结

使用@EnableTransactionManagement之后，最终会实例化以及初始化三个Bean，BeanFactoryTransactionAttributeSourceAdvisor、TransactionInterceptor、TransactionAttributeSource

## 使用Advisor进行动态代理增强

### 增强的时机

增强时机和一致todo见Spring的Aop源码详解

只是事务使用findCandidateAdvisors来得到容器中所有的Advisor

分析findCandidateAdvisors的逻辑

```java
protected List<Advisor> findCandidateAdvisors() {
   Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
   return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

```java
public List<Advisor> findAdvisorBeans() {
   // Determine list of advisor bean names, if not cached already.
   // 确认advisor的beanName列表，优先从缓存中拿
   String[] advisorNames = this.cachedAdvisorBeanNames;
   if (advisorNames == null) {
      // Do not initialize FactoryBeans here: We need to leave all regular beans
      // uninitialized to let the auto-proxy creator apply to them!
      //  如果缓存为空，则获取class类型为Advisor的所有bean名称
      advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
            this.beanFactory, Advisor.class, true, false);
      this.cachedAdvisorBeanNames = advisorNames;
   }
   if (advisorNames.length == 0) {
      return new ArrayList<>();
   }

   // 遍历处理advisorNames
   List<Advisor> advisors = new ArrayList<>();
   for (String name : advisorNames) {
      if (isEligibleBean(name)) {
         // 跳过当前正在创建的advisor
         if (this.beanFactory.isCurrentlyInCreation(name)) {
            if (logger.isDebugEnabled()) {
               logger.debug("Skipping currently created advisor '" + name + "'");
            }
         }
         else {
            try {
               // 通过beanName获取对应的bean对象，并添加到advisors
               advisors.add(this.beanFactory.getBean(name, Advisor.class));
            }
            catch (BeanCreationException ex) {
               Throwable rootCause = ex.getMostSpecificCause();
               if (rootCause instanceof BeanCurrentlyInCreationException) {
                  BeanCreationException bce = (BeanCreationException) rootCause;
                  String bceBeanName = bce.getBeanName();
                  if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                     if (logger.isDebugEnabled()) {
                        logger.debug("Skipping advisor '" + name +
                              "' with dependency on currently created bean: " + ex.getMessage());
                     }
                     // Ignore: indicates a reference back to the bean we're trying to advise.
                     // We want to find advisors other than the currently created bean itself.
                     continue;
                  }
               }
               throw ex;
            }
         }
      }
   }
   // 返回符合条件的advisor列表
   return advisors;
}
```

这里实例化以及初始化了BeanFactoryTransactionAttributeSourceAdvisor

继续分析findAdvisorsThatCanApply从所有候选的Advisor中找出符合条件的，需要遍历初始化成功后类以及类上的方法，判断是否符合条件

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

```java
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
   if (advisor instanceof IntroductionAdvisor) {
      return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
   }
   else if (advisor instanceof PointcutAdvisor) {
      // AOP增强处理，判断当前bean是否可以应用于当前遍历的增强器
      PointcutAdvisor pca = (PointcutAdvisor) advisor;
      return canApply(pca.getPointcut(), targetClass, hasIntroductions);
   }
   else {
      // It doesn't have a pointcut so we assume it applies.
      return true;
   }
}
```

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

之后调用TransactionAttributeSourcePointcut的matches方法来判断该方法是否应该被事务增强

```java
public boolean matches(Method method, @Nullable Class<?> targetClass) {
   if (targetClass != null && TransactionalProxy.class.isAssignableFrom(targetClass)) {
      return false;
   }
   TransactionAttributeSource tas = getTransactionAttributeSource();
   return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
}
```

调用类上或者方法上是否有@Transaction注解

```java
public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
   if (method.getDeclaringClass() == Object.class) {
      return null;
   }

   // First, see if we have a cached value.
   Object cacheKey = getCacheKey(method, targetClass);
   TransactionAttribute cached = this.attributeCache.get(cacheKey);
   if (cached != null) {
      // Value will either be canonical value indicating there is no transaction attribute,
      // or an actual transaction attribute.
      if (cached == NULL_TRANSACTION_ATTRIBUTE) {
         return null;
      }
      else {
         return cached;
      }
   }
   else {
      // We need to work it out.获取事务注解
      TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
      // Put it in the cache.
      if (txAttr == null) {
         this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
      }
      else {
         String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
         if (txAttr instanceof DefaultTransactionAttribute) {
            ((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
         }
         if (logger.isDebugEnabled()) {
            logger.debug("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
         }
         this.attributeCache.put(cacheKey, txAttr);
      }
      return txAttr;
   }
}
```

```java
protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
   // Don't allow no-public methods as required.
   if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
      return null;
   }

   // The method may be on an interface, but we need attributes from the target class.
   // If the target class is null, the method will be unchanged.
   Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

   // First try is the method in the target class.解析方法上的@Transactional注解
   TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
   if (txAttr != null) {
      return txAttr;
   }

   // Second try is the transaction attribute on the target class.解析类上的@Transactional注解
   txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
   if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
      return txAttr;
   }
   if (specificMethod != method) {
      // Fallback is to look at the original method.
      txAttr = findTransactionAttribute(method);
      if (txAttr != null) {
         return txAttr;
      }
      // Last fallback is the class of the original method.
      txAttr = findTransactionAttribute(method.getDeclaringClass());
      if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
         return txAttr;
      }
   }

   return null;
}
```

获取类上或者方法上的@Transaction注解





### 真正的执行逻辑

详细步骤见todo Spring的AOP之动代理源码详解

直接分析invocation.proceed()分析责任链模式来调用增强器Advice，以及最后调用真正的方法。

```java
public Object proceed() throws Throwable {
   // We start with an index of -1 and increment early.
   // 如果所有拦截器执行结束，调用真正的方法
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }
   // 获取下一个拦截器，currentInterceptorIndex记录执行到哪个拦截器
   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
  // 是AOP增强
   if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
      // Evaluate dynamic method matcher here: static part will already have
      // been evaluated and found to match.
      InterceptorAndDynamicMethodMatcher dm =
            (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
      // 判断调用是否匹配，如果匹配则调用拦截器方法
      if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
         return dm.interceptor.invoke(this);
      }
      else {
         // Dynamic matching failed.
         // Skip this interceptor and invoke the next in the chain.
         // 否则递归执行下一个拦截器
         return proceed();
      }
   }
   else {
      // It's an interceptor, so we just invoke it: The pointcut will have
      // been evaluated statically before this object was constructed.
      // 它是一个拦截器，所以我们只调用它:在构造这个对象之前，切入点将被静态地计算。
      // 咱们这里最终调用的是((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
      // 就是TransactionInterceptor事务拦截器回调 目标业务方法
      return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
   }
}
```

1. 判断增强是InterceptorAndDynamicMethodMatcher即是AOP增强，则先判断方法匹配，后调用增强
2. 如果是事务增强等Spring内部实现的增强直接走((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
3. 如果增强递归执行（在增强里面再次调用ReflectiveMethodInvocation的proceed方法）完毕，则反射执行被增强的方法

分析((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this)方法

真正是调用TransactionInterceptor#invoke方法

```java
public Object invoke(MethodInvocation invocation) throws Throwable {
   // Work out the target class: may be {@code null}.
   // The TransactionAttributeSource should be passed the target class
   // as well as the method, which may be from an interface.
   Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

   // Adapt to TransactionAspectSupport's invokeWithinTransaction...
   return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
}
```



1. 事务逻辑处理
2. 调用ReflectiveMethodInvocation的proceed方法递归调用，达到责任链顺序执行的逻辑
3. 事务逻辑处理

继续分析invokeWithinTransaction方法，分析事务的处理逻辑

**本文只分析传播机制为REQUIRED的事务**

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
      final InvocationCallback invocation) throws Throwable {

   // If the transaction attribute is null, the method is non-transactional.
   TransactionAttributeSource tas = getTransactionAttributeSource();
   // 获取@Transaction里面的配置
   final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
   // 获取TransactionManager，优先级是 Transactional.value>设置的默认的>根据类型获取
   final PlatformTransactionManager tm = determineTransactionManager(txAttr);
   final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

   // 标准声明式事务：如果事务属性为空 或者 非回调偏向的事务管理器
   if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
      // Standard transaction demarcation with getTransaction and commit/rollback calls.
      TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

      Object retVal;
      try {
         // This is an around advice: Invoke the next interceptor in the chain.
         // This will normally result in a target object being invoked.
         // 真正的反射调方法
         retVal = invocation.proceedWithInvocation();
      }
      catch (Throwable ex) {
         // target invocation exception
         // 根据事务定义的，该异常需要回滚就回滚，否则提交事务
         completeTransactionAfterThrowing(txInfo, ex);
         throw ex;
      }
      finally {
         cleanupTransactionInfo(txInfo);
      }
      commitTransactionAfterReturning(txInfo);
      return retVal;
   }

   // 编程式事务：（回调偏向）
   else {
      final ThrowableHolder throwableHolder = new ThrowableHolder();

      // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
      try {
         Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
            TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
            try {
               return invocation.proceedWithInvocation();
            }
            catch (Throwable ex) {
               if (txAttr.rollbackOn(ex)) {
                  // A RuntimeException: will lead to a rollback.
                  if (ex instanceof RuntimeException) {
                     throw (RuntimeException) ex;
                  }
                  else {
                     throw new ThrowableHolderException(ex);
                  }
               }
               else {
                  // A normal return value: will lead to a commit.
                  throwableHolder.throwable = ex;
                  return null;
               }
            }
            finally {
               cleanupTransactionInfo(txInfo);
            }
         });

         // Check result state: It might indicate a Throwable to rethrow.
         if (throwableHolder.throwable != null) {
            throw throwableHolder.throwable;
         }
         return result;
      }
      catch (ThrowableHolderException ex) {
         throw ex.getCause();
      }
      catch (TransactionSystemException ex2) {
         if (throwableHolder.throwable != null) {
            logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
            ex2.initApplicationException(throwableHolder.throwable);
         }
         throw ex2;
      }
      catch (Throwable ex2) {
         if (throwableHolder.throwable != null) {
            logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
         }
         throw ex2;
      }
   }
}
```

1. 获取事务@Transaction注解
2. 获取TransactionManager，优先级是 Transactional.value>设置的默认的>根据类型获取
3. 开启事务，并把connection绑定到线程上
4. 利用ReflectiveMethodInvocation的proceed方法反射调被事务增强的方法，如果结合Mybatis的话，Mybatis动态代理生成的类会从线程上拿到connection，保证一次事务使用的是一个connection
5. 事务提交或者回滚

分析createTransactionIfNecessary开启事务

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
      @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

   // If no name specified, apply method identification as transaction name.
   // 如果还没有定义名字，把连接点的ID定义成事务的名称
   if (txAttr != null && txAttr.getName() == null) {
      txAttr = new DelegatingTransactionAttribute(txAttr) {
         @Override
         public String getName() {
            return joinpointIdentification;
         }
      };
   }

   TransactionStatus status = null;
   if (txAttr != null) {
      if (tm != null) {
         // 根据事务属性获取事务TransactionStatus，都是调用PlatformTransactionManager.getTransaction()
         status = tm.getTransaction(txAttr);
      }
      else {
         if (logger.isDebugEnabled()) {
            logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                  "] because no transaction manager has been configured");
         }
      }
   }
   // 构造一个TransactionInfo事务信息对象，绑定当前线程：ThreadLocal<TransactionInfo>
   return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

继续分析getTransaction方法

```java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
   // 得到DataSourceTransactionObject 含有ConnectionHolder属性（含有connection）
   Object transaction = doGetTransaction();

   // Cache debug flag to avoid repeated checks.
   boolean debugEnabled = logger.isDebugEnabled();

   if (definition == null) {
      // Use defaults if no transaction definition given.
      definition = new DefaultTransactionDefinition();
   }

   // 如果当前已经存在事务
   if (isExistingTransaction(transaction)) {
      // Existing transaction found -> check propagation behavior to find out how to behave.
      // 根据不同传播机制不同处理
      return handleExistingTransaction(definition, transaction, debugEnabled);
   }

   // Check definition settings for new transaction.
   // 超时不能小于默认值
   if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
      throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
   }

   // No existing transaction found -> check propagation behavior to find out how to proceed.
   // 当前不存在事务，传播机制=MANDATORY（支持当前事务，没事务报错），报错
   if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
      throw new IllegalTransactionStateException(
            "No existing transaction found for transaction marked with propagation 'mandatory'");
   }
   // 当前不存在事务，传播机制=REQUIRED/REQUIRED_NEW/NESTED,这三种情况，需要新开启事务，且加上事务同步
   else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
         definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
         definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
      SuspendedResourcesHolder suspendedResources = suspend(null);
      if (debugEnabled) {
         logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
      }
      try {
         // 是否需要新开启同步// 开启
         boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
         DefaultTransactionStatus status = newTransactionStatus(
               definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
         // 开启新事务
         doBegin(transaction, definition);
         prepareSynchronization(status, definition);
         return status;
      }
      catch (RuntimeException | Error ex) {
         resume(null, suspendedResources);
         throw ex;
      }
   }
   else {
      // // 当前不存在事务当前不存在事务，且传播机制=PROPAGATION_SUPPORTS/PROPAGATION_NOT_SUPPORTED/PROPAGATION_NEVER，
      // 这三种情况，创建“空”事务:没有实际事务，但可能是同步。警告：定义了隔离级别，
      // 但并没有真实的事务初始化，隔离级别被忽略有隔离级别但是并没有定义实际的事务初始化，有隔离级别但是并没有定义实际的事务初始化，
      // Create "empty" transaction: no actual transaction, but potentially synchronization.
      if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
         logger.warn("Custom isolation level specified but no actual transaction initiated; " +
               "isolation level will effectively be ignored: " + definition);
      }
      boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
      return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
   }
}
```

1. 得到DataSourceTransactionObject 含有ConnectionHolder属性，其中ConnectionHolder含有connection
2. 根据事务的传播机制不同走不同的策略来获取事务

本文只分析REQUIRED传播机制，即默认的传播机制

分析doBegin获取新事务的逻辑

```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
   Connection con = null;

   try {
      // 如果事务还没有connection或者connection在事务同步状态，重置新的connectionHolder
      if (!txObject.hasConnectionHolder() ||
            txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
         Connection newCon = obtainDataSource().getConnection();
         if (logger.isDebugEnabled()) {
            logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
         }
         // 重置新的connectionHolder
         txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
      }

      //设置新的连接为事务同步中
      txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
      con = txObject.getConnectionHolder().getConnection();

      //conn设置事务隔离级别,只读
      Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
      txObject.setPreviousIsolationLevel(previousIsolationLevel);

      // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
      // so we don't want to do it unnecessarily (for example if we've explicitly
      // configured the connection pool to set it already).
      // 如果是自动提交切换到手动提交
      if (con.getAutoCommit()) {
         txObject.setMustRestoreAutoCommit(true);
         if (logger.isDebugEnabled()) {
            logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
         }
         con.setAutoCommit(false);
      }

      // 如果只读，执行sql设置事务只读
      prepareTransactionalConnection(con, definition);
      txObject.getConnectionHolder().setTransactionActive(true);// 设置connection持有者的事务开启状态

      int timeout = determineTimeout(definition);
      if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
         txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
      }

      // Bind the connection holder to the thread.
      // 绑定connection持有者到当前线程
      if (txObject.isNewConnectionHolder()) {
         TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
      }
   }

   catch (Throwable ex) {
      if (txObject.isNewConnectionHolder()) {
         DataSourceUtils.releaseConnection(con, obtainDataSource());
         txObject.setConnectionHolder(null, false);
      }
      throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
   }
}
```

1.  如果事务还没有connection或者connection在事务同步状态，重置新的connectionHolder，即从线程池中获取新的connection。
2. 设置事务为手动提交
3.  绑定connectionHolder，即含有connection的类到当前线程

分析TransactionSynchronizationManager.bindResource的绑定含有connection的到当前线程

```java
public static void bindResource(Object key, Object value) throws IllegalStateException {
   Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
   Assert.notNull(value, "Value must not be null");
   Map<Object, Object> map = resources.get();
   // set ThreadLocal Map if none found
   if (map == null) {
      map = new HashMap<>();
      resources.set(map);
   }
   Object oldValue = map.put(actualKey, value);
   // Transparently suppress a ResourceHolder that was marked as void...
   if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
      oldValue = null;
   }
   if (oldValue != null) {
      throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
            actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
   }
   if (logger.isTraceEnabled()) {
      logger.trace("Bound value [" + value + "] for key [" + actualKey + "] to thread [" +
            Thread.currentThread().getName() + "]");
   }
}
```

**将map放到线程上，其中key为datasource，value为ConnectionHolder，后面Mybatis也是从线程上根据datasource拿到ConnectionHolder，再拿到connection。**



继续分析根据ReflectiveMethodInvocation的proceed方法来反射调方法的逻辑，Mybatis动态代理生成的类会从线程上拿到connection，保证一次事务使用的是一个connection

反射调方法是最终会调到Mybatis的动态代理类SqlSessionInterceptor的invoke这个方法

```java
private class SqlSessionInterceptor implements InvocationHandler {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    SqlSession sqlSession = getSqlSession(
        SqlSessionTemplate.this.sqlSessionFactory,
        SqlSessionTemplate.this.executorType,
        SqlSessionTemplate.this.exceptionTranslator);
    try {
      Object result = method.invoke(sqlSession, args);
      if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
        // force commit even on non-dirty sessions because some databases require
        // a commit/rollback before calling close()
        sqlSession.commit(true);
      }
      return result;
    } catch (Throwable t) {
      Throwable unwrapped = unwrapThrowable(t);
      if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
        // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        sqlSession = null;
        Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
        if (translated != null) {
          unwrapped = translated;
        }
      }
      throw unwrapped;
    } finally {
      if (sqlSession != null) {
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
      }
    }
  }
}
```

分析getSqlSession的获取SqlSession的逻辑

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

  notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
  notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

  SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

  SqlSession session = sessionHolder(executorType, holder);
  if (session != null) {
    return session;
  }

  LOGGER.debug(() -> "Creating a new SqlSession");
  session = sessionFactory.openSession(executorType);

  registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

  return session;
}
```

1. 没有在线程上获取到则创建

继续分析sessionFactory.openSession新建SqlSession的逻辑

```java
public SqlSession openSession(ExecutorType execType) {
  return openSessionFromDataSource(execType, null, false);
}
```

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    final Executor executor = configuration.newExecutor(tx, execType);
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    closeTransaction(tx); // may have fetched a connection so lets call close()
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

其中DefaultSqlSession中含有数据源datasource属性

数据源的设置在

```java
@Bean
   public SqlSessionFactory sqlSessionFactory() throws Exception {
      SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
      org.apache.ibatis.session.Configuration configuration=new org.apache.ibatis.session.Configuration();
      configuration.setLogImpl(Log4jImpl.class);
      sessionFactory.setDataSource(dataSource());
//    sessionFactory.setMapperLocations();
      return sessionFactory.getObject();
   }
```



在Mybatis真正执行sql的时候再通过DefaultSqlSession中的datasource属性来获取connection。



```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  Connection connection = getConnection(statementLog);
  stmt = handler.prepare(connection, transaction.getTimeout());
  handler.parameterize(stmt);
  return stmt;
}
```

在执行prepareStatement的时候获取connection

```java
protected Connection getConnection(Log statementLog) throws SQLException {
  Connection connection = transaction.getConnection();
  if (statementLog.isDebugEnabled()) {
    return ConnectionLogger.newInstance(connection, statementLog, queryStack);
  } else {
    return connection;
  }
}
```



```java
public Connection getConnection() throws SQLException {
  if (this.connection == null) {
    openConnection();
  }
  return this.connection;
}
```

```java
private void openConnection() throws SQLException {
  this.connection = DataSourceUtils.getConnection(this.dataSource);
  this.autoCommit = this.connection.getAutoCommit();
  this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);

  LOGGER.debug(() ->
      "JDBC Connection ["
          + this.connection
          + "] will"
          + (this.isConnectionTransactional ? " " : " not ")
          + "be managed by Spring");
}
```

最终调到Spring的DataSourceUtils工具类来获取connection

继续分析DataSourceUtils的getConnection方法

```java
public static Connection getConnection(DataSource dataSource) throws CannotGetJdbcConnectionException {
   try {
      return doGetConnection(dataSource);
   }
   catch (SQLException ex) {
      throw new CannotGetJdbcConnectionException("Failed to obtain JDBC Connection", ex);
   }
   catch (IllegalStateException ex) {
      throw new CannotGetJdbcConnectionException("Failed to obtain JDBC Connection: " + ex.getMessage());
   }
}
```

```java
public static Connection doGetConnection(DataSource dataSource) throws SQLException {
   Assert.notNull(dataSource, "No DataSource specified");
   // 先从线程上根据dataSource获取ConnectionHolder
		ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
		if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
			conHolder.requested();
			if (!conHolder.hasConnection()) {
				logger.debug("Fetching resumed JDBC Connection from DataSource");
				conHolder.setConnection(fetchConnection(dataSource));
			}
			return conHolder.getConnection();
		}
		// Else we either got no holder or an empty thread-bound holder here.

		logger.debug("Fetching JDBC Connection from DataSource");
		// 拿到ConnectionHolder中的connection
   Connection con = fetchConnection(dataSource);

   if (TransactionSynchronizationManager.isSynchronizationActive()) {
      try {
         // Use same Connection for further JDBC actions within the transaction.
         // Thread-bound object will get removed by synchronization at transaction completion.
         ConnectionHolder holderToUse = conHolder;
         if (holderToUse == null) {
            holderToUse = new ConnectionHolder(con);
         }
         else {
            holderToUse.setConnection(con);
         }
         holderToUse.requested();
         TransactionSynchronizationManager.registerSynchronization(
               new ConnectionSynchronization(holderToUse, dataSource));
         holderToUse.setSynchronizedWithTransaction(true);
         if (holderToUse != conHolder) {
            TransactionSynchronizationManager.bindResource(dataSource, holderToUse);
         }
      }
      catch (RuntimeException ex) {
         // Unexpected exception from external delegation call -> close Connection and rethrow.
         releaseConnection(con, dataSource);
         throw ex;
      }
   }

   return con;
}
```

1. 先从线程上根据dataSource获取ConnectionHolder
2. 拿到ConnectionHolder中的connection

继续分析TransactionSynchronizationManager.getResource方法

```java
public static Object getResource(Object key) {
   Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
   // 从 ThreadLocal 上根据 actualKey 获取其对应的value
   Object value = doGetResource(actualKey);
   if (value != null && logger.isTraceEnabled()) {
      logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
            Thread.currentThread().getName() + "]");
   }
   return value;
}
```

```java
private static Object doGetResource(Object actualKey) {
   Map<Object, Object> map = resources.get();
   if (map == null) {
      return null;
   }
   Object value = map.get(actualKey);
   // Transparently remove ResourceHolder that was marked as void...
   if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
      map.remove(actualKey);
      // Remove entire ThreadLocal if empty...
      if (map.isEmpty()) {
         resources.remove();
      }
      value = null;
   }
   return value;
}
```

继续分析completeTransactionAfterThrowing方法，来确定是回滚还是提交事务

```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
   // 最终调用AbstractPlatformTransactionManager的rollback()，
   // 提交事务commitTransactionAfterReturning()最终调用AbstractPlatformTransactionManager的commit()
   if (txInfo != null && txInfo.getTransactionStatus() != null) {
      if (logger.isTraceEnabled()) {
         logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
               "] after exception: " + ex);
      }
      if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
         try {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
         }
      }
      else {
         // We don't roll back on this exception.
         // Will still roll back if TransactionStatus.isRollbackOnly() is true.
         try {
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
         }
         catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
         }
         catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            throw ex2;
         }
      }
   }
}
```

继续分析txInfo.getTransactionManager().commit方法

```java
public final void commit(TransactionStatus status) throws TransactionException {
   if (status.isCompleted()) {// 如果事务已完结，报错无法再次提交
      throw new IllegalTransactionStateException(
            "Transaction is already completed - do not call commit or rollback more than once per transaction");
   }

   DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
   if (defStatus.isLocalRollbackOnly()) {// 如果事务明确标记为回滚，
      if (defStatus.isDebug()) {
         logger.debug("Transactional code has requested rollback");
      }
      //执行回滚
      processRollback(defStatus, false);
      return;
   }

   //如果不需要全局回滚时提交 且 全局回滚
   if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
      if (defStatus.isDebug()) {
         logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
      }
      //执行回滚
      processRollback(defStatus, true);
      return;
   }

   // 执行提交事务
   processCommit(defStatus);
}
```

分析processCommit方法

```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
   try {
      boolean beforeCompletionInvoked = false;

      try {
         //3个前置操作
         boolean unexpectedRollback = false;
         prepareForCommit(status);
         //遍历事务同步器，把每个事务同步器都执行“提交前”操作，比如咱们用的jdbc事务，
         // 那么最终就是SqlSessionUtils.beforeCommit()->this.holder.getSqlSession().commit();提交会话。
         triggerBeforeCommit(status);
         // 完成前触发操作，如果是jdbc事务，那么最终就是
         //SqlSessionUtils.beforeCompletion->
         //TransactionSynchronizationManager.unbindResource(sessionFactory); 解绑当前线程的会话工厂
         //this.holder.getSqlSession().close();关闭会话。
         triggerBeforeCompletion(status);
         beforeCompletionInvoked = true;//新事务 或 全局回滚失败

         // 有保存点，即嵌套事务
         if (status.hasSavepoint()) {
            if (status.isDebug()) {
               logger.debug("Releasing transaction savepoint");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            //释放保存点
            status.releaseHeldSavepoint();
         }// 新事务
         else if (status.isNewTransaction()) {
            if (status.isDebug()) {
               logger.debug("Initiating transaction commit");
            }
            unexpectedRollback = status.isGlobalRollbackOnly();
            //调用事务处理器提交事务
            doCommit(status);
         }

         else if (isFailEarlyOnGlobalRollbackOnly()) {
            unexpectedRollback = status.isGlobalRollbackOnly();
         }

         // Throw UnexpectedRollbackException if we have a global rollback-only
         // marker but still didn't get a corresponding exception from commit.
         // 3.非新事务，且全局回滚失败，但是提交时没有得到异常，抛出异常
         if (unexpectedRollback) {
            throw new UnexpectedRollbackException(
                  "Transaction silently rolled back because it has been marked as rollback-only");
         }
      }
      catch (UnexpectedRollbackException ex) {
         // 触发完成后事务同步，状态为回滚
         // can only be caused by doCommit

         triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
         throw ex;
      }
      catch (TransactionException ex) {
         // 提交失败回滚
         // can only be caused by doCommit
         if (isRollbackOnCommitFailure()) {
            doRollbackOnCommitException(status, ex);
         }
         else {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
         }
         throw ex;
      }
      // 运行时异常// 其它异常
      catch (RuntimeException | Error ex) {
         if (!beforeCompletionInvoked) {
            // 如果3个前置步骤未完成，调用前置的最后一步操作
            triggerBeforeCompletion(status);
         }
         // 提交异常回滚
         doRollbackOnCommitException(status, ex);
         throw ex;
      }

      // Trigger afterCommit callbacks, with an exception thrown there
      // propagated to callers but the transaction still considered as committed.
      try {
         //提交事务后触发操作。
         // TransactionSynchronizationUtils.triggerAfterCommit();->TransactionSynchronizationUtils.invokeAfterCommit，
         triggerAfterCommit(status);
      }
      finally {
         // 如果会话任然活着，关闭会话，
         // 重置各种属性：SQL会话同步器（SqlSessionSynchronization）的SQL会话持有者（SqlSessionHolder）
         // 的referenceCount引用计数、synchronizedWithTransaction同步事务、rollbackOnly只回滚、deadline超时时间点。
         triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
      }

   }
   finally {
      // 设置事务状态为已完成。
      // 如果是新的事务同步，解绑当前线程绑定的数据库资源，重置数据库连接
      // 如果存在挂起的事务（嵌套事务），唤醒挂起的老事务的各种资源：数据库资源、同步器。
      cleanupAfterCompletion(status);
   }
}
```

分析doCommit方法

```java
protected void doCommit(DefaultTransactionStatus status) {
   DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
   Connection con = txObject.getConnectionHolder().getConnection();
   if (status.isDebug()) {
      logger.debug("Committing JDBC transaction on Connection [" + con + "]");
   }
   try {
      con.commit();
   }
   catch (SQLException ex) {
      throw new TransactionSystemException("Could not commit JDBC transaction", ex);
   }
}
```

最终调用con.commit()来提交事务

### 小结

1. 获取事务@Transaction注解
2. 获取TransactionManager，优先级是 Transactional.value>设置的默认的>根据类型获取
3.  如果事务还没有connection或者connection在事务同步状态，重置新的connectionHolder（含有connection属性），即从线程池中获取新的connection。
4. 设置事务为手动提交
5. 绑定map放到线程上，其中一个key为datasource，value为ConnectionHolder（含有connection属性），后面Mybatis也是从线程上根据datasource拿到ConnectionHolder，再拿到connection
6. 利用ReflectiveMethodInvocation的proceed方法反射调被事务增强的方法，如果结合Mybatis的话，Mybatis动态代理生成的类会从线程上拿到connection，保证一次事务使用的是一个connection
7. Mybatis动态代理生成类在真正执行sql调prepareStatement的时候最终调到Spring的DataSourceUtils工具类来获取connection。
8. 先从线程上根据dataSource获取ConnectionHolder，再拿到ConnectionHolder中的connection
9. 利用connection进行提交或者回滚

**重要：SqlSessionFactory中的dataSource要和DataSourceTransactionManager中的dataSource是一个引用，否则导致事务不生效，通常见于多数据源的场景，我们线上出现过这种错误**



## 总结



