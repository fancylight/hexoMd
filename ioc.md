---
title: ioc
date: 2019-05-21 10:27:05
tags:
- 框架
- spring
- spirngIoc
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
### ioc源码分析
#### BeanFactory体系
{%asset_img BeanFactory.png DefaultListableBeanFactory%}
该类采用实现众多接口,每个接口功能分离,非常容易理解
{%codeblock lang:java BeanFactory%}
//BeanFactory是一个典型的工厂,通过名字返回对象,通过java反射和配置文件减少了代码耦合
public interface BeanFactory{
<T> T getBean(String name, Class<T> requiredType) throws BeansException;
}


//ListableBeanFactory该接口标示的工厂拥有获取大量bean的能力,如某类型及其子类,获取工厂中持有的bean数量
public interface ListableBeanFactory{
int getBeanDefinitionCount();
String[] getBeanDefinitionNames();
String[] getBeanNamesForType(@Nullable Class<?> type);
}


//HierarchicalBeanFactory 该类型的工厂可以作为继承链使用
public interface HierarchicalBeanFactory extends BeanFactory {
BeanFactory getParentBeanFactory();
boolean containsLocalBean(String name);
}
//SingletonBeanRegistry 表示bean单例注册和获取
public interface SingletonBeanRegistry {
Object getSingleton(String beanName);
void registerSingleton(String beanName, Object singletonObject);
}
//ConfigurableBeanFactory 表示可以对工厂以及本身做出一些改变,如设置工厂的类加载器,设置工厂的父工厂
public interface ConfigurableBeanFactory extends HierarchicalBeanFactory, SingletonBeanRegistry {
void setParentBeanFactory(BeanFactory parentBeanFactory) throws IllegalStateException;
void setBeanClassLoader(@Nullable ClassLoader beanClassLoader);//默认为当前线程加载器
void addBeanPostProcessor(BeanPostProcessor beanPostProcessor);//添加bean处理器

}
//AutowireCapableBeanFactory 完成自动装配的接口,以及创建bean的接口
public interface AutowireCapableBeanFactory extends BeanFactory {

}
//
public interface ConfigurableListableBeanFactory
        extends ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory {}
//定义别名的接口
public interface AliasRegistry {        
void registerAlias(String name, String alias);
}
//注册BeanDfinition的接口,继承别名接口是为了具有给beanName其别名的能力
//BeanDfinition是来描述bean的类,解析xml的定义后生成的
public interface BeanDefinitionRegistry extends AliasRegistry {
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException;
}            
{%endcodeblock%}
#### getBean
##### 流程
{%asset_img getBean流程.png 流程图%}
{%codeblock lang:java AbstractBeanFactory%}
//spring中以do开头的函数是真正实现的函数
public Object getBean(String name) throws BeansException {
        return doGetBean(name, null, null, false);
    }

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
            @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

        //[1]别名和&处理
        //去除&前缀 ;将可能是别名的,转换为beanName
        final String beanName = transformedBeanName(name);
        Object bean;
        //[1]
        //[2]获取单例缓存,此处同时处理关于单例循环依赖的问题
        Object sharedInstance = getSingleton(beanName);
        //[2]

        //[3]存在单例缓存
        if (sharedInstance != null && args == null) {
            if (logger.isTraceEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) { //说明单例循环
                    logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            //该函数判断sharedInstance是否为FactoryBean,若不是直接返回;若是,则判断返回内部bean还是工厂本身
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }
        //[3]
        //不存在缓存: 1.存在于父容器中 2.未创建 3.不是单例
        else {
            //判断可能为非单例情况,是否存在循环依赖,spring不支持非单例的循环依赖
            if (isPrototypeCurrentlyInCreation(beanName)) { //若该bean在创建中,则说明此时处于循环中,则
                throw new BeanCurrentlyInCreationException(beanName);
            }

            //[4]从父容器中获取
            BeanFactory parentBeanFactory = getParentBeanFactory();
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                            nameToLookup, requiredType, args, typeCheckOnly);
                }
                else if (args != null) {
                    // Delegation to parent with explicit args.
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else if (requiredType != null) {
                    // No args -> delegate to standard getBean method.
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
                else {
                    return (T) parentBeanFactory.getBean(nameToLookup);
                }
            }
           //[4]
            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }

            try {
            //[5] 合并merge
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                checkMergedBeanDefinition(mbd, beanName, args);
            //[5]  
                //[6] 创建depend-on依赖的bean
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        registerDependentBean(dep, beanName);
                        try {
                            getBean(dep);
                        }
                        catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }
                //[6]
                //[7]创建实例
                //单例情况
                if (mbd.isSingleton()) {
                //createBean函数是该接口的子类AbstractAutowireCapableBeanFactory实现
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
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
                    //再次处理FactoryBean情况
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }
                //非单例情况,这段代码和getSingleton同等级,prototype类型不需要缓存,因此spring没有为该类型写一个类
                else if (mbd.isPrototype()) {
                    // It's a prototype -> create a new instance.
                    Object prototypeInstance = null;
                    try {
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }
                //web环境下
                else {
                    String scopeName = mbd.getScope();
                    final Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                    //将部分逻辑交给了scope实现类处理,如sessionScope
                        Object scopedInstance = scope.get(beanName, () -> {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        });
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
            //[7]
         //[8]处理类型转换
        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                }
                return convertedBean;
            }
            catch (TypeMismatchException ex) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Failed to convert bean '" + name + "' to required type '" +
                            ClassUtils.getQualifiedName(requiredType) + "'", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
        //[8]
    }

{%endcodeblock%}

- ioc通过几个缓存map解决了单例情况下的循环依赖
- 对于FactoryBean的处理
- spring对于bean的创建的时期通过map来说明,如创建中,创建结束
#### 创建一个新的单例
##### getSingleton
spring的接口非常清晰,对于处理单例,以及缓存单例的函数都在该类中实现
{%codeblock lang:java DefaultSingletonBeanRegistry%}
    //该map存放着最终的单例,FactoryBean类型也在其中
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    //ObjectFactory的最终实现是是包含了当前BeanFactory的lambda表达式,这个map是为了解决依赖循环
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    //该map存在着未实例化完成的bean,解决单例依赖循环的核心
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    //当bean创建完成会存放在该map中,保留了bean创建的顺序
    private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

    //表示正在创建的单例,当调用getSingleton(String beanName, ObjectFactory<?> singletonFactory) 时会管理bean的创建周期
    private final Set<String> singletonsCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    /** Names of beans currently excluded from in creation checks. */
    private final Set<String> inCreationCheckExclusions =
            Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    /** List of suppressed Exceptions, available for associating related causes. */
    @Nullable
    private Set<Exception> suppressedExceptions;

    //正在销毁的单例
    private boolean singletonsCurrentlyInDestruction = false;

    /** Disposable bean instances: bean name to disposable instance. */
    private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

    /** Map between containing bean names: bean name to Set of bean names that the bean contains. */
    private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

    /** Map between dependent bean names: bean name to Set of dependent bean names. */
    private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

    /** Map between depending bean names: bean name to Set of bean names for the bean's dependencies. */
    private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);


    //该接口就是由AbstractBeanFactory#doGetBean()调用的,用来创建新的单例
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
        [1]变量锁,这个锁说明spring对于获取单例的锁必须等一个单例流程结束后才会释放
        //一个单例创建往往伴随着其引用的bean的创建
        synchronized (this.singletonObjects) {
            //存在这句代码的原因在于,进行获取缓存单例的条件是 if (sharedInstance != null && args == null) ,也就是说进入创建语句也有可能是调用者使用了
            //getBean(String name, Object... args),并且给args赋值了,args参数只有创建实例的时候才能使用,也就是说每次非单例调用getBean和单例第一次创建时有用
            //为了防止在单例bean又创建一次,因此又做了一次检查
            //具体arg如何使用要参看createBean代码
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                if (this.singletonsCurrentlyInDestruction) {
                    throw new BeanCreationNotAllowedException(beanName,
                            "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                            "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
                }
                //[2] 设置该bean为创建中状态
                beforeSingletonCreation(beanName);
                //[2]
                boolean newSingleton = false;
                boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = new LinkedHashSet<>();
                }
                try {
                    //[3]创建bean,当前逻辑中调用的是AbstractCapableBeanFactory#createBean()
                    // 该函数具体创建了当前bean,并进行赋值等复杂操作,同时对于其引用bean也进行之前的流程
                    singletonObject = singletonFactory.getObject();
                    newSingleton = true;
                    //[3]
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
                    //[4] 设置bean为创建结束状态
                    afterSingletonCreation(beanName);
                    //[4]
                }
                if (newSingleton) {
                    //[5] 将新创建的bean放到缓存中
                    addSingleton(beanName, singletonObject);
                    //[5]
                }
            }
            return singletonObject;
        }
    }
    //短路逻辑,未放置忽略检查的队列中,若未成功添加到创建队列则抛出异常
    protected void beforeSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }

    /**
     * Callback after singleton creation.
     * <p>The default implementation marks the singleton as not in creation anymore.
     * @param beanName the name of the singleton that has been created
     * @see #isSingletonCurrentlyInCreation
     */
    protected void afterSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
            throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
        }
    }
{%endcodeblock%}
#### createBean
 实例化->装配
{%codeblock lang:java AbstractAutoWireCapableBeanFactory%}
//外壳createBean
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException {

        if (logger.isTraceEnabled()) {
            logger.trace("Creating instance of bean '" + beanName + "'");
        }
        RootBeanDefinition mbdToUse = mbd;

        //[1]解析class对象,确保正确的RootBeanDefinition
        Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }
        //[1]
        //[2] 此处给look-up 和 repalce-method 做标记
        try {
            mbdToUse.prepareMethodOverrides();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                    beanName, "Validation of method overrides failed", ex);
        }
        //[2]
        try {
            //[3] 调用用户注册的InstantiationAwareBeanPostProcessor接口,而不是BeanPostProcess接口
            Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
            if (bean != null) { //若用户实现的before函数创建了bean则直接返回
                return bean;
            }
            //[3]
        }
        catch (Throwable ex) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                    "BeanPostProcessor before instantiation of bean failed", ex);
        }

        try {
        //[4] 真正的createBean函数
            Object beanInstance = doCreateBean(beanName, mbdToUse, args);
            if (logger.isTraceEnabled()) {
                logger.trace("Finished creating instance of bean '" + beanName + "'");
            }
            return beanInstance;
        //[4]
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

//真正的createBean
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
            throws BeanCreationException {

        // [1]构造一个实例,并且由BeanWrapper包裹,此时的实例仅仅通过构造器进行了创建
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
        }
        if (instanceWrapper == null) {
        //此处逻辑复杂,完成autowire 构造器初始化| 默认构造 |
        //此处也完成了对于init-method以及look-up的代理对象创造,如此早期引用也是代理对象,这也是ioc没有把这两种情况放到后处理器中处理的原因
            instanceWrapper = createBeanInstance(beanName, mbd, args);
        }
        final Object bean = instanceWrapper.getWrappedInstance();
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
            mbd.resolvedTargetType = beanType;
        }
        //[1]
        //[2] 调用注册的MergedBeanDefinitionPostProcessor接口来改变bean definition
        //典型 的 AutowiredAnnotationBeanPostProcessor,用来处理autowire原信息
        synchronized (mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                try {
                    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                            "Post-processing of merged bean definition failed", ex);
                }
                mbd.postProcessed = true;
            }
        }
        //[2]
        //[3] 此部分就是为了解决单例循环依赖的核心
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
            if (logger.isTraceEnabled()) {
                logger.trace("Eagerly caching bean '" + beanName +
                        "' to allow for resolving potential circular references");
            }
            //当单例进入创建逻辑,并处于创建中时,到了此处就将放置到\
            //此处还涉及到早期引用,调用自动代理创建代理对象的情况
            addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        }
        //[3]
        //[4] 属性注入
        Object exposedObject = bean;
        try {
            populateBean(beanName, mbd, instanceWrapper); //属性注入核心
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
        //[4]
        if (earlySingletonExposure) {
            Object earlySingletonReference = getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                }
                else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                    String[] dependentBeans = getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                    for (String dependentBean : dependentBeans) {
                        if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }
                    if (!actualDependentBeans.isEmpty()) {
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
            registerDisposableBeanIfNecessary(beanName, bean, mbd);
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
        }

        return exposedObject;
    }
{%endcodeblock%}
##### 装配bean和属性注入
{%codeblock lang:java AbstractAutoWireCapableBeanFactory%}
//装配函数
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
        if (bw == null) {
            if (mbd.hasPropertyValues()) {
                throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            }
            else {
                // Skip property population phase for null instance.
                return;
            }
        }


        boolean continueWithPropertyPopulation = true;
        //[1]若包含InstantiationAwareBeanPostProcessor 则执行after逻辑
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) { //若返回为false则表示bean填充完毕
                        continueWithPropertyPopulation = false;
                        break;
                    }
                }
            }
        }

        if (!continueWithPropertyPopulation) {
            return;
        }
        //[1]
        PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
        //[2]检查是否需要使用自动装填,自动装填并不是真正的将属性注入bean,而是创建关于属性的pvs,这是为了下边可能的处理器调用
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
            MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
            // Add property values based on autowire by name if applicable.
            if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
                autowireByName(beanName, mbd, bw, newPvs);
            }
            // Add property values based on autowire by type if applicable.
            if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
                autowireByType(beanName, mbd, bw, newPvs);
            }
            pvs = newPvs;
        }
        //[2]
        boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
        boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

        PropertyDescriptor[] filteredPds = null;
        //[3]若存在InstantiationAwareBeanPostProcessors,则调用其postProcessProperties来在真正填入属性之前对pvs可以做一次改变
        if (hasInstAwareBpps) {
            if (pvs == null) {
                pvs = mbd.getPropertyValues();
            }
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        if (filteredPds == null) {
                            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                        }
                        pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                        if (pvsToUse == null) {
                            return;
                        }
                    }
                    pvs = pvsToUse;
                }
            }
        }
        //[3]
        if (needsDepCheck) {
            if (filteredPds == null) {
                filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            }
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
        //[4] 开始填充
        if (pvs != null) {
            applyPropertyValues(beanName, mbd, bw, pvs);
        }
        //[4]
    }
//自动装填
protected void autowireByName(
            String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

        String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);//属性名
        for (String propertyName : propertyNames) {
            if (containsBean(propertyName)) {
                Object bean = getBean(propertyName); //根据属性名获取bean
                pvs.add(propertyName, bean);//加入到pvs
                registerDependentBean(propertyName, beanName); //注册依赖
                if (logger.isTraceEnabled()) {
                    logger.trace("Added autowiring by name from bean name '" + beanName +
                            "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
                }
            }
            else {
                if (logger.isTraceEnabled()) {
                    logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                            "' by name: no matching bean found");
                }
            }
        }
    }
//

{%endcodeblock%}
##### pvs处理
pvs处理,此处涉及到类型转换,参考[BeanWrapper](/2020/02/11/spring常见/#beanwrapper)
- valueResolver.resolveValueIfNecessary  将bd#pvs转换成较标准类型
{%codeblock lang:java AbstractAutoWireCapableBeanFactory%}
//一般来说,当前bean是xml定义的,也就是说有<property>,在bd创建过程中才会拥有pvs
//@Component, @Bean 扫描创建的bean,不会经过该函数,类型转换发生在 如autowire..post过程.
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
        if (pvs.isEmpty()) {
            return;
        }

        if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
            ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
        }

        MutablePropertyValues mpvs = null;
        List<PropertyValue> original;
        //[1] 获取pvs,若已经Converted则直接返回
        if (pvs instanceof MutablePropertyValues) {
            mpvs = (MutablePropertyValues) pvs;
            if (mpvs.isConverted()) {
                // Shortcut: use the pre-converted values as-is.
                try {
                    bw.setPropertyValues(mpvs);
                    return;
                }
                catch (BeansException ex) {
                    throw new BeanCreationException(
                            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
                }
            }
            original = mpvs.getPropertyValueList();
        }
        else {
            original = Arrays.asList(pvs.getPropertyValues());
        }
        //[1!]
        //[2] 获取自定义类型转化器,创建Resolver
        TypeConverter converter = getCustomTypeConverter();
        if (converter == null) {
            converter = bw;
        }
        //委托
        BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);
        //[2!]
        // Create a deep copy, resolving any references for values.
        //[3] 遍历处理每个pv
        List<PropertyValue> deepCopy = new ArrayList<>(original.size());
        boolean resolveNecessary = false;
        for (PropertyValue pv : original) {
            if (pv.isConverted()) {
                deepCopy.add(pv);
            }
            else {
                String propertyName = pv.getName();
                Object originalValue = pv.getValue();
                //初步类型转换,实际上很多类型经过此步处理已经可以了,convertForProperty只是一个过场
                Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
                Object convertedValue = resolvedValue;
                boolean convertible = bw.isWritableProperty(propertyName) &&
                        !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
                        //类型转换器转换类型
                if (convertible) {
                    convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
                }
                // Possibly store converted value in merged bean definition,
                // in order to avoid re-conversion for every created bean instance.
                if (resolvedValue == originalValue) {
                    if (convertible) {
                        pv.setConvertedValue(convertedValue);
                    }
                    deepCopy.add(pv);
                }
                else if (convertible && originalValue instanceof TypedStringValue &&
                        !((TypedStringValue) originalValue).isDynamic() &&
                        !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                    pv.setConvertedValue(convertedValue);
                    deepCopy.add(pv);
                }
                else {
                    resolveNecessary = true;
                    deepCopy.add(new PropertyValue(pv, convertedValue));
                }
            }
        }
        if (mpvs != null && !resolveNecessary) {
            mpvs.setConverted();
        }
        //[3!]
        // Set our (possibly massaged) deep copy.
        try {
            bw.setPropertyValues(new MutablePropertyValues(deepCopy));
        }
        catch (BeansException ex) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Error setting property values", ex);
        }
    }

    //----------------------------------- valueResolver.resolveValueIfNecessar---------------------------------
    //该函数将bd#pvs经过初步转换
    public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
        // We must check each value to see whether it requires a runtime reference
        // to another bean to be resolved.
        //[1] 典型的<ref> xml处理时的pv#value类型
        if (value instanceof RuntimeBeanReference) {
            RuntimeBeanReference ref = (RuntimeBeanReference) value;
            return resolveReference(argName, ref);
        }
        //[1!]
        //[2] <idRef> 处理
        else if (value instanceof RuntimeBeanNameReference) {
            String refName = ((RuntimeBeanNameReference) value).getBeanName();
            //如不存在该id,直接抛出异常
            refName = String.valueOf(doEvaluate(refName));
            if (!this.beanFactory.containsBean(refName)) {
                throw new BeanDefinitionStoreException(
                        "Invalid bean name '" + refName + "' in bean reference for " + argName);
            }
            return refName;
        }
        //[2!]
        //[3]
        //<bean><property><bean/><property></bean> 这种情况,即匿名bean
        else if (value instanceof BeanDefinitionHolder) {
            // Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
            BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
            return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
        }
        else if (value instanceof BeanDefinition) {
            // Resolve plain BeanDefinition, without contained name: use dummy name.
            BeanDefinition bd = (BeanDefinition) value;
            String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
                    ObjectUtils.getIdentityHexString(bd);
            return resolveInnerBean(argName, innerBeanName, bd);
        }
        //[3!]
        //[4] 遇到集合类型,若xml中有的如<list>等元素,bd会创建对应的ManageXX,在这里会转换为
        //对应的Collection<String> 类型,或者Map<String,String>类型,而将此处的各种字面值转换
        //为对应的bean中定义的集合<T>类型,并没有发生在该函数
        else if (value instanceof ManagedArray) {
            // May need to resolve contained runtime references.
            ManagedArray array = (ManagedArray) value;
            Class<?> elementType = array.resolvedElementType;
            if (elementType == null) {
                String elementTypeName = array.getElementTypeName();
                if (StringUtils.hasText(elementTypeName)) {
                    try {
                        elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
                        array.resolvedElementType = elementType;
                    }
                    catch (Throwable ex) {
                        // Improve the message by showing the context.
                        throw new BeanCreationException(
                                this.beanDefinition.getResourceDescription(), this.beanName,
                                "Error resolving array type for " + argName, ex);
                    }
                }
                else {
                    elementType = Object.class;
                }
            }
            return resolveManagedArray(argName, (List<?>) value, elementType);
        }
        else if (value instanceof ManagedList) {
            // May need to resolve contained runtime references.
            return resolveManagedList(argName, (List<?>) value);
        }
        else if (value instanceof ManagedSet) {
            // May need to resolve contained runtime references.
            return resolveManagedSet(argName, (Set<?>) value);
        }
        else if (value instanceof ManagedMap) {
            // May need to resolve contained runtime references.
            return resolveManagedMap(argName, (Map<?, ?>) value);
        }
        else if (value instanceof ManagedProperties) {
            Properties original = (Properties) value;
            Properties copy = new Properties();
            original.forEach((propKey, propValue) -> {
                if (propKey instanceof TypedStringValue) {
                    propKey = evaluate((TypedStringValue) propKey);
                }
                if (propValue instanceof TypedStringValue) {
                    propValue = evaluate((TypedStringValue) propValue);
                }
                if (propKey == null || propValue == null) {
                    throw new BeanCreationException(
                            this.beanDefinition.getResourceDescription(), this.beanName,
                            "Error converting Properties key/value pair for " + argName + ": resolved to null");
                }
                copy.put(propKey, propValue);
            });
            return copy;
        }
        //[4]
        //[5] 标准的字面类型,即 value="xx"
        else if (value instanceof TypedStringValue) {
            // Convert value to target type here.
            TypedStringValue typedStringValue = (TypedStringValue) value;
            Object valueObject = evaluate(typedStringValue);
            try {
                //进行类型转换,一般的typeString都是null类型,此处一般也不会做类型转换
                Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
                if (resolvedTargetType != null) {
                    return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
                }
                else {
                    return valueObject;
                }
            }
            catch (Throwable ex) {
                // Improve the message by showing the context.
                throw new BeanCreationException(
                        this.beanDefinition.getResourceDescription(), this.beanName,
                        "Error converting typed String value for " + argName, ex);
            }
        }
        //[5!]
        //[6] null值
        else if (value instanceof NullBean) {
            return null;
        }
        else {
            return evaluate(value);
        }
        //[6!]
    }

    //------------------------------------匿名类处理
    private Object resolveInnerBean(Object argName, String innerBeanName, BeanDefinition innerBd) {
        RootBeanDefinition mbd = null;
        try {
            //[1] 处理merge情况
            mbd = this.beanFactory.getMergedBeanDefinition(innerBeanName, innerBd, this.beanDefinition);
            // Check given bean name whether it is unique. If not already unique,
            // add counter - increasing the counter until the name is unique.
            String actualInnerBeanName = innerBeanName;
            if (mbd.isSingleton()) {
                actualInnerBeanName = adaptInnerBeanName(innerBeanName);
            }
            //[1!]
            //[2] 注册并处理以来
            this.beanFactory.registerContainedBean(actualInnerBeanName, this.beanName);
            // Guarantee initialization of beans that the inner bean depends on.
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dependsOnBean : dependsOn) {
                    this.beanFactory.registerDependentBean(dependsOnBean, actualInnerBeanName);
                    this.beanFactory.getBean(dependsOnBean);
                }
            }
            //[2!]
            // Actually create the inner bean instance now...
            //[3]实际创建
            Object innerBean = this.beanFactory.createBean(actualInnerBeanName, mbd, null);
            if (innerBean instanceof FactoryBean) {
                boolean synthetic = mbd.isSynthetic();
                innerBean = this.beanFactory.getObjectFromFactoryBean(
                        (FactoryBean<?>) innerBean, actualInnerBeanName, !synthetic);
            }
            if (innerBean instanceof NullBean) {
                innerBean = null;
            }
            return innerBean;
        }
        catch (BeansException ex) {
            throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Cannot create inner bean '" + innerBeanName + "' " +
                    (mbd != null && mbd.getBeanClassName() != null ? "of type [" + mbd.getBeanClassName() + "] " : "") +
                    "while setting " + argName, ex);
        }
        //[3!]
    }
{%endcodeblock%}
- 总结:
    - xml中<value>类型对应bd#TypeString,<property name="xx",value="xx">,若该类型没有转换器,则直接异常
    - xml中集合类型对应bd#managerXX,若内部泛型不存在转换器,则异常
    - <ref>对应RuntimeBeanReference,若beanFactory中不存在该bean,则异常
    - <idRef> 对应RuntimeBeanNameReference,实际类型为String,若beanFactory不存在该bean,则异常
    - 匿名bean对应BeanHold,实际每一个正常<bean>也是如此创建,关于xml解析参考[xml解析](http://localhost:4000/2020/02/11/spring%E5%B8%B8%E8%A7%81/#xml解析)
##### 开始初始化
{%codeblock lang:java AbstractAutoWireCapableBeanFactory%}
//初始化bean 完成 aware 接口调用   before init after调用
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                invokeAwareMethods(beanName, bean);
                return null;
            }, getAccessControlContext());
        }
        else {
            invokeAwareMethods(beanName, bean);//调用bean实现的aware接口
        }

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName); //调用BeanPostProcess.before函数,对bean做进行处理,spring的aop就是这么整合进ioc的
        }

        try {
            invokeInitMethods(beanName, wrappedBean, mbd); //调用实现了InitializingBean接口函数,然后再调用bean定义的init-method
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                    (mbd != null ? mbd.getResourceDescription() : null),
                    beanName, "Invocation of init method failed", ex);
        }
        if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName); //调用after
        }

        return wrappedBean; //返回
    }
//aware接口有三种 name classLoader  beanFactory
private void invokeAwareMethods(final String beanName, final Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof BeanNameAware) {
                ((BeanNameAware) bean).setBeanName(beanName);
            }
            if (bean instanceof BeanClassLoaderAware) {
                ClassLoader bcl = getBeanClassLoader();
                if (bcl != null) {
                    ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
                }
            }
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            }
        }
    }
//init的调用
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
            throws Throwable {

        boolean isInitializingBean = (bean instanceof InitializingBean);
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            if (logger.isTraceEnabled()) {
                logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
            }
            if (System.getSecurityManager() != null) {
                try {
                    AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                        ((InitializingBean) bean).afterPropertiesSet();
                        return null;
                    }, getAccessControlContext());
                }
                catch (PrivilegedActionException pae) {
                    throw pae.getException();
                }
            }
            else {
                ((InitializingBean) bean).afterPropertiesSet(); //接口在前
            }
        }

        if (mbd != null && bean.getClass() != NullBean.class) { //init-method在后
            String initMethodName = mbd.getInitMethodName();
            if (StringUtils.hasLength(initMethodName) &&
                    !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                    !mbd.isExternallyManagedInitMethod(initMethodName)) {
                invokeCustomInitMethod(beanName, bean, mbd);
            }
        }
    }
{%endcodeblock%}
{%asset_img bean周期.png%}
- 注意InstantiationAwareBeanPostProcessor 和des....processer的特殊性
- aop的织入就是spring实现了一个BeanPostProcesser实现的
#### 关于单例bean的生命周期和循环依赖问题
- spring通过使singletonsCurrentlyInCreation来表示正在创建中的单例,正在创建中说明该bean至少没有完成所有属性的注入,
简单来说就是singletonObject = singletonFactory.getObject();该函数没有返回
- spring通过一个earlySingletonObjects来存放一个通过反射创建,但是但是没有完成赋值的单例,当进行该单例的属性注入时,也许会遇到循环依赖的情况,此时该map就起了作用
- spring 通过
{%codeblock lang:java DefaultSingletonBeanRegistry%}
//进行单例创建时,当该bean无异常,则将该bean放置到map中
protected void beforeSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }
//当单例创建完毕,remove map
protected void afterSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
            throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
        }
    }
//首次创建单例时,在属性注入前,保存该单例工厂
//这个函数被单例创建完成前调用,和下边的函数相互其作用
//我认为下函数中this.earlySingletonObjects.remove(beanName)不应该存在的理由
//1.创建一个新的单例,并缓存单例工厂 2.存在循环依赖去尝试获取早期单例 3.通过单例工厂获取单例后,存入earlySingletonObjects缓存,供应更深的循环调用
//那么当嵌套的引用创建结束并返回时,总会返回到早期单例中,并结束改单例,根本没有机会再执行addSingletonFactory函数,那么这个remove的意义何在
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(singletonFactory, "Singleton factory must not be null");
        synchronized (this.singletonObjects) {
            if (!this.singletonObjects.containsKey(beanName)) {
                this.singletonFactories.put(beanName, singletonFactory); //存储单例创建工厂,实际就是返回一个早期引用
                //我目前认为该语句不应该存在,原因是如果要进行remove就应该先进行put,而put逻辑在下文中,put的前提是singletonFactory!=null
                this.earlySingletonObjects.remove(beanName);
                this.registeredSingletons.add(beanName);
            }
        }
    }
//尝试命中缓存单例
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
//当一个单例完成后执行
protected void addSingleton(String beanName, Object singletonObject) {
        synchronized (this.singletonObjects) {
            this.singletonObjects.put(beanName, singletonObject);
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
{%endcodeblock%}
#### Context的逻辑
##### 体系概述
- 公共的抽象层AbstractApplicationContext
{%asset_img AbstractApplicationContext.png AbstractApplicationContext%}
由该抽象层实现了公共逻辑,特别是refresh函数的基本逻辑,由该接口分化了两种`AbstractRefreshableApplicationContext`以及`GenericApplicationContext`,
从实现功能上来说,spring特别实现了`WebApplicationContext`用以表示web应用的Context接口
- AbstractRefreshableApplicationContext
{%asset_img AbstractRefreshableApplicationContext.png AbstractRefreshableApplicationContext%}
典型的实现为`ClassPathXmlApplicationContext`
- GenericApplicationContext
{%asset_img GenericApplictionContext.png GenericApplictionContext%}
典型的如springBoot中的`AnnotationConfigServletWebApplicationContext`
- WebApplicationContext
{%asset_img WebApplicationContext.png WebApplicationContext%}
`WebApplicationContext`和上述两种类型并非是对立面,可以任意组合,功能性接口,表示该context是作为web应用使用.
- ARC和GC的区别
    - 前者支持多次调用`refresh`函数重新创建内部beanFactory,后者不会重新销毁内部bf,详情参考`AC#refreshBeanFactory`函数
    - 后者实现了`BeanDefinitionRegistry`接口,因此可以使用多种BeanDefinitionReader加载,源码注释由一段代码如下:
            ```java
            GenericApplicationContext ctx = new GenericApplicationContext();
          XmlBeanDefinitionReader xmlReader = new XmlBeanDefinitionReader(ctx);
          xmlReader.loadBeanDefinitions(new ClassPathResource("applicationContext.xml"));
          PropertiesBeanDefinitionReader propReader = new PropertiesBeanDefinitionReader(ctx);
          propReader.loadBeanDefinitions(new ClassPathResource("otherBeans.properties"));
          ctx.refresh();
            ```

##### 共同逻辑
{%asset_img Context逻辑.png 抽象Context逻辑%}
- AbstractApplicationContext#refresh概要
{%codeblock lang:java AbstractApplicationContext%}
//-----------------构造器-------------------------------
public AbstractApplicationContext() {
    //创建资源解析器
 this.resourcePatternResolver = getResourcePatternResolver();
}
public AbstractApplicationContext(@Nullable ApplicationContext parent) {
 this();
 setParent(parent);
}
//---------------------------ApplicationContext的核心逻辑--------------------------------
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            //设置一些标志和启动数据
            prepareRefresh();
            //两者情况:
            //对于AbstractRefreshableApplicationContext子类,此步骤中完成了xml-->beanDefinition的解析
            //对于GenericApplicationContext,仅仅是设置一个factoryId
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            //向beanFactory添加了一些bean
            prepareBeanFactory(beanFactory);

            try {
                //有些功能是通过,spring的逻辑是在此处提供了一个方法可以对配置好的factory做出一些修改
                postProcessBeanFactory(beanFactory);

                //启动BeanFactoryPostProcessors
                invokeBeanFactoryPostProcessors(beanFactory);

                //注册beanFactory中的BeanPostProcessors,这是为了提前将注册好的BeanProcess初始化,为了之后用户bean的处理做准备
                //比如说注解ioc注入就是通过AutowiredAnnotationBeanPostProcessor完成的
                //自动代理是通过AbstractAdvisorAutoProxyCreator的子类完成的
                registerBeanPostProcessors(beanFactory);

                // 初始化消息源,消息源即国际化处理
                initMessageSource();
                 // 初始化事件广播器,较为好理解的东西
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                // 由子类context来实现一部分独有的逻辑,如boot中servlet会创建服务器
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // 获取所有非lazy的bean,context提前getBean,而不是想beanFactory那样
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();

                // Reset 'active' flag.
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            }

            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
{%endcodeblock%}
- prepareRefresh:设置标记,处理env
{%codeblock lang:java AbstractBeanFactory#prepareRefresh%}
//-----------------------设置标志,将env放置到内部属性中-------------------------
 protected void prepareRefresh() {
     // Switch to active.
     this.startupDate = System.currentTimeMillis();
     this.closed.set(false);
     this.active.set(true);

     if (logger.isDebugEnabled()) {
         if (logger.isTraceEnabled()) {
             logger.trace("Refreshing " + this);
         }
         else {
             logger.debug("Refreshing " + getDisplayName());
         }
     }

     //一般来说是webAC通过env设置ServletContext的值,env的创建者就是Context的创建者,如SpringApplication
     initPropertySources();
     //初始化早期事件监听器和早期事件,这两个属性和下文中提及的事件有关
     if (this.earlyApplicationListeners == null) {
             this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
         }
         else {
             // Reset local application listeners to pre-refresh state.
             this.applicationListeners.clear();
             this.applicationListeners.addAll(this.earlyApplicationListeners);
         }

         // Allow for the collection of early ApplicationEvents,
         // to be published once the multicaster is available...
         this.earlyApplicationEvents = new LinkedHashSet<>();
 }
{%endcodeblock%}
- obtainFreshBeanFactory:获取BF,子类出现分歧
这里的实现就明确说明的上文提及的两种实现的区别
{%codeblock lang:java obtainFreshBeanFactory%}
//------------------AbstractApplicationContext---------------------
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}
//-----------------GenericApplictionContext-----------------------
//不做任何操作,仅仅标记id
protected final void refreshBeanFactory() throws IllegalStateException {
        if (!this.refreshed.compareAndSet(false, true)) {
            throw new IllegalStateException(
                    "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
        }
        this.beanFactory.setSerializationId(getId());
    }
 //返回 内部一直保持的BF
    public final ConfigurableListableBeanFactory getBeanFactory() {
            return this.beanFactory;
        }
//------------------AbstractRefreshableApplicationContext-------------------
protected final void refreshBeanFactory() throws BeansException {
        if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
        }
        try {
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            //设置覆盖以及是否允许早期引用
            customizeBeanFactory(beanFactory);
            //由子类实现,典型的xml..Reader / Groovy /Annotation
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
    }        
    public final ConfigurableListableBeanFactory getBeanFactory() {
        synchronized (this.beanFactoryMonitor) {
            if (this.beanFactory == null) {
                throw new IllegalStateException("BeanFactory not initialized or already closed - " +
                        "call 'refresh' before accessing beans via the ApplicationContext");
            }
            return this.beanFactory;
        }
    }
{%endcodeblock%}
- prepareBeanFactory:配置bF
{%codeblock lang:java prepareBeanFactory%}
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Tell the internal bean factory to use the context's class loader etc.
        beanFactory.setBeanClassLoader(getClassLoader());
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
        beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

        // 设置一个ApplicationContextAwareProcessor,用来处理Bf处理的三种aware之外的aware接口
        beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
        // autowire忽略的接口
        beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
        beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
        beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
        beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

        // BeanFactory interface not registered as resolvable type in a plain factory.
        // MessageSource registered (and found for autowiring) as a bean.
        beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
        beanFactory.registerResolvableDependency(ResourceLoader.class, this);
        beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
        beanFactory.registerResolvableDependency(ApplicationContext.class, this);

        // 该post将属于ApplicationListen的bean添加到当前Context中
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

        // 如果存在LTW(载入时织入),该post用来将LoadTimeWeaverAware类型的bean设置LOAD_TIME_WEAVER_BEAN_NAME
        //
        if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            // Set a temporary ClassLoader for type matching.
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }

        // 注册环境相关bean
        if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
            beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
        }
        if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
            beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
        }
        if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
            beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
        }
    }
{%endcodeblock%}
关于LTW原理参考[LTW原理](/2020/02/11/spring常见/#LTW)
- postProcessBeanFactory:上文完成标准Context初始化, 通过 子类实现,用来 载入bD,如利用内部的reader 或者添加独有的处理器
{%codeblock lang:java postProcessBeanFactory%}
/* Modify the application context's internal bean factory after its standard
* initialization. All bean definitions will have been loaded, but no beans
* will have been instantiated yet. This allows for registering special
* BeanPostProcessors etc in certain ApplicationContext implementations.
* @param beanFactory the bean factory used by the application context
* 用来修改beanFacotry,一般由子类实现
*/
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
//--------------------AnnotationConfigServletWebServerApplicationContext----------------------
//springBoot web项目启动的context此处的逻辑
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        super.postProcessBeanFactory(beanFactory); //ServletWebServerApplicationContext 添加了一个处理,注册了scope
        //注册用户的bean,此处逻辑不要和ConfigClassProcess做的混淆,后者是用来处理bf#map中的@Configuration
        if (this.basePackages != null && this.basePackages.length > 0) {
            this.scanner.scan(this.basePackages);
        }
        if (!this.annotatedClasses.isEmpty()) {
            this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
        }
    }
{%endcodeblock%}
- invokeBeanFactoryPostProcessors:启动BeanFactoryPost
{%codeblock lang:java%}
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

        //假设beanPost又添加了LTW,此时再把LTW处理器加入,原因是要尽量早的将LTW完善,至少要在用户bean创建之前
        if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }
    }
{%endcodeblock%}

- PostProcessorRegistrationDelegate逻辑
{%asset_img BFP.png BeanFactpryProcess调用%}
bf=BeanFactory
BFP=BeanFactoryPostProcessor
BDRP=BeanDefinitionRegistryPostProcessor
{%codeblock lang:java %}
public static void invokeBeanFactoryPostProcessors(
            ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

        // Invoke BeanDefinitionRegistryPostProcessors first, if any.
        // [1] 若存在BeanDefinitionRegistryPostProcessor类型处理器,则首先执行该类型,
        // 该set表示Factory中BeanDefinitionRegistryPostProcessor的bean,以及执行post后新增的
        Set<String> processedBeans = new HashSet<>();

        if (beanFactory instanceof BeanDefinitionRegistry) {//当该beanFactory为可注册的类型时,进行beanDef相关处理
        //否则仅仅遍历调用factoryPost处理器
            BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
            //factory中一般post处理器
            List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
            //registry处理器
            List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

            for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
                if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                    BeanDefinitionRegistryPostProcessor registryProcessor =
                            (BeanDefinitionRegistryPostProcessor) postProcessor;
                    registryProcessor.postProcessBeanDefinitionRegistry(registry);
                    registryProcessors.add(registryProcessor);//registry列表
                }
                else {
                    regularPostProcessors.add(postProcessor);//一般处理器列表
                }
            }

            // Do not initialize FactoryBeans here: We need to leave all regular beans
            // uninitialized to let the bean factory post-processors apply to them!
            // Separate between BeanDefinitionRegistryPostProcessors that implement
            // PriorityOrdered, Ordered, and the rest.
            // 将factory中的实现了BeanDefinitionRegistryPostProcessors的bean按照PriorityOrdered,
            // Ordered顺序进行调用BeanDefinitionRegistryPostProcessors#postProcessBeanDefinitionRegistry
            // 逻辑
            List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

            // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
            // 执行优先post
            String[] postProcessorNames =
                    beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();

            // Next, invoke     the BeanDefinitionRegistryPostProcessors that implement Ordered.
            // 执行顺序post
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();

            // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
            // 执行非顺序接口post
            boolean reiterate = true;
            while (reiterate) {
                reiterate = false;
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                for (String ppName : postProcessorNames) {
                    if (!processedBeans.contains(ppName)) {
                        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                        processedBeans.add(ppName);
                        reiterate = true;
                    }
                }
                sortPostProcessors(currentRegistryProcessors, beanFactory);
                registryProcessors.addAll(currentRegistryProcessors);
                invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                currentRegistryProcessors.clear();
            }

            // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
            // 执行FactoryPost逻辑
            invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
            invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
        }

        else {
            // Invoke factory processors registered with the context instance.
            invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

        // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
        // Ordered, and the rest.
        //找出不是BeanDefinitionRegistryPostProcessor的子类,即到此逻辑还没有执行过FactoryPost的bean的分类并执行post
        //一下逻辑多次获取names和添加set的原因在于执行定义的bean post时可能会创建新的bean def到beanFactory中
        List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
        List<String> orderedPostProcessorNames = new ArrayList<>();
        List<String> nonOrderedPostProcessorNames = new ArrayList<>();
        for (String ppName : postProcessorNames) {
            if (processedBeans.contains(ppName)) {
                // skip - already processed in first phase above
            }
            else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            }
            else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                orderedPostProcessorNames.add(ppName);
            }
            else {
                nonOrderedPostProcessorNames.add(ppName);
            }
        }

        // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
        sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

        // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
        List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
        for (String postProcessorName : orderedPostProcessorNames) {
            orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        sortPostProcessors(orderedPostProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

        // Finally, invoke all other BeanFactoryPostProcessors.
        List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
        for (String postProcessorName : nonOrderedPostProcessorNames) {
            nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

        // Clear cached merged bean definitions since the post-processors might have
        // modified the original metadata, e.g. replacing placeholders in values...
        beanFactory.clearMetadataCache();
    }

    //补充
    //BeanDefinitionRegistryPostProcessor中定义
    /**
     * Modify the application context's internal bean definition registry after its
     * standard initialization. All regular bean definitions will have been loaded,
     * but no beans will have been instantiated yet. This allows for adding further
     * bean definitions before the next post-processing phase kicks in.
     * 当registry完成标准初始化后内容;所有的定义仅仅载入,没有被实例化;在下一次post-process
     * 前添加更多的bean def
     * @param registry the bean definition registry used by the application context
     * @throws org.springframework.beans.BeansException in case of errors
     */
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
    //以springBoot中AnnotationConfigServletWebServerApplicationContext来说,
    //它内部有一个SharedMetadataReaderFactoryContextInitializer#CachingMetadataReaderFactoryPostProcessor
        @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)
                throws BeansException {
            register(registry);
            configureConfigurationClassPostProcessor(registry);
        }
        //添加SharedMetadataReaderFactoryBean
        private void register(BeanDefinitionRegistry registry) {
            BeanDefinition definition = BeanDefinitionBuilder
                    .genericBeanDefinition(SharedMetadataReaderFactoryBean.class,
                            SharedMetadataReaderFactoryBean::new)
                    .getBeanDefinition();
            registry.registerBeanDefinition(BEAN_NAME, definition);
        }
        //从registry获取CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME,并设置了该bean的metadataReaderFactory值
        //该bean实际上是处理注解的处理器,由AnnotationConfigServletWebServerApplicationContext构造函数放置到
        //beanDefMap中
        private void configureConfigurationClassPostProcessor(
                BeanDefinitionRegistry registry) {
            try {
                BeanDefinition definition = registry.getBeanDefinition(
                        AnnotationConfigUtils.CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME);
                definition.getPropertyValues().add("metadataReaderFactory",
                        new RuntimeBeanReference(BEAN_NAME));
            }
            catch (NoSuchBeanDefinitionException ex) {
            }
        }

{%endcodeblock%}
- registerBeanPostProcessors: 提前将bd中属于bpp的创建出来,加入到bf#beanPostProcessors中
{%codeblock lang:java %}
/**
 * Instantiate and invoke all registered BeanPostProcessor beans,
 * respecting explicit order if given.
 * <p>Must be called before any instantiation of application beans.
 * 实例化beanFactory#def中属于beanPost的类
 */
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
//PostProcessorRegistrationDelegate中
//整体逻辑和上边invokefactoryPost类似
public static void registerBeanPostProcessors(
        ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    //获取bean中属于post类型
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

    // Register BeanPostProcessorChecker that logs an info message when
    // a bean is created during BeanPostProcessor instantiation, i.e. when
    // a bean is not eligible for getting processed by all BeanPostProcessors.
    //这个数量= beanFactory内部已经存在的+bean中属于post的,以及该函数最后一行添加的post
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

    // Separate between BeanPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    //排序执行addPost操作
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, register the BeanPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

    // Next, register the BeanPostProcessors that implement Ordered.
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
    for (String ppName : orderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);

    // Now, register all regular BeanPostProcessors.
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
    for (String ppName : nonOrderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

    // Finally, re-register all internal BeanPostProcessors.
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, internalPostProcessors);

    // Re-register post-processor for detecting inner beans as ApplicationListeners,
    // moving it to the end of the processor chain (for picking up proxies etc).
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
{%endcodeblock%}
- initMessageSource:设置内部MessageSource,提供国际化能力
观察一下ApplicationContext是MessageSource的子类,典型的包装类
[MessageSource](http://localhost:4000/2020/02/11/spring%E5%B8%B8%E8%A7%81/#messagesource)
{%codeblock lang:java %}
protected void initMessageSource() {

        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) { //设置内部真实的MessageSource
            this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
            // Make MessageSource aware of parent MessageSource.
            if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
                HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
                if (hms.getParentMessageSource() == null) {
                    // Only set parent context as parent MessageSource if no parent MessageSource
                    // registered already.
                    hms.setParentMessageSource(getInternalParentMessageSource());
                }
            }
            if (logger.isTraceEnabled()) {
                logger.trace("Using MessageSource [" + this.messageSource + "]");
            }
        }
        else {
            //这个dms实际 调用的父类的messageSource
            DelegatingMessageSource dms = new DelegatingMessageSource();
            dms.setParentMessageSource(getInternalParentMessageSource());
            this.messageSource = dms;
            beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
            if (logger.isTraceEnabled()) {
                logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
            }
        }
    }
{%endcodeblock%}
- initApplicationEventMulticaster:注册事件传播
spring事件是常见的监听器模式,参考[事件机制](http://localhost:4000/2019/09/05/springBoot%E5%AD%A6%E4%B9%A0%E4%B8%80/#spring中事件概述)
在抽象类中并没有添加监听器,典型的如SpringApplication构造时
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    //...
    //获取spring.factories中的ApplicationListener,并添加到SpringApplication中
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  //...
}
//随后在prepareContext函数中将监听器加入到了Context中
```
{%codeblock lang:java %}
protected void initApplicationEventMulticaster() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
            this.applicationEventMulticaster =
                    beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
            if (logger.isTraceEnabled()) {
                logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
            }
        }
        else {
            this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
            beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
            if (logger.isTraceEnabled()) {
                logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                        "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
            }
        }
    }
{%endcodeblock%}
- onRefresh:在实例化前,给Context完成自己独特的工作
{%codeblock lang:java%}
/**
 * Template method which can be overridden to add context-specific refresh work.
 * Called on initialization of special beans, before instantiation of singletons.
 * <p>This implementation is empty.
 * @throws BeansException in case of errors
 * @see #refresh()
 */
protected void onRefresh() throws BeansException {
    // For subclasses: do nothing by default.
}
//-----------------------AbstractRefreshableWebApplicationContext-----------------------
//-----------------------GenericWebApplicationContext-----------------------
//----------------------StaticWebApplicationContext------------------------
protected void onRefresh() {
    //初始化ThemeSource
        this.themeSource = UiApplicationContextUtils.initThemeSource(this);
    }
//springBoot中启动了服务器
//---------------------ServletWebServerApplicationContext------------------
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer(); //参考springBoot
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
{%endcodeblock%}
- registerListeners: 处理监听器
{%codeblock lang:java AbstractApplicationContext%}
protected void registerListeners() {
        // Register statically specified listeners first.
        //将context中的监听器添加到播放器中
        for (ApplicationListener<?> listener : getApplicationListeners()) {
            getApplicationEventMulticaster().addApplicationListener(listener);
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let post-processors apply to them!
        // 仅仅添加beanName,并没有实例化当前的监听器
        String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
        for (String listenerBeanName : listenerBeanNames) {
            getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
        }

        // Publish early application events now that we finally have a multicaster...
        //获取早期 事件
        Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
        this.earlyApplicationEvents = null;
        if (earlyEventsToProcess != null) {
            for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
                getApplicationEventMulticaster().multicastEvent(earlyEvent);
            }
        }
    }
 //-------------------------早期事件------------------------
 首先早期事件相关属性初始化在prepareRefresh,及refresh的第一步

 //发布事件
 protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
        Assert.notNull(event, "Event must not be null");

        // Decorate event as an ApplicationEvent if necessary
        ApplicationEvent applicationEvent;
        if (event instanceof ApplicationEvent) {
            applicationEvent = (ApplicationEvent) event;
        }
        else {
            applicationEvent = new PayloadApplicationEvent<>(this, event);
            if (eventType == null) {
                eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
            }
        }

        // Multicast right now if possible - or lazily once the multicaster is initialized
        //如果调用publishEvent的时候早期事件队列已经初始化,则此时不进行事件的传递,一直等到registerListeners的调用
        //即等到context初始化了multicaster
        if (this.earlyApplicationEvents != null) {
            this.earlyApplicationEvents.add(applicationEvent);
        }
        else {
            getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
        }

        // Publish event via parent context as well...
        if (this.parent != null) {
            if (this.parent instanceof AbstractApplicationContext) {
                ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
            }
            else {
                this.parent.publishEvent(event);
            }
        }
    }

{%endcodeblock%}
- finishBeanFactoryInitialization 完成未实例化的bean
{%codeblock lang:java AbstractApplicationContext%}
/**
     * Finish the initialization of this context's bean factory,
     * initializing all remaining singleton beans.
     * 实例bean
     */
    protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        // Initialize conversion service for this context.
        // 设置一个类型转化器
        if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
                beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
            beanFactory.setConversionService(
                    beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
        }

        // Register a default embedded value resolver if no bean post-processor
        // (such as a PropertyPlaceholderConfigurer bean) registered any before:
        // at this point, primarily for resolution in annotation attribute values.
        if (!beanFactory.hasEmbeddedValueResolver()) {
            beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
        }

        // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        for (String weaverAwareName : weaverAwareNames) {
            getBean(weaverAwareName);
        }

        // Stop using the temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(null);

        // Allow for caching all bean definition metadata, not expecting further changes.
        beanFactory.freezeConfiguration();

        // Instantiate all remaining (non-lazy-init) singletons.
        // 实例化bean
        beanFactory.preInstantiateSingletons();
    }
    //DefaultListableBeanFactory
    public void preInstantiateSingletons() throws BeansException {
        if (logger.isTraceEnabled()) {
            logger.trace("Pre-instantiating singletons in " + this);
        }

        // Iterate over a copy to allow for init methods which in turn register new bean definitions.
        // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
        // 获取所有的bean定义名
        List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

        // Trigger initialization of all non-lazy singleton beans...
        //开始处理
        for (String beanName : beanNames) {
            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
            if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                //初始化FactoryBean类型
                if (isFactoryBean(beanName)) {
                    Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                    if (bean instanceof FactoryBean) {
                        final FactoryBean<?> factory = (FactoryBean<?>) bean;
                        boolean isEagerInit;
                        if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                            isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                            ((SmartFactoryBean<?>) factory)::isEagerInit,
                                    getAccessControlContext());
                        }
                        else {
                            isEagerInit = (factory instanceof SmartFactoryBean &&
                                    ((SmartFactoryBean<?>) factory).isEagerInit());
                        }
                        if (isEagerInit) {
                            getBean(beanName);
                        }
                    }
                }
                else {    //一般类型
                    getBean(beanName);
                }
            }
        }

        // Trigger post-initialization callback for all applicable beans...
        //若bean是SmartInitializingSingleton类型则调用
        for (String beanName : beanNames) {
            Object singletonInstance = getSingleton(beanName);
            if (singletonInstance instanceof SmartInitializingSingleton) {
                final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
                if (System.getSecurityManager() != null) {
                    AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }, getAccessControlContext());
                }
                else {
                    smartSingleton.afterSingletonsInstantiated();
                }
            }
        }
    }
{%endcodeblock%}
- finishRefresh:处理LifecycleProcessor,并且发布ContextRefreshedEvent事件
{%codeblock lang:java%}
protected void finishRefresh() {
        // Clear context-level resource caches (such as ASM metadata from scanning).
        clearResourceCaches();

        // Initialize lifecycle processor for this context.
        initLifecycleProcessor();

        // Propagate refresh to lifecycle processor first.
        getLifecycleProcessor().onRefresh();

        // Publish the final event.
        //发布事件
        publishEvent(new ContextRefreshedEvent(this));

        // Participate in LiveBeansView MBean, if active.
        //这实际上是一个jmx
        LiveBeansView.registerApplicationContext(this);
    }

    //-----------initLifecycleProcessor---------------
    protected void initLifecycleProcessor() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
            this.lifecycleProcessor =
                    beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
            if (logger.isTraceEnabled()) {
                logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
            }
        }
        else {
            DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor(); //默认的lifecycleProcessor
            defaultProcessor.setBeanFactory(beanFactory); //持有bf
            this.lifecycleProcessor = defaultProcessor;
            beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
            if (logger.isTraceEnabled()) {
                logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
                        "[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
            }
        }
    }
    //------------------DefaultLifecycleProcessor#onRefresh----------------
    public void onRefresh() {
        startBeans(true);
        this.running = true;
    }
    //总而言之就是启动lifeCycle bean的start()
    private void startBeans(boolean autoStartupOnly) {
        Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans(); //获取bf中LifeCycle的bean
        Map<Integer, LifecycleGroup> phases = new HashMap<>();
        lifecycleBeans.forEach((beanName, bean) -> {
            if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
                int phase = getPhase(bean);
                LifecycleGroup group = phases.get(phase);
                if (group == null) {
                    group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
                    phases.put(phase, group);
                }
                group.add(beanName, bean);
            }
        });
        if (!phases.isEmpty()) {
            List<Integer> keys = new ArrayList<>(phases.keySet());
            Collections.sort(keys);
            for (Integer key : keys) {
                phases.get(key).start();
            }
        }
    }
{%endcodeblock%}
使用注解的ioc
```java
package ioc.annotation;

import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.annotation.*;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

/**
 * 通用注解
 *
 * @see Component#value() 表示bean名称
 * @see Scope#value() 表示单例|非单例..
 * @see Scope#proxyMode()  表示该bean的代理模式  可以代理  jdk代理 cglib代理
 */
@Component("car1")
@Scope(value = "prototype")
public class MyCar {
    MyCar2 myCar2;

    /**
     * @see Autowired#required() 表示是否一定要注入
     * @see Qualifier#value() 表示注入bean的名称
     * 关于注值的过程是实例化后,注入pvs之前由
     * @see AutowiredAnnotationBeanPostProcessor#postProcessProperties(PropertyValues, Object, String) 调用过程,改变了pvs
     * 然后注入的
     */
    @Autowired
    @Qualifier("car2")
    public void setMyCar2(MyCar2 myCar2) {
        this.myCar2 = myCar2;
    }

    public MyCar2 getMyCar2() {
        return myCar2;
    }

    /**
     * 首先这两个注解是javax.annotation提供的jdk11中并没有
     * beanFactory调用这两个函数是在装配bean结束后
     *
     * @see InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization(Object, String)
     * 这种bean的init逻辑就没有用,被上边的这个处理器完成了
     * 以及beanFactory销毁bean时
     * @see InitDestroyAnnotationBeanPostProcessor#postProcessBeforeDestruction(Object, String)
     */
    @PostConstruct
    public void init() {
        System.out.println("init");
    }

    @PreDestroy
    public void des() {
        System.out.println("des");
    }

    @Value("3")
    int a;

    public int getA() {
        return a;
    }
}

```
##### 补充点
- 完成loadBeanDefition的都是context内部的reader,在reader的创建过程会产生特异点
     - AnnotationConfigApplicationContext
     ```java
     public AnnotationConfigApplicationContext() {
     this.reader = new AnnotatedBeanDefinitionReader(this);
     this.scanner = new ClassPathBeanDefinitionScanner(this);
 }
 //------------AnnotatedBeanDefinitionReader创建----------------
 public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
        this(registry, getOrCreateEnvironment(registry));
    }

    public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        Assert.notNull(environment, "Environment must not be null");
        this.registry = registry;
        this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
        //注意此处这个工具类主动提供了一些beanFactory处理器
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
     ```
