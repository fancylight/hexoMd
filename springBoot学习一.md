---
title: springBoot学习一
date: 2019-09-05 09:40:16
tags:
- 框架
- spring
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
#### springBoot概述
##### 启动逻辑
- 构造器
{%codeblock lang:java SpringApplication%}
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));//实际就是调用#run(),传递的源
        this.webApplicationType = WebApplicationType.deduceFromClasspath();//判断web类型,如servlet
        setInitializers((Collection) getSpringFactoriesInstances(
                ApplicationContextInitializer.class)); //获取ApplicationContextInitializer子类,该子类实际上属于一种回调,设置springApplication中的回调
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));//获取监听器
        this.mainApplicationClass = deduceMainApplicationClass();//源
    }
//加载通过"META-INF/spring.factories"文件定义的类,参考SpringFactoriesLoader实现,逻辑并不是很难
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
        return getSpringFactoriesInstances(type, new Class<?>[] {});
    }
    /**
        * @para type 表示要创建的class类型
    * @para parametterTypes 该类型的构造器参数类型
    * @para args 实际参数
    */
    private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
            Class<?>[] parameterTypes, Object... args) {
        ClassLoader classLoader = getClassLoader();
        // Use names and ensure unique to protect against duplicates
        Set<String> names = new LinkedHashSet<>(
                SpringFactoriesLoader.loadFactoryNames(type, classLoader));
        List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
                classLoader, args, names);
        AnnotationAwareOrderComparator.sort(instances);
        return instances;
    }
{%endcodeblock%}
简单来说构造器完成了Initializer回调的设置和监听器的设置
- run整体逻辑
{%codeblock lang:java run%}
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();//该工具是spring的计时工具
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        configureHeadlessProperty();
        SpringApplicationRunListeners listeners = getRunListeners(args);//获取SpringApplicationRunListener,并执行这些监听器,观察者模式,见下文.
        listeners.starting();//执行starting相关监听器
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                    args);//封装入参
            ConfigurableEnvironment environment = prepareEnvironment(listeners,
                    applicationArguments);//处理环境
            //根据env中 spring.beaninfo.ignore来设置System.pre中的属性,具体这是干啥的,我不是很清楚
            configureIgnoreBeanInfo(environment);
            //这是一个花里胡哨的东西,用来打印控制台图案,定义banner.txt|banner.jpg|banner.gif试试
            Banner printedBanner = printBanner(environment);
            //根据app类型创建context,如AnnotationConfigServletWebServerApplicationContext
            context = createApplicationContext();
            //创建Reporters
            exceptionReporters = getSpringFactoriesInstances(
                    SpringBootExceptionReporter.class,
                    new Class[] { ConfigurableApplicationContext.class }, context);
            prepareContext(context, environment, listeners, applicationArguments,
                    printedBanner);
            refreshContext(context);
            afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass)
                        .logStarted(getApplicationLog(), stopWatch);
            }
            listeners.started(context);
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            listeners.running(context);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }
{%endcodeblock%}
- prepareEnvironment
{%codeblock lang:java 环境准备%}
private ConfigurableEnvironment prepareEnvironment(
            SpringApplicationRunListeners listeners,
            ApplicationArguments applicationArguments) {
        //创建ServletEnv
        ConfigurableEnvironment environment = getOrCreateEnvironment();
        configureEnvironment(environment, applicationArguments.getSourceArgs());
        listeners.environmentPrepared(environment);
        bindToSpringApplication(environment);
        if (!this.isCustomEnvironment) {
            environment = new EnvironmentConverter(getClassLoader())
                    .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
        }
        ConfigurationPropertySources.attach(environment);
        return environment;
    }
    protected void configureEnvironment(ConfigurableEnvironment environment,
            String[] args) {
        //类型转换器
        if (this.addConversionService) {
            ConversionService conversionService = ApplicationConversionService
                    .getSharedInstance();
            environment.setConversionService(
                    (ConfigurableConversionService) conversionService);
        }
        //将arg作为env中的属性加入
        configurePropertySources(environment, args);
        //设置Profile
        configureProfiles(environment, args);
    }
{%endcodeblock%}
- prepareContext
{%codeblock lang:java context准备%}
private void prepareContext(ConfigurableApplicationContext context,
            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
            ApplicationArguments applicationArguments, Banner printedBanner) {
        context.setEnvironment(environment);
        //添加了几个bean
        postProcessApplicationContext(context);
        //执行ApplicationInit
        applyInitializers(context);
        //发送contextPrepared事件
        listeners.contextPrepared(context);
        if (this.logStartupInfo) {
            logStartupInfo(context.getParent() == null);
            logStartupProfileInfo(context);
        }
        //添加参数bean和banner
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
        if (printedBanner != null) {
            beanFactory.registerSingleton("springBootBanner", printedBanner);
        }
        if (beanFactory instanceof DefaultListableBeanFactory) {
            ((DefaultListableBeanFactory) beanFactory)
                    .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
        //加载源
        Set<Object> sources = getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        load(context, sources.toArray(new Object[0]));
        //执行ApplicationPreparedEvent事件
        listeners.contextLoaded(context);
    }
protected void load(ApplicationContext context, Object[] sources) {
        if (logger.isDebugEnabled()) {
            logger.debug(
                    "Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
        }
        BeanDefinitionLoader loader = createBeanDefinitionLoader(
                getBeanDefinitionRegistry(context), sources);
        if (this.beanNameGenerator != null) {
            loader.setBeanNameGenerator(this.beanNameGenerator);
        }
        if (this.resourceLoader != null) {
            loader.setResourceLoader(this.resourceLoader);
        }
        if (this.environment != null) {
            loader.setEnvironment(this.environment);
        }
        loader.load();
    }
//省略...
//BeanDefinitionLoader#load
private int load(Class<?> source) {
        if (isGroovyPresent()
                && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
            // Any GroovyLoaders added in beans{} DSL can contribute beans here
            GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source,
                    GroovyBeanDefinitionSource.class);
            load(loader);
        }
        if (isComponent(source)) {//不使用groovy的情况,这里通过递归判断source本身是否声明Component注解||其注解中是否有声明Compoent||其接口||父类
            //如果符合条件,则通过注解reader将该bean设置为BeanFacotry#Registry中,等待实例化
            //此处还应该注意的点是,该reader将符合条件的源,转为了AnnotatedGenericBeanDefinition,供Config..Precessor解析时处理
            this.annotatedReader.register(source);
            return 1;
        }
        return 0;
    }
{%endcodeblock%}
##### spring中机制
###### spring中事件概述
{%codeblock lang:java 事件%}
//典型的观察者模式,由事件源发起事件,然后传递事件给下属监听器,以springBoot中SpringApplicationRunListeners部分为例子
//
class SpringApplicationRunListeners {
private final List<SpringApplicationRunListener> listeners;
public void starting(){
//其中的属性都是由SpringFactoriesLoader获取的子类,默认情况下只有EventPublishingRunListener
for (SpringApplicationRunListener listener : this.listeners) {
            listener.starting();//
        }
}
}
//事件监听器
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

    private final SpringApplication application;

    private final String[] args;

    private final SimpleApplicationEventMulticaster initialMulticaster;

    public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;//反射构造时指定的SpringApplication对象
        this.args = args;//该参数便是run(arg)
        this.initialMulticaster = new SimpleApplicationEventMulticaster();//spring默认的事件广播器
        for (ApplicationListener<?> listener : application.getListeners()) {
            this.initialMulticaster.addApplicationListener(listener);//这样就application构造时创建的监听器加入到了广播器中
        }
    }
    @Override
    public void starting() {
        //传播事件
        this.initialMulticaster.multicastEvent(
                new ApplicationStartingEvent(this.application, this.args));
    }
}
//caster
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
@Override
    public void multicastEvent(ApplicationEvent event) {
        multicastEvent(event, resolveDefaultEventType(event));
    }

    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            //这里的逻辑就是从之前创建caster时放置的listener中取出能够支持这种事件类型的监听器
            Executor executor = getTaskExecutor();
            if (executor != null) {
                executor.execute(() -> invokeListener(listener, event));
            }
            else {
                invokeListener(listener, event);
            }
        }
    }
}

{%endcodeblock%}
springApp整个过程经历的监听器:
- start阶段
    - LoggingApplicationListener:用来初始化日志系统
    - BackgroundPreinitializer:通过设置system中IGNORE_BACKGROUNDPREINITIALIZER_PROPERTY_NAME为true可以屏蔽提前初始化
    - DelegatingApplicationListener:用来委托其他监听器的监听器,在starting时没有作用
    - LiquibaseServiceLocatorApplicationListener:liqiubase是一个数据库工具,该监听器会在使用者使用了该库时替换掉serviceLoader
- environmentPrepared阶段:进行环境设置后调用
    {%asset_img envedLis.png%}
    - ConfigFileApplicationListener:从`classpath:/,classpath:/config/,file:./,file:./config/`加载`application.properties`或`application.yml`到env中
    - AnsiOutputApplicationListener:根据`pring.output.ansi.enabled`,决定是否开启AnsiOutput
    - LoggingApplicationListener
    - ...
- contextPrepared阶段:完成initializers调用后进行
    {%asset_img contextLis.png%}
    - 前者不做事情
- contextLoaded阶段:完成source注册后启动
    {%asset_img loadedLis.png%}
###### Environment相关
{%asset_img StanderSE.png%}
- PropertyResolver:spring提供的访问属性接口
- Environment:表示环境接口,Profiles(配置文件)和Properties.
- AbstractEnvironment: 该类中的propertySources则真正缓存着环境属性
{%codeblock lang:java AbstractEnvironment%}
public abstract class AbstractEnvironment implements ConfigurableEnvironment {

    public static final String IGNORE_GETENV_PROPERTY_NAME = "spring.getenv.ignore";

    public static final String ACTIVE_PROFILES_PROPERTY_NAME = "spring.profiles.active";

    public static final String DEFAULT_PROFILES_PROPERTY_NAME = "spring.profiles.default";


    protected final Log logger = LogFactory.getLog(getClass());

    private final Set<String> activeProfiles = new LinkedHashSet<>();

    private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());

    //该对象实际缓存着多个PropertySource
    private final MutablePropertySources propertySources = new MutablePropertySources();
        //通过该接口进行读取属性
    private final ConfigurablePropertyResolver propertyResolver =
            new PropertySourcesPropertyResolver(this.propertySources);

    //构造env对象时进行加载属性
    public AbstractEnvironment() {
        customizePropertySources(this.propertySources);
    }
    //父类为空,显而易见将不同子类要初始化的env属性添加到propertySources中
        protected void customizePropertySources(MutablePropertySources propertySources) {
    }
//子类
public class StandardEnvironment extends AbstractEnvironment {

    /** System environment property source name: {@value}. */
    public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

    /** JVM system properties property source name: {@value}. */
    public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";

    //子类进行的初始化工作
    @Override
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));//获取System中proprety
        propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));//System中env
    }

}
//同理serverlt
public class StandardServletEnvironment extends StandardEnvironment implements ConfigurableWebEnvironment {

    /** Servlet context init parameters property source name: {@value}. */
    public static final String SERVLET_CONTEXT_PROPERTY_SOURCE_NAME = "servletContextInitParams";

    /** Servlet config init parameters property source name: {@value}. */
    public static final String SERVLET_CONFIG_PROPERTY_SOURCE_NAME = "servletConfigInitParams";

    /** JNDI property source name: {@value}. */
        //添加servlet相关的环境
    public static final String JNDI_PROPERTY_SOURCE_NAME = "jndiProperties";
        protected void customizePropertySources(MutablePropertySources propertySources) {
        //这两种类型的PropretySource说明该环境不能在初始化时获取,因此作为占位符
        propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
        propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
        if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
            //
            propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
        }
        super.customizePropertySources(propertySources);
    }
    //该函数会被springMVC的FrameServlet调用,此时才设置有关于serlvet的环境变量
    @Override
    public void initPropertySources(@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
        WebApplicationContextUtils.initServletPropertySources(getPropertySources(), servletContext, servletConfig);
    }
}
//假设springMVC进行初始化时,会调到此处
public static void initServletPropertySources(MutablePropertySources sources,
            @Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {

        Assert.notNull(sources, "'propertySources' must not be null");
        String name = StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME;
        if (servletContext != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
            //这里逻辑很简单,仅仅是替换掉初始化时的占位符,缓存servletContext
            sources.replace(name, new ServletContextPropertySource(name, servletContext));
        }
        name = StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME;
        if (servletConfig != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
            //同上,缓存servletConfig
            sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
        }
    }
//关于PropertySource
public abstract class PropertySource<T> {

    protected final Log logger = LogFactory.getLog(getClass());

    protected final String name;
    //该有可能是System.Properties System.env等
    protected final T source;


    /**
     * Create a new {@code PropertySource} with the given name and source object.
     */
    public PropertySource(String name, T source) {
        Assert.hasText(name, "Property source name must contain at least one character");
        Assert.notNull(source, "Property source must not be null");
        this.name = name;
        this.source = source;
    }
}
{%endcodeblock%}
###### ApplicationContextInitializer
这些类是在context#refreshed之前调用,在SpringApplication中有一下实例:
- DelegatingApplicationContextInitializer:由名字也能知道,该类从env中获取context.initializer.classes定义的Initializer对象,再调用它们的init函数
- SharedMetadataReaderFactoryContextInitializer:向context中添加了一个BeanFacotryPost处理器,CachingMetadataReaderFactoryPostProcessor,context#fresh阶段调用
- ContextIdApplicationContextInitializer:向context设置了一个ContextId对象,并设置了该bean
- ConfigurationWarningsApplicationContextInitializer:添加ConfigurationWarningsPostProcessor处理器
- ServerPortInfoApplicationContextInitializer:该init也是一个ApplicationListener,用来处理WebServerInitializedEvent,设置web prot信息
- ConditionEvaluationReportLoggingListener:该类和上类似,用来设置一个appListener,输出一些报告
###### boot中context
AnnotationConfigServletWebServerApplicationContext的特有逻辑
- 构造
{%codeblock lang:java AnnotationConfigServletWebServerApplicationContext%}
    public AnnotationConfigServletWebServerApplicationContext() {
        //这里构造给此种类型的context附加了很多def和post
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }
    //AnnotatedBeanDefinitionReader
    public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
        this(registry, getOrCreateEnvironment(registry));
    }
    public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        Assert.notNull(environment, "Environment must not be null");
        this.registry = registry;
        this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
    //AnnotationConfigUtils
    public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
        registerAnnotationConfigProcessors(registry, null);
    }
    public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
            BeanDefinitionRegistry registry, @Nullable Object source) {

        DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
        if (beanFactory != null) {
            if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
                beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
            }
            if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
                beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
            }
        }

        Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
        // ConfigurationClassPostProcessor用来处理config注解的factoryPost处理器,调用时机在
        //AbstractApplicationContext #invokeBeanFactoryPostProcessors()
        if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
            def.setSource(source);
            beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
        }
        //BeanPost子类,AutowiredAnnotationBeanPostProcessor,ioc逻辑
        if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
            def.setSource(source);
            beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
        }

        // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
        // CommonAnnotationBeanPostProcessor,用来实现JSR-250,详见 javax-annotation-api包
        if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
            def.setSource(source);
            beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
        }

        // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
        // JPA支持,PersistenceAnnotationBeanPostProcessor,属于spring... . orm 包下
        if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition();
            try {
                def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                        AnnotationConfigUtils.class.getClassLoader()));
            }
            catch (ClassNotFoundException ex) {
                throw new IllegalStateException(
                        "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
            }
            def.setSource(source);
            beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
        }
        //这两个暂时不看
        if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
            def.setSource(source);
            beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
        }

        if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
            def.setSource(source);
            beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
        }

        return beanDefs;
    }
    //将bean注册到registry,并没有分类成不同的处理器,分类交给了context自己去实现
    private static BeanDefinitionHolder registerPostProcessor(
            BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

        definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        registry.registerBeanDefinition(beanName, definition);
        return new BeanDefinitionHolder(definition, beanName);
    }
{%endcodeblock%}
经过以上步骤,该context可能创建一下几个bean定义:
- ConfigurationClassPostProcessor
    {%asset_img ConfigurationClassPostProcessor.png%}
    - ioc创建实例后调用aware接口
        - context#fresh时,执行registry和factoryPost逻辑
    {%codeblock lang:java ConfigurationClassPostProcessor%}
    //BeanDefinitionRegistryPostProcessor逻辑,完成了@Configurable注解类的解析处理
    /**
     * Derive further bean definitions from the configuration classes in the registry.
     */
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        int registryId = System.identityHashCode(registry);
        if (this.registriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException(
                    "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
        }
        if (this.factoriesPostProcessed.contains(registryId)) {
            throw new IllegalStateException(
                    "postProcessBeanFactory already called on this post-processor against " + registry);
        }
        this.registriesPostProcessed.add(registryId);

        processConfigBeanDefinitions(registry);
    }
    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
        String[] candidateNames = registry.getBeanDefinitionNames();

        for (String beanName : candidateNames) {
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);

            if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                    ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
            //如果这个bean中有一个属性org.springframework.context.annotation.ConfigurationClassPostProcessor.configurationClass
            //被设置为full||lite,那么说明给bean被处理过,则不进行处理
                if (logger.isDebugEnabled()) {
                    logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
                }
            }
            //否则检查该bean是否@Configuration或者@Component,也就是说@Bean并发一定要写在Configuration类中
            else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
            }
        }

        // Return immediately if no @Configuration classes were found
        if (configCandidates.isEmpty()) {
            return;
        }

        // Sort by previously determined @Order value, if applicable
        configCandidates.sort((bd1, bd2) -> {
            int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
            int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
            return Integer.compare(i1, i2);
        });

        // Detect any custom bean name generation strategy supplied through the enclosing application context
        SingletonBeanRegistry sbr = null;
        if (registry instanceof SingletonBeanRegistry) {
            sbr = (SingletonBeanRegistry) registry;
            if (!this.localBeanNameGeneratorSet) {
                BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
                if (generator != null) {
                    this.componentScanBeanNameGenerator = generator;
                    this.importBeanNameGenerator = generator;
                }
            }
        }

        if (this.environment == null) {
            this.environment = new StandardEnvironment();
        }

        // Parse each @Configuration class
        //解析regsitry中包含的Configuration类
        ConfigurationClassParser parser = new ConfigurationClassParser(
                this.metadataReaderFactory, this.problemReporter, this.environment,
                this.resourceLoader, this.componentScanBeanNameGenerator, registry);

        //表示要处理的配置类
        Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
        //已经解析过的类
        Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
        do {
            //使用解析器进行解析,
            parser.parse(candidates);
            parser.validate();
            //获取进行parser处理的classes
            Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
            configClasses.removeAll(alreadyParsed);

            // Read the model and create bean definitions based on its content
            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                        registry, this.sourceExtractor, this.resourceLoader, this.environment,
                        this.importBeanNameGenerator, parser.getImportRegistry());
            }
            //载入beanDef,至此spring将用户定义的能够扫描的所有bean都注册;
            //将@Bean注解的方法设置为ConfigClass中的一个属性,交由下边这个reader进行处理
            // reader类型为ConfigurationClassBeanDefinitionReader,单独的类,不属于任何继承树
            this.reader.loadBeanDefinitions(configClasses);
            //设置已经处理
            alreadyParsed.addAll(configClasses);
            //清空备选
            candidates.clear();
            if (registry.getBeanDefinitionCount() > candidateNames.length) {
                //再次获取一次names,判断是否要继续进行一次
                String[] newCandidateNames = registry.getBeanDefinitionNames();
                Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
                Set<String> alreadyParsedClasses = new HashSet<>();
                for (ConfigurationClass configurationClass : alreadyParsed) {
                    alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
                }
                for (String candidateName : newCandidateNames) {
                    if (!oldCandidateNames.contains(candidateName)) {
                        BeanDefinition bd = registry.getBeanDefinition(candidateName);
                        if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                                !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                            candidates.add(new BeanDefinitionHolder(bd, candidateName));
                        }
                    }
                }
                candidateNames = newCandidateNames;
            }
        }
        while (!candidates.isEmpty());

        // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
        if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
            sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
        }

        if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
            // Clear cache in externally provided MetadataReaderFactory; this is a no-op
            // for a shared cache since it'll be cleared by the ApplicationContext.
            ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
        }
    }
    {%endcodeblock%}
    - ConfigurationClassParser逻辑
    {%codeblock lang:java ConfigurationClassParser%}
    //该类是用来解析@Configuration注解类的入口类
    public void parse(Set<BeanDefinitionHolder> configCandidates) {
        for (BeanDefinitionHolder holder : configCandidates) {
            BeanDefinition bd = holder.getBeanDefinition();
            try { //def的类型选择解析的方式
                if (bd instanceof AnnotatedBeanDefinition) {
                    parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
                }
                else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                    parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
                }
                else {
                    parse(bd.getBeanClassName(), holder.getBeanName());
                }
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
            }
        }

        this.deferredImportSelectorHandler.process();
    }
    //根据不同beanDef创建不同的ConfigClass
    protected final void parse(@Nullable String className, String beanName) throws IOException {
        Assert.notNull(className, "No bean class name for configuration class bean definition");
        MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
        processConfigurationClass(new ConfigurationClass(reader, beanName));
    }

    protected final void parse(Class<?> clazz, String beanName) throws IOException {
        processConfigurationClass(new ConfigurationClass(clazz, beanName));
    }

    protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
        processConfigurationClass(new ConfigurationClass(metadata, beanName));
    }
    protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
        if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
            return;
        }

        //configurationClasses是一个map,每次调用前获取给class是否之前处理过,处理import逻辑
        ConfigurationClass existingClass = this.configurationClasses.get(configClass);
        if (existingClass != null) {
            if (configClass.isImported()) {
                if (existingClass.isImported()) {
                    existingClass.mergeImportedBy(configClass);
                }
                // Otherwise ignore new imported config class; existing non-imported class overrides it.
                return;
            }
            else {
                // Explicit bean definition found, probably replacing an import.
                // Let's remove the old one and go with the new one.
                this.configurationClasses.remove(configClass);
                this.knownSuperclasses.values().removeIf(configClass::equals);
            }
        }

        // Recursively process the configuration class and its superclass hierarchy.
        //递归的处理该配置类其父类,当srouce==null,表示处理干净了
        SourceClass sourceClass = asSourceClass(configClass);
        do {
            sourceClass = doProcessConfigurationClass(configClass, sourceClass);
        }
        while (sourceClass != null);

        this.configurationClasses.put(configClass, configClass);
    }
    protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
            throws IOException {

        if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
            // Recursively process any member (nested) classes first
            //如果该配置是Component注解类,则开始处理内部类
            processMemberClasses(configClass, sourceClass);
        }

        // Process any @PropertySource annotations
        // 处理该类的PropertySource注解,将资源注入到env中,使用在ioc中
        for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
                sourceClass.getMetadata(), PropertySources.class,
                org.springframework.context.annotation.PropertySource.class)) {
            if (this.environment instanceof ConfigurableEnvironment) {
                processPropertySource(propertySource);
            }
            else {
                logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                        "]. Reason: Environment must implement ConfigurableEnvironment");
            }
        }

        // Process any @ComponentScan annotations
        // 处理@ComponentScan注解
        // [1] AnnotationAttributes是一个map,用来封装注解信息,并附带类型转换方法
        // [2] attributesForRepeatable方法用来获取注解信息
        // [3] componentScanParser真正用来处理
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
                sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
        if (!componentScans.isEmpty() &&
                !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
            for (AnnotationAttributes componentScan : componentScans) {
                // The config class is annotated with @ComponentScan -> perform the scan immediately
                Set<BeanDefinitionHolder> scannedBeanDefinitions =
                        this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
                // Check the set of scanned definitions for any further config classes and parse recursively if needed
                // 将通过Scan获取的def再进行一次处理
                for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                    BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                    if (bdCand == null) {
                        bdCand = holder.getBeanDefinition();
                    }
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }

        // Process any @Import annotations
        // @Import注解相当于将一个bean,引入配置
        processImports(configClass, sourceClass, getImports(sourceClass), true);

        // Process any @ImportResource annotations
        // 处理是否存在ImportResource注解,并使用对应的reader来处理,默认为BeanDefinitionReader
        // 此时仅仅将该resource和reader放置到configClass中,并没有处理
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            for (String resource : resources) {
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        }

        // Process individual @Bean methods
        // 获取@Bean定义的函数类型,在这个过程中并没有将@Bean注解的方法注册进去
        Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
        for (MethodMetadata methodMetadata : beanMethods) {
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }

        // Process default methods on interfaces
        // 获取配置类的接口中,定义为default的@Bean函数
        processInterfaces(configClass, sourceClass);

        // Process superclass, if any
        // 如果有父类则处理父类,java是单向继承
        if (sourceClass.getMetadata().hasSuperClass()) {
            String superclass = sourceClass.getMetadata().getSuperClassName();
            if (superclass != null && !superclass.startsWith("java") &&
                    !this.knownSuperclasses.containsKey(superclass)) {
                this.knownSuperclasses.put(superclass, configClass);
                // Superclass found, return its annotation metadata and recurse
                return sourceClass.getSuperClass();
            }
        }

        // No superclass -> processing is complete
        //到这里说明处理
        return null;
    }
    /**
     * Register member (nested) classes that happen to be configuration classes themselves.
     * 处理配置类的内部类,是否符合条件的
     */
    private void processMemberClasses(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
        //获取内部类
        Collection<SourceClass> memberClasses = sourceClass.getMemberClasses();
        if (!memberClasses.isEmpty()) {
            List<SourceClass> candidates = new ArrayList<>(memberClasses.size());
            for (SourceClass memberClass : memberClasses) {
                //判断是否符合条件
                if (ConfigurationClassUtils.isConfigurationCandidate(memberClass.getMetadata()) &&
                        !memberClass.getMetadata().getClassName().equals(configClass.getMetadata().getClassName())) {
                    candidates.add(memberClass);
                }
            }
            OrderComparator.sort(candidates);
            for (SourceClass candidate : candidates) {
                if (this.importStack.contains(configClass)) {
                    this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
                }
                else {
                    this.importStack.push(configClass);
                    try {
                        //调用上边的函数进行处理
                        processConfigurationClass(candidate.asConfigClass(configClass));
                    }
                    finally {
                        this.importStack.pop();
                    }
                }
            }
        }
    }

    {%endcodeblock%}
    - ComponentScanAnnotationParser,扫描包,获取配置了@Component注解的类进行处理
    {%codeblock lang:java ComponentScanAnnotationParser%}
    //这部分代码什么简单,通过componentScan获取注解内部的属性,以设置ClassPathBeanDefinitionScanner的属性
    public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
        //此处useDefaultFilters属性,决定了后后边进行匹配时应该符合的类,默认匹配@Component和@ManagedBean
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
                componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);

        Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
        boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
        scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
                BeanUtils.instantiateClass(generatorClass));

        ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
        if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
            scanner.setScopedProxyMode(scopedProxyMode);
        }
        else {
            Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
            scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
        }

        scanner.setResourcePattern(componentScan.getString("resourcePattern"));

        for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
            for (TypeFilter typeFilter : typeFiltersFor(filter)) {
                scanner.addIncludeFilter(typeFilter);
            }
        }
        for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
            for (TypeFilter typeFilter : typeFiltersFor(filter)) {
                scanner.addExcludeFilter(typeFilter);
            }
        }

        boolean lazyInit = componentScan.getBoolean("lazyInit");
        if (lazyInit) {
            scanner.getBeanDefinitionDefaults().setLazyInit(true);
        }

        Set<String> basePackages = new LinkedHashSet<>();
        String[] basePackagesArray = componentScan.getStringArray("basePackages");
        for (String pkg : basePackagesArray) {
            String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
                    ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
            Collections.addAll(basePackages, tokenized);
        }
        for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
            basePackages.add(ClassUtils.getPackageName(clazz));
        }
        //如果注解默认没有设置扫描路径,那么扫描路径就是注解类的包以及子路径
        //如test.Demo,则扫描路径就是test包下
        if (basePackages.isEmpty()) {
            basePackages.add(ClassUtils.getPackageName(declaringClass));
        }

        scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
            @Override
            protected boolean matchClassName(String className) {
                return declaringClass.equals(className);
            }
        });
        //进行解析
        return scanner.doScan(StringUtils.toStringArray(basePackages));
    }

    //ClassPathBeanDefinitionScanner类
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        for (String basePackage : basePackages) {
            //寻找拥有Component注解的类,间接拥有也可以,如Service,Configuration等注解
            //jsr250 @ManagedBean
            //jsr330 @Named
            Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            for (BeanDefinition candidate : candidates) {
                //设置scope
                ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
                candidate.setScope(scopeMetadata.getScopeName());
                //使用命名器设置beanName
                String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
                //根据不同的def,使用不同方式处理,并注册到resgity中,具体参考源码
                if (candidate instanceof AbstractBeanDefinition) {
                    postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
                }
                if (candidate instanceof AnnotatedBeanDefinition) {
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
                }
                if (checkCandidate(beanName, candidate)) {
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder =
                            AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    //注册
                    registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }
        return beanDefinitions;
    }
    //获取符合的类
    public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
            return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
        }
        else {
            return scanCandidateComponents(basePackage);
        }
    }
    private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
        Set<BeanDefinition> candidates = new LinkedHashSet<>();
        try {
            //使用Resource,拼装一个ant路径,如test包,会创建一个test/**/*.class,就能获取所有该包下的类
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX
                    resolveBasePackage(basePackage) + '/' + this.resourcePattern;
            Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
            boolean traceEnabled = logger.isTraceEnabled();
            boolean debugEnabled = logger.isDebugEnabled();
            for (Resource resource : resources) {
                if (traceEnabled) {
                    logger.trace("Scanning " + resource);
                }
                if (resource.isReadable()) {
                    try {
                        MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                        if (isCandidateComponent(metadataReader)) {
                            //ScannedGenericBeanDefinition(sbd)创建
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                            sbd.setResource(resource);
                            sbd.setSource(resource);
                            if (isCandidateComponent(sbd)) {
                                if (debugEnabled) {
                                    logger.debug("Identified candidate component class: " + resource);
                                }
                                candidates.add(sbd);
                            }
                            else {
                                if (debugEnabled) {
                                    logger.debug("Ignored because not a concrete top-level class: " + resource);
                                }
                            }
                        }
                        else {
                            if (traceEnabled) {
                                logger.trace("Ignored because not matching any filter: " + resource);
                            }
                        }
                    }
                    catch (Throwable ex) {
                        throw new BeanDefinitionStoreException(
                                "Failed to read candidate component class: " + resource, ex);
                    }
                }
                else {
                    if (traceEnabled) {
                        logger.trace("Ignored because not readable: " + resource);
                    }
                }
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
        }
        return candidates;
    }
    {%endcodeblock%}
    - @Import注解处理
    {%codeblock lang:java ConfigurationClassParser%}
    //关于@import注解有关的类,这些类关系不是很复杂,简单来说就是传递一些要加入的bean的信息
    //ImportSelector:用来返回要注册的bean的String
    //在ConfigurationClassParser有5个辅助用的内部类
    //DeferredImportSelectorHolder:包装ImportSelector和configurationClass
    //DeferredImportSelectorHandler:包装上者的list
    //DeferredImportSelectorGrouping:包装Group
    //DeferredImportSelectorGroupingHandler:包装上者list


    /*
     *  [1] ImportSelector类型进行hander,以获取更多的group
     *  [2] 一般类型则当做Config进行解析,实际上无论是否是Config类型,都会加入到Classes中等待注册
     *
         *  SpringBoot会提供一个AutoConfigurationImportSelector,它会将META下所有的spring.factory不分
     * type加入到group中,供后边进行处理
     */
    private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
            Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

        if (importCandidates.isEmpty()) {
            return;
        }

        if (checkForCircularImports && isChainedImportOnStack(configClass)) {
            this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
        }
        else {
            this.importStack.push(configClass);
            try {
                for (SourceClass candidate : importCandidates) {
                    if (candidate.isAssignable(ImportSelector.class)) {
                        // Selector类型
                        // Candidate class is an ImportSelector -> delegate to it to determine imports
                        Class<?> candidateClass = candidate.loadClass();

                        ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                        ParserStrategyUtils.invokeAwareMethods(
                                selector, this.environment, this.resourceLoader, this.registry);
                        if (selector instanceof DeferredImportSelector) {
                            //Deferred类型则加入到Handler中
                            this.deferredImportSelectorHandler.handle(
                                    configClass, (DeferredImportSelector) selector);
                        }
                        else {
                            //否则递归调用
                            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                            processImports(configClass, currentSourceClass, importSourceClasses, false);
                        }
                    }
                    else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                        // Candidate class is an ImportBeanDefinitionRegistrar ->
                        // delegate to it to register additional bean definitions
                        Class<?> candidateClass = candidate.loadClass();
                        ImportBeanDefinitionRegistrar registrar =
                                BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                        ParserStrategyUtils.invokeAwareMethods(
                                registrar, this.environment, this.resourceLoader, this.registry);
    `                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                    }
                    else {
                        // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                        // process it as an @Configuration class
                        // 注意看,如果该该类型与select无关,则直接把它当做配置类型用来,可以回去看处理Conf的代码,
                        // 最后一步将class加入到了ConfigClasses中,这个map最后会被注册
                        this.importStack.registerImport(
                                currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                        processConfigurationClass(candidate.asConfigClass(configClass));
                    }
                }
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to process import candidates for configuration class [" +
                        configClass.getMetadata().getClassName() + "]", ex);
            }
            finally {
                this.importStack.pop();
            }
        }
    }

    public void parse(Set<BeanDefinitionHolder> configCandidates) {
        for (BeanDefinitionHolder holder : configCandidates) {
            BeanDefinition bd = holder.getBeanDefinition();
            try {
                if (bd instanceof AnnotatedBeanDefinition) {
                    parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
                }
                else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                    parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
                }
                else {
                    parse(bd.getBeanClassName(), holder.getBeanName());
                }
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
            }
        }
        //当处理完一个Config类之后,开始处理Import
        this.deferredImportSelectorHandler.process();
    }
    //处理内部
    private class DeferredImportSelectorHandler {
    public void process() {
            List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
            this.deferredImportSelectors = null;
            try {
                if (deferredImports != null) {
                    DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
                    deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
                    //填充deferredImports
                    deferredImports.forEach(handler::register);
                    //
                    handler.processGroupImports();
                }
            }
            finally {
                this.deferredImportSelectors = new ArrayList<>();
            }
        }
    }
    private class DeferredImportSelectorGroupingHandler {
        public void processGroupImports() {
            for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
                //遍历group,调用getImport(),遍历其中的entry,
                grouping.getImports().forEach(entry -> {
                    ConfigurationClass configurationClass = this.configurationClasses.get(
                            entry.getMetadata());
                    try {
                        //将group#entry的每一组都去调用processImports逻辑
                        processImports(configurationClass, asSourceClass(configurationClass),
                                asSourceClasses(entry.getImportClassName()), false);
                    }
                    catch (BeanDefinitionStoreException ex) {
                        throw ex;
                    }
                    catch (Throwable ex) {
                        throw new BeanDefinitionStoreException(
                                "Failed to process import candidates for configuration class [" +
                                        configurationClass.getMetadata().getClassName() + "]", ex);
                    }
                });
            }
    }
    {%endcodeblock%}
    - ConfigurationClassBeanDefinitionReader 处理Classes
    {%codeblock lang:java ConfigurationClassBeanDefinitionReader%}
        //遍历目标
        public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
        TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
        for (ConfigurationClass configClass : configurationModel) {
            loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
        }
        //
        /**
         * Read a particular {@link ConfigurationClass}, registering bean definitions
         * for the class itself and all of its {@link Bean} methods.
         */
        private void loadBeanDefinitionsForConfigurationClass(
            ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

        if (trackedConditionEvaluator.shouldSkip(configClass)) {
            String beanName = configClass.getBeanName();
            if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
                this.registry.removeBeanDefinition(beanName);
            }
            this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
            return;
        }

        //处理import类型,并进行注册
        if (configClass.isImported()) {
            registerBeanDefinitionForImportedConfigurationClass(configClass);
        }
        //此处处理@Bean
        for (BeanMethod beanMethod : configClass.getBeanMethods()) {
            loadBeanDefinitionsForBeanMethod(beanMethod);
        }
        //加载@ImportResource
        loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
        loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
    }
    }
    {%endcodeblock%}
- AutowiredAnnotationBeanPostProcessor
- CommonAnnotationBeanPostProcessor
- PersistenceAnnotationBeanPostProcessor
- EventListenerMethodProcessor
- DefaultEventListenerFactory

##### 待整理内容
- JSR250:指javax.annation包中的注解,如postConstruct
- JSR330:指javax.inject包,用来用来替代ioc框架中的注解,以达到通配
- JPA:需要多个依赖,使用spring-boot-starter-data-jpa这个依赖就可以解决这些问题
- AliasFor注解:参考[AliasFor](https://www.jianshu.com/p/869ed7037833)
- 注解的实质:参考[注解是怎么实现的](https://www.zhihu.com/question/24401191)
- @Repeatable:jdk8新增的注解,实现单个注解重复使用
- JSR303:验证注解
