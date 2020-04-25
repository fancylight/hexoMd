---
title: Spring_aop织入分析
date: 2020-03-04 19:57:05
tags:
- 框架
- spring
- springAop
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
### 概述
本文用来分析`aop`在`spring`的织入原理,今后的博客会尽量分段,将篇幅降低,更加易于阅读.
此文涉及到的一些原理参考[aop](/2019/05/25/aop/)
#### 使用
##### DefaultAdvisorAutoProxyCreator
```xml
<beans
    <!--自动代理-->
    <bean id="advisorAutoProxyCreator" class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />
    <bean id="myInstance" class="aop.jdkProxy.MyProxyInstance"/>
    <bean id="advisor" class="advisor.StaticAdvisor">
        <property name="advice">
            <bean class="aop.BeforeMethod"></bean>
        </property>
    </bean>
</beans>
```
简要说明:
- MyProxyInstance是`ProxyInterface`子类,接口仅仅定义了`void test()`
- 未使用 p:proxyTargetClass="true",即默认使用jdk代理
```java
@Test
   public void test(){
        //载入bf
       xml.loadBeanDefinitions("aop1.xml");
       //bf添加后处理
       var proxy=beanFactory.getBean("advisorAutoProxyCreator",DefaultAdvisorAutoProxyCreator.class);
       beanFactory.addBeanPostProcessor(proxy);
       //此处若转型不是接口则会抛出异常
       //beanFactory.getBean("myInstance",MyProxyInstance.class).test();
       beanFactory.getBean("myInstance", ProxyInterface.class).test();
   }
```
简要说明
  - 我没有使用`context`是为了更好的说明
  - 异常的原因在于`AbstractBeanFactory#doGetBean`,最后会再进行一次类型转换,就是为了将创造的bean和`getBean`传递的参数进行转换,
  一般到了此步是没有转换的必要,但是代理器处理过的bean已经是proxy了,后文无法匹配到合适的转换器,因此抛出异常[^1]

##### BeanNameAutoProxyCreator
```xml
<beans>
    <bean id="myInstance" class="aop.jdkProxy.MyProxyInstance"/>
    <bean id="advice" class="aop.BeforeMethod"/>
    <bean id="myInstanceProxy" class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator"
          p:proxyTargetClass="true"
          p:interceptorNames="advice" p:beanNames="myInstance"/>
</beans>
```
java代码和上类似,只是此处使用了`p:proxyTargetClass="true"`,即调用cglib,就可以直接转换成源类型
##### AnnotationAwareAspectJAutoProxyCreator
```xml
<aop:aspectj-autoproxy>
```
开启自动aspectj,实际是创建AnnotationAwareAspectJAutoProxyCreator处理器
#### 分析源码
{%asset_img DefaultAdvisorAutoProxyCreator.png DefaultAdvisorAutoProxyCreator%}
{%asset_img BeanNameAutoProxyCreator.png BeanNameAutoProxyCreator%}
{%asset_img AnnotationAwareAspectJAutoProxyCreator.png AnnotationAwareAspectJAutoProxyCreator%}
##### AbstractAutoProxyCreator核心逻辑
###### 抽象逻辑
{%codeblock lang:java %}
  //这两个object都代表拦截器,可能是advice 或advisor
    protected static final Object[] DO_NOT_PROXY = null;
  protected static final Object[] PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS = new Object[0];
  //MethodInterceptor生成器
  private AdvisorAdapterRegistry advisorAdapterRegistry = GlobalAdvisorAdapterRegistry.getInstance();
  //key实际上就是后文的cacheKey,表示代理源是谁,value表示代理对象(如jdk代理后的proxy)
  private final Map<Object, Class<?>> proxyTypes = new ConcurrentHashMap<>(16);
  //key如上,value表示ture,此bean可以代理,false表示不能代理
  private final Map<Object, Boolean> advisedBeans = new ConcurrentHashMap<>(256);
  //拦截器名
  private String[] interceptorNames = new String[0];
//-----------------  postProcessBeforeInstantiation-----------------------------
  public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
          Object cacheKey = getCacheKey(beanClass, beanName);
      //第一次
          if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
              if (this.advisedBeans.containsKey(cacheKey)) { //已经处理
                  return null;
              }
        //若为组件类,永不处理,并跳过
              if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
                  this.advisedBeans.put(cacheKey, Boolean.FALSE);
                  return null;
              }
          }

          // Create proxy here if we have a custom TargetSource.
          // Suppresses unnecessary default instantiation of the target bean:
          // The TargetSource will handle target instances in a custom fashion.
          TargetSource targetSource = getCustomTargetSource(beanClass, beanName); //CTS给了从别处创建TargetSource机会,一般没有设置CTS,则会返回null
          if (targetSource != null) {
              if (StringUtils.hasLength(beanName)) {
                  this.targetSourcedBeans.add(beanName);
              }
        //子类实现获取拦截器
              Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        //创建代理对象
              Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
              this.proxyTypes.put(cacheKey, proxy.getClass());
              return proxy;
          }

          return null;
      }
  //----------------------------postProcessAfterInitialization-------------------------------
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (!this.earlyProxyReferences.contains(cacheKey)) {
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }

  //-----------------------------getCacheKey------------------------------------
  protected Object getCacheKey(Class<?> beanClass, @Nullable String beanName) {
    如果是匿名bean则返回Class,否则返回真是的beanName
  if (StringUtils.hasLength(beanName)) {
    return (FactoryBean.class.isAssignableFrom(beanClass) ?
        BeanFactory.FACTORY_BEAN_PREFIX + beanName : beanName);
  }
  else {
    return beanClass;
  }
  //----------------------------wrapIfNecessary-----------------------------
  protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //targetSourcedBeans说明before处理过
        if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
            return bean;
        }
    //不处理的bean
        if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
            return bean;
        }
    //组件bean,永不处理
        if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }

        // Create proxy if we have advice.
    //子类创建拦截器
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        if (specificInterceptors != DO_NOT_PROXY) { //子类没有放弃,并非是子类一定要提供拦截器(BeanNameAutoProxyCreator就是如此)
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            Object proxy = createProxy(
                    bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        }

        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
//----------------------------------createProxy------------------------------------------
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
            @Nullable Object[] specificInterceptors, TargetSource targetSource) {

        if (this.beanFactory instanceof ConfigurableListableBeanFactory) { //给当前bd#attr 设置originalTargetClass-targetClass这样一个属性
            AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
        }
    //创建proxyFactory并设置拷贝设置
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.copyFrom(this);


        if (!proxyFactory.isProxyTargetClass()) { //未设置
            if (shouldProxyTargetClass(beanClass, beanName)) {
                proxyFactory.setProxyTargetClass(true);
            }
            else { //获取beanClass所有接口并添加到当前proxyFacotry中
                evaluateProxyInterfaces(beanClass, proxyFactory);
            }
        }

        Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    //准备proxyFactory,并创建代理
        proxyFactory.addAdvisors(advisors);
        proxyFactory.setTargetSource(targetSource);
        customizeProxyFactory(proxyFactory);

        proxyFactory.setFrozen(this.freezeProxy);
        if (advisorsPreFiltered()) {
            proxyFactory.setPreFiltered(true);
        }

        return proxyFactory.getProxy(getProxyClassLoader());
    }

  //-------------------buildAdvisors-------------------------------
  protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
        // Handle prototypes correctly...
        Advisor[] commonInterceptors = resolveInterceptorNames();

        List<Object> allInterceptors = new ArrayList<>();
    //如果子类传递了Interceptors,并且applyCommonInterceptorsFirst==false,则将子类的排在前面
        if (specificInterceptors != null) {
            allInterceptors.addAll(Arrays.asList(specificInterceptors));
            if (commonInterceptors.length > 0) {
                if (this.applyCommonInterceptorsFirst) {
                    allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
                }
                else {
                    allInterceptors.addAll(Arrays.asList(commonInterceptors));
                }
            }
        }
        if (logger.isTraceEnabled()) {
            int nrOfCommonInterceptors = commonInterceptors.length;
            int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
            logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
                    " common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
        }

        Advisor[] advisors = new Advisor[allInterceptors.size()];
        for (int i = 0; i < allInterceptors.size(); i++) {
            advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
        }
        return advisors; //返回
    }

    /**
     * Resolves the specified interceptor names to Advisor objects.
     * @see #setInterceptorNames
     */
    private Advisor[] resolveInterceptorNames() {
    //根据interceptorNames获取advisor
        BeanFactory bf = this.beanFactory;
        ConfigurableBeanFactory cbf = (bf instanceof ConfigurableBeanFactory ? (ConfigurableBeanFactory) bf : null);
        List<Advisor> advisors = new ArrayList<>();
        for (String beanName : this.interceptorNames) {
            if (cbf == null || !cbf.isCurrentlyInCreation(beanName)) {
                Assert.state(bf != null, "BeanFactory required for resolving interceptor names");
                Object next = bf.getBean(beanName);
                advisors.add(this.advisorAdapterRegistry.wrap(next));
            }
        }
        return advisors.toArray(new Advisor[0]);
    }
}
//----------------------wrap-----------------------
public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
        if (adviceObject instanceof Advisor) {
            return (Advisor) adviceObject;
        }
        if (!(adviceObject instanceof Advice)) {
            throw new UnknownAdviceTypeException(adviceObject);
        }
        Advice advice = (Advice) adviceObject;
        if (advice instanceof MethodInterceptor) { //引介增强也是一个MethodInterceptor
            // So well-known it doesn't even need an adapter.
            return new DefaultPointcutAdvisor(advice);
        }
        for (AdvisorAdapter adapter : this.adapters) {
            // Check that it is supported.
            if (adapter.supportsAdvice(advice)) {
                return new DefaultPointcutAdvisor(advice);
            }
        }
        throw new UnknownAdviceTypeException(advice);
    }
{%endcodeblock%}

参考链接[^2]
###### 子类获取advisor
- BeanNameAutoProxyCreator实现
    {%codeblock lang:java BeanNameAutoProxyCreator%}
    //用户要代理的源bean
    private List<String> beanNames;

    protected Object[] getAdvicesAndAdvisorsForBean(
                Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

            if (this.beanNames != null) {
                for (String mappedName : this.beanNames) {
                    if (FactoryBean.class.isAssignableFrom(beanClass)) { //当前bean是FB情况,
                        if (!mappedName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
                            continue;
                        }
              //获取真实的beanName
                        mappedName = mappedName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
                    }
                    if (isMatch(beanName, mappedName)) { //若能匹配返回父类的默认属性
                        return PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS;
                    }
            //别名情况
                    BeanFactory beanFactory = getBeanFactory();
                    if (beanFactory != null) {
                        String[] aliases = beanFactory.getAliases(beanName);
                        for (String alias : aliases) {
                            if (isMatch(alias, mappedName)) {
                                return PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS;
                            }
                        }
                    }
                }
            }
        //若到此处说明不进行代理
            return DO_NOT_PROXY;
        }
      /**
           * Return if the given bean name matches the mapped name.
           * <p>The default implementation checks for "xxx*", "*xxx" and "*xxx*" matches,
           * as well as direct equality. Can be overridden in subclasses.
         *123
      */
      protected boolean isMatch(String beanName, String mappedName) {
        return PatternMatchUtils.simpleMatch(mappedName, beanName);
    }
    {%endcodeblock%}

   - AbstractAdvisorAutoProxyCreator:一般标准实现
   {%codeblock lang:java AbstractAdvisorAutoProxyCreator%}
   //内部获取advisor的类
   private BeanFactoryAdvisorRetrievalHelper advisorRetrievalHelper;
   //------------------getAdvicesAndAdvisorsForBean----------------
   public void setBeanFactory(BeanFactory beanFactory) {
        super.setBeanFactory(beanFactory);
        if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
            throw new IllegalArgumentException(
                    "AdvisorAutoProxyCreator requires a ConfigurableListableBeanFactory: " + beanFactory);
        }
        initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
    }

    protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //初始化BARH
        this.advisorRetrievalHelper = new BeanFactoryAdvisorRetrievalHelperAdapter(beanFactory);
    }
  //-----------------------------getAdvicesAndAdvisorsForBean-----------------------
   protected Object[] getAdvicesAndAdvisorsForBean(
            Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

        List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
        if (advisors.isEmpty()) {
            return DO_NOT_PROXY;
        }
        return advisors.toArray();
    }

  protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //简单来说就是获取bf中的advisor,源码不分析比较简单
  List<Advisor> candidateAdvisors = findCandidateAdvisors();
  List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
  extendAdvisors(eligibleAdvisors);
  if (!eligibleAdvisors.isEmpty()) {
    eligibleAdvisors = sortAdvisors(eligibleAdvisors);
  }
  return eligibleAdvisors;
}
protected List<Advisor> findAdvisorsThatCanApply(
            List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

        ProxyCreationContext.setCurrentProxiedBeanName(beanName);
        try {
            return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
        }
        finally {
            ProxyCreationContext.setCurrentProxiedBeanName(null);
        }
    }
  protected List<Advisor> findCandidateAdvisors() {
        Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
        return this.advisorRetrievalHelper.findAdvisorBeans();
    }

  //---------------------------------AopUtil----------------------------------------------
  //-----------------------------------findAdvisorsThatCanApply---------------------------------------------------
  public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
        if (candidateAdvisors.isEmpty()) {
            return candidateAdvisors;
        }
        List<Advisor> eligibleAdvisors = new ArrayList<>();
        for (Advisor candidate : candidateAdvisors) {
            if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) { //引介advisor仅仅匹配类型
                eligibleAdvisors.add(candidate);
            }
        }
        boolean hasIntroductions = !eligibleAdvisors.isEmpty();
        for (Advisor candidate : candidateAdvisors) {
            if (candidate instanceof IntroductionAdvisor) {
                // already processed
                continue;
            }
            if (canApply(candidate, clazz, hasIntroductions)) { //非引进匹配类型和切点
                eligibleAdvisors.add(candidate);
            }
        }
        return eligibleAdvisors;
    }
  //-------------------------------canApply----------------------------------------
  public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
        if (advisor instanceof IntroductionAdvisor) {
            return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass); //匹配类
        }
        else if (advisor instanceof PointcutAdvisor) {
            PointcutAdvisor pca = (PointcutAdvisor) advisor;
            return canApply(pca.getPointcut(), targetClass, hasIntroductions);
        }
        else {
            // It doesn't have a pointcut so we assume it applies.
            return true;
        }
    }
  public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
        Assert.notNull(pc, "Pointcut must not be null");
        if (!pc.getClassFilter().matches(targetClass)) {
            return false;
        }

        MethodMatcher methodMatcher = pc.getMethodMatcher();
        if (methodMatcher == MethodMatcher.TRUE) { //DefaultPointcutAdvisor中MM就是True
            // No need to iterate the methods if we're matching any method anyway...
            return true;
        }

        IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
        if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
            introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
        }

        Set<Class<?>> classes = new LinkedHashSet<>();
        if (!Proxy.isProxyClass(targetClass)) { //若为代理对象
            classes.add(ClassUtils.getUserClass(targetClass)); //获取原本的类对象
        }
        classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass)); //接口和类对象

        for (Class<?> clazz : classes) { //若该源的任意父接口|父类存在方法匹配则ture
            Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
            for (Method method : methods) {
                if (introductionAwareMethodMatcher != null ?
                        introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
                        methodMatcher.matches(method, targetClass)) {
                    return true;
                }
            }
        }
    //至此不匹配
        return false;
    }
   {%endcodeblock%}
   关于advisor分类为`PointCutAdvisor`和`IntroductionAdvisor`,前者就是非引介类型,即前 后 异常 环绕[^3]
- DefaultAdvisorAutoProxyCreator获取advisor
  {%codeblock lang:java  DefaultAdvisorAutoProxyCreator%}
  //用户设置
  private boolean usePrefix = false;
  //前缀
  private String advisorBeanNamePrefix;
  protected boolean isEligibleAdvisorBean(String beanName) {
          if (!isUsePrefix()) { //无自定义,则恒true
              return true;
          }
          String prefix = getAdvisorBeanNamePrefix();
      //advisor必须拥有前缀
          return (prefix != null && beanName.startsWith(prefix));
      }
  {%endcodeblock%}
- AnnotationAwareAspectJAutoProxyCreator
  {%codeblock lang:java AnnotationAwareAspectJAutoProxyCreator%}
  private AspectJAdvisorFactory aspectJAdvisorFactory;

  @Override
    protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        super.initBeanFactory(beanFactory);
        if (this.aspectJAdvisorFactory == null) {
            this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
        }
        this.aspectJAdvisorsBuilder =
                new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
    }

  @Override
    protected List<Advisor> findCandidateAdvisors() {
        // Add all the Spring advisors found according to superclass rules.
        List<Advisor> advisors = super.findCandidateAdvisors();
        // Build Advisors for all AspectJ aspects in the bean factory.
        if (this.aspectJAdvisorsBuilder != null) {
            advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
        }
        return advisors;
    }
  //-------------------------BeanFactoryAspectJAdvisorsBuilderAdapter------------------------
  public List<Advisor> buildAspectJAdvisors() {
        List<String> aspectNames = this.aspectBeanNames;

        if (aspectNames == null) { //第一次都是null
            synchronized (this) {
                aspectNames = this.aspectBeanNames;
                if (aspectNames == null) {
                    List<Advisor> advisors = new ArrayList<>();
                    aspectNames = new ArrayList<>();
                    String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                            this.beanFactory, Object.class, true, false);
                    for (String beanName : beanNames) {
                        if (!isEligibleBean(beanName)) {
                            continue;
                        }
                        // We must be careful not to instantiate beans eagerly as in this case they
                        // would be cached by the Spring container but would not have been weaved.
                        Class<?> beanType = this.beanFactory.getType(beanName);
                        if (beanType == null) {
                            continue;
                        }
                        if (this.advisorFactory.isAspect(beanType)) { //判断该bean是否为Aspect,根据注解
                            aspectNames.add(beanName);
              //根据源信息创建advisor
                            AspectMetadata amd = new AspectMetadata(beanType, beanName);
                            if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                                MetadataAwareAspectInstanceFactory factory =
                                        new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                                List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                                if (this.beanFactory.isSingleton(beanName)) {
                                    this.advisorsCache.put(beanName, classAdvisors);
                                }
                                else {
                                    this.aspectFactoryCache.put(beanName, factory);
                                }
                                advisors.addAll(classAdvisors);
                            }
                            else {
                                // Per target or per this.
                                if (this.beanFactory.isSingleton(beanName)) {
                                    throw new IllegalArgumentException("Bean with name '" + beanName +
                                            "' is a singleton, but aspect instantiation model is not singleton");
                                }
                                MetadataAwareAspectInstanceFactory factory =
                                        new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                                this.aspectFactoryCache.put(beanName, factory);
                                advisors.addAll(this.advisorFactory.getAdvisors(factory));
                            }
                        }
                    }
                    this.aspectBeanNames = aspectNames;
                    return advisors;
                }
            }
        }

        if (aspectNames.isEmpty()) {
            return Collections.emptyList();
        }
        List<Advisor> advisors = new ArrayList<>();
        for (String aspectName : aspectNames) {
            List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
            if (cachedAdvisors != null) {
                advisors.addAll(cachedAdvisors);
            }
            else {
                MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
                advisors.addAll(this.advisorFactory.getAdvisors(factory));
            }
        }
        return advisors;
    }

  {%endcodeblock%}
  aspectJAdvisorFactory是将aspect转换为advisor的工厂[^4]
[^1]:参考[类型转换](/2020/02/11/spring常见/#属性类型转换)以及[ioc原理](/2019/05/21/ioc/#流程)
[^2]:AdvisorAdapterRegistry参考[aop原理](/2019/05/25/aop/#调用逻辑)
[^3]:参考[advice到advisor](/2019/05/25/aop/#advice到advisor的转换)
[^4]:参考[aspectj使用](/2019/05/25/aop/aspectj使用)
