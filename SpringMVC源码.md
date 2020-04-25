---
title: SpringMVC体系
date: 2019-02-19 17:11:25
tags:
- 框架
- spring
- springMvc
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
##### Servlet部分概念复习
- ServletConfig 对应一个servlet的init para
```java
public interface ServletConfig {
    public String getServletName();
    public ServletContext getServletContext();
    public String getInitParameter(String name);
    public Enumeration<String> getInitParameterNames();
}
```
- ServletContext 则代表了这个web应用
- include 和 include
```java
public interface RequestDispatcher{
 public void forward(ServletRequest request, ServletResponse response)
        throws ServletException, IOException;

   public void include(ServletRequest request, ServletResponse response)
        throws ServletException, IOException;        
}
```
首先这两个api的作用都是进行'转发',前者的调用结果会导致,响应处理权限转交给调用的servlet;后者还是自己
如下例子
```java
public class X extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        //直接使用forward会导致后边的相应无效,因为被x2接管了
        req.getRequestDispatcher("/x2").forward(req, resp);
        resp.getOutputStream().write(1);
    }
}
```
##### DispatcherServlet
- 继承图
{%asset_img DispatcherSevlet.png%}
- 概述: 简而言之,既然这是一个servlet,查看的源码的顺序就按照servlet的生命周期进行;这个类的构造器仅仅设置了一个bool,并没有进行大量的初始化.
- 一个简单的xml配置
```xml
    <!-- 配置spring ioc容器 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!--springMVC-->
    <servlet>
        <servlet-name>springMVc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springMvc.xml</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>springMVc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
```
ioc容器初始化是通过ContextLoaderListener
###### 传统xml情况ioc容器创建过程
- ContextLoaderListener:Servlet监听器
{%codeblock lang:java ContextLoaderListener%}
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
  public void contextInitialized(ServletContextEvent event) {
          initWebApplicationContext(event.getServletContext());
      }
    //-------------------initWebApplicationContext--------------------
    public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
      if (this.context == null) {
        //创建context
                this.context = createWebApplicationContext(servletContext);
            }
      //初始化context,如到contextConfigLocation目录下设置xml源
            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
                if (!cwac.isActive()) {
                    // The context has not yet been refreshed -> provide services such as
                    // setting the parent context, setting the application context id, etc
                    if (cwac.getParent() == null) {
                        // The context instance was injected without an explicit parent ->
                        // determine parent for root web application context, if any.
                        ApplicationContext parent = loadParentContext(servletContext);
                        cwac.setParent(parent);
                    }
                    configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }
      //设置context到ServletContext中
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
    }  
    //-------------------createWebApplicationContext-------------------
    protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
        Class<?> contextClass = determineContextClass(sc);
        if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
            throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                    "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
        }
        return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
    }
  //-------------------determineContextClass---------------------------
  protected Class<?> determineContextClass(ServletContext servletContext) {
    //若用户没有指定则根据defaultStrategies配置的创建一个context
    //实际上在springFramework web模块定义了一个ContextLoader.properties
    //这里默认是XmlWebApplicationContext
        String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
        if (contextClassName != null) {
            try {
                return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load custom context class [" + contextClassName + "]", ex);
            }
        }
        else {
            contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
            try {
                return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load default context class [" + contextClassName + "]", ex);
            }
        }
    }
}
{%endcodeblock%}
###### 使用WebApplicationInitializer体系
- ServletContainerInitializer:Serlvet标准提供的让第三方框架用以加载的接口
  - 使用参考[ServletContainerInitializer](https://blog.csdn.net/wangyangzhizhou/article/details/52013779)
  - 原理ContextConfig:该类是tomcat是Context的配置监听器,由[Digester](/2018/10/06/tomcat/#server.xml对于java代码加载)加载
    {%codeblock lang:java ContextConfig%}
    //该函数会被监听触发是时调用,获取HandlesTypes注解的类
    protected void processServletContainerInitializers() {

      List<ServletContainerInitializer> detectedScis;
      try {
          WebappServiceLoader<ServletContainerInitializer> loader = new WebappServiceLoader<>(context);
          detectedScis = loader.load(ServletContainerInitializer.class);
      } catch (IOException e) {
          log.error(sm.getString(
                  "contextConfig.servletContainerInitializerFail",
                  context.getName()),
              e);
          ok = false;
          return;
      }

      for (ServletContainerInitializer sci : detectedScis) {
          initializerClassMap.put(sci, new HashSet<Class<?>>());

          HandlesTypes ht;
          try {
              ht = sci.getClass().getAnnotation(HandlesTypes.class);
          } catch (Exception e) {
              if (log.isDebugEnabled()) {
                  log.info(sm.getString("contextConfig.sci.debug",
                          sci.getClass().getName()),
                          e);
              } else {
                  log.info(sm.getString("contextConfig.sci.info",
                          sci.getClass().getName()));
              }
              continue;
          }
          if (ht == null) {
              continue;
          }
          Class<?>[] types = ht.value();
          if (types == null) {
              continue;
          }

          for (Class<?> type : types) {
              if (type.isAnnotation()) {
                  handlesTypesAnnotations = true;
              } else {
                  handlesTypesNonAnnotations = true;
              }
              Set<ServletContainerInitializer> scis =
                      typeInitializerMap.get(type);
              if (scis == null) {
                  scis = new HashSet<>();
                  typeInitializerMap.put(type, scis);
              }
              scis.add(sci);
          }
      }
  }
    {%endcodeblock%}
- SpringServletContainerInitializer:spring web实现的初始化框架接口
{%codeblock lang:java SpringServletContainerInitializer%}
public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
            throws ServletException {
    //调用WebApplicationInitializer接口    
        List<WebApplicationInitializer> initializers = new LinkedList<>();

        if (webAppInitializerClasses != null) {
            for (Class<?> waiClass : webAppInitializerClasses) {
                // Be defensive: Some servlet containers provide us with invalid classes,
                // no matter what @HandlesTypes says...
                if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
                        WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
                    try {
                        initializers.add((WebApplicationInitializer)
                                ReflectionUtils.accessibleConstructor(waiClass).newInstance());
                    }
                    catch (Throwable ex) {
                        throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
                    }
                }
            }
        }

        if (initializers.isEmpty()) {
            servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
            return;
        }

        servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
        AnnotationAwareOrderComparator.sort(initializers);
        for (WebApplicationInitializer initializer : initializers) {
            initializer.onStartup(servletContext);
        }
    }
{%endcodeblock%}   
- WebApplicationInitializer:spring提供的真正执行逻辑的初始化接口
  - AbstractAnnotationConfigDispatcherServletInitializer:
    {%codeblock lang:java AbstractAnnotationConfigDispatcherServletInitializer%}
    //ioc context 配置类,被父类调用创建ioc容器
    protected abstract Class<?>[] getRootConfigClasses();
    //web context 配置类,父类调用创建mvc容器
    protected abstract Class<?>[] getServletConfigClasses();
    //DispatchServlet 根路径
    protected String[] getServletMappings() {
       return new String[]{"/"};
   }
    {%endcodeblock%}
- 说明
  - @EnableMVC和纯使用注解无关,这个实际上引入了一个@Import,网上傻逼还是多
  - 使用@ComponentScan和@Configuration注解就够了,这俩个注解被ConfigrutaionPoss..处理,mvc中的Context都是Annotaion..Context默认带有这个处理器
  - RequestMapping处理@Controller的时候不会从ioc容器中遍历,因此对于mvc组件要放在mvc容器中
###### 初始化部分
{%codeblock lang:java 初始化部分%}

    //HttpServletBean#int()
    public final void init() throws ServletException {

        // 通过propery来设置this的属性,将web.xml中的配置信息,如init-para注入
        // 这几个类在ioc中是非常重要的
        PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
        if (!pvs.isEmpty()) {
            try {
                BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
                ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
                bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
                initBeanWrapper(bw);
                bw.setPropertyValues(pvs, true);
            }
            catch (BeansException ex) {
                if (logger.isErrorEnabled()) {
                    logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
                }
                throw ex;
            }
        }

        // Let subclasses do whatever initialization they like.
        initServletBean();
    }
    //FrameworkServlet#initServletBean()
        protected final void initServletBean() throws ServletException {
        getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
        if (logger.isInfoEnabled()) {
            logger.info("Initializing Servlet '" + getServletName() + "'");
        }
        long startTime = System.currentTimeMillis();

        try {
            //初始化容器
            this.webApplicationContext = initWebApplicationContext();
            //这个函数是空的,用户可以自定义实现
            initFrameworkServlet();
        }
        //...日志
    }

    //简而言之,寻找父类容器,即ioc注入容器|创建mvc容器,将前者作为parent
    protected WebApplicationContext initWebApplicationContext() {
        WebApplicationContext rootContext =
                WebApplicationContextUtils.getWebApplicationContext(getServletContext()); //ioc容器
        WebApplicationContext wac = null; //mvc容器

        if (this.webApplicationContext != null) {
            wac = this.webApplicationContext;
            if (wac instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
                if (!cwac.isActive()) {

                    if (cwac.getParent() == null) {

                        cwac.setParent(rootContext);
                    }
                    configureAndRefreshWebApplicationContext(cwac);
                }
            }
        }
        if (wac == null) {
            wac = findWebApplicationContext();
        }
        if (wac == null) {
            wac = createWebApplicationContext(rootContext);
        }

        if (!this.refreshEventReceived) {
            synchronized (this.onRefreshMonitor) {
                onRefresh(wac);//一般情况不会调用这里
            }
        }

        if (this.publishContext) {
            String attrName = getServletContextAttributeName();
            getServletContext().setAttribute(attrName, wac);
        }

        return wac; //最终返回mvc容器
    }
    protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
        if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
            // The application context id is still set to its original default value
            // -> assign a more useful id based on available information
            if (this.contextId != null) {
                wac.setId(this.contextId);
            }
            else {
                // Generate default id...
                wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                        ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
            }
        }

        wac.setServletContext(getServletContext());
        wac.setServletConfig(getServletConfig());
        wac.setNamespace(getNamespace());
        //注册监听器,调用点在 Context#Refrsh()中进行的,具体context refresh做了那些工作,参考spring ioc|aop部分
        wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

        // The wac environment's #initPropertySources will be called in any case when the context
        // is refreshed; do it eagerly here to ensure servlet property sources are in place for
        // use in any post-processing or initialization that occurs below prior to #refresh
        ConfigurableEnvironment env = wac.getEnvironment();
        if (env instanceof ConfigurableWebEnvironment) {
            ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
        }

        postProcessWebApplicationContext(wac);
        applyInitializers(wac);
        wac.refresh();
    }
    //初始化组件,此处被调用的原因是因为上边注册了一个事件监听器
    protected void onRefresh(ApplicationContext context) {
        initStrategies(context);
    }
    //具体组件分析再看这部分代码
    protected void initStrategies(ApplicationContext context) {
    //各组件的基本都是存在默认情况,若用户提供了则从容器中获取
        initMultipartResolver(context);  //处理上传文件
        initLocaleResolver(context);     //根据不同地区,相应不同视图
        initThemeResolver(context);      //主题风格
        initHandlerMappings(context);     //
        initHandlerAdapters(context);
        initHandlerExceptionResolvers(context);
        initRequestToViewNameTranslator(context);
        initViewResolvers(context);
        initFlashMapManager(context);
    }
{%endcodeblock%}
##### service逻辑
- 代码
{%codeblock lang:java DispatcherServlet%}
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        logRequest(request);

        //如果请求时include请求,则保存一份源request的数据
        Map<String, Object> attributesSnapshot = null;
        if (WebUtils.isIncludeRequest(request)) {
            attributesSnapshot = new HashMap<>();
            Enumeration<?> attrNames = request.getAttributeNames();
            while (attrNames.hasMoreElements()) {
                String attrName = (String) attrNames.nextElement();
                if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
                    attributesSnapshot.put(attrName, request.getAttribute(attrName));
                }
            }
        }

        //向当前请求添加一些必要的属性,如WebApplicationContext  localeResolver  themeResolver  getThemeSource
        request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
        request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
        request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
        request.setAttribute(THEME_SOURCE_ATTRIBUTE,  ());
        //此处用来处理重定向,通过该类来记录信息
        if (this.flashMapManager != null) {
            FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
            if (inputFlashMap != null) {
                request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
            }
            request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
            request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
        }

        try {
        //核心逻辑
            doDispatch(request, response);
        }
        finally {
            if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
                // Restore the original attribute snapshot, in case of an include.
                if (attributesSnapshot != null) {
                    restoreAttributesAfterInclude(request, attributesSnapshot);
                }
            }
        }
    }


    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null;
        boolean multipartRequestParsed = false;

        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

        try {
            ModelAndView mv = null;
            Exception dispatchException = null;

            try {
                //[1] 检查是否是Multipart
                processedRequest = checkMultipart(request);
                multipartRequestParsed = (processedRequest != request);

                // Determine handler for the current request.
        //[2] 获取chain
                mappedHandler = getHandler(processedRequest);
                if (mappedHandler == null) {
                    noHandlerFound(processedRequest, response);
                    return;
                }

                // Determine handler adapter for the current request.
        //[3] adaptor
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

                // Process last-modified header, if supported by the handler.
                String method = request.getMethod();
                boolean isGet = "GET".equals(method);
                if (isGet || "HEAD".equals(method)) {
                    long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                    if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                        return;
                    }
                }

                if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                    return;
                }

                // Actually invoke the handler.
        //[4] handl调用
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }

                applyDefaultViewName(processedRequest, mv);
                mappedHandler.applyPostHandle(processedRequest, response, mv);
            }
            catch (Exception ex) {
                dispatchException = ex;
            }
            catch (Throwable err) {
                // As of 4.3, we're processing Errors thrown from handler methods as well,
                // making them available for @ExceptionHandler methods and other scenarios.
                dispatchException = new NestedServletException("Handler dispatch failed", err);
            }
      //[5] 异常和视图处理
            processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
        }
        catch (Exception ex) {
            triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
        }
        catch (Throwable err) {
            triggerAfterCompletion(processedRequest, response, mappedHandler,
                    new NestedServletException("Handler processing failed", err));
        }
    //[6] 异步框架
        finally {
            if (asyncManager.isConcurrentHandlingStarted()) {
                // Instead of postHandle and afterCompletion
                if (mappedHandler != null) {
                    mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                }
            }
            else {
                // Clean up any resources used by a multipart request.
                if (multipartRequestParsed) {
                    cleanupMultipart(processedRequest);
                }
            }
        }
    }

  private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
            @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
            @Nullable Exception exception) throws Exception {

        boolean errorView = false;
    //[1] 异常处理机制
        if (exception != null) {
            if (exception instanceof ModelAndViewDefiningException) {
                logger.debug("ModelAndViewDefiningException encountered", exception);
                mv = ((ModelAndViewDefiningException) exception).getModelAndView();
            }
            else {
                Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
                mv = processHandlerException(request, response, handler, exception);
                errorView = (mv != null);
            }
        }

        // Did the handler return a view to render?
    //[2] 视图处理
        if (mv != null && !mv.wasCleared()) {
            render(mv, request, response);
            if (errorView) {
                WebUtils.clearErrorRequestAttributes(request);
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("No view rendering, null ModelAndView returned.");
            }
        }

        if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
            // Concurrent handling started during a forward
            return;
        }
    //[3] chain.conletion
        if (mappedHandler != null) {
            // Exception (if any) is already handled..
            mappedHandler.triggerAfterCompletion(request, response, null);
        }
    }

{%endcodeblock%}
- 逻辑
{%asset_img doService.png doService%}
{%asset_img doDispatcher.png doDispatcher%}
{%asset_img processDisRe.png processDisRe%}
##### 组件说明
###### MultipartResolver
- 作用:用来处理上传文件的解析器
{%codeblock lang:java%}
    //在doDispatch()中被调用
    protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
        if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
            if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
                if (request.getDispatcherType().equals(DispatcherType.REQUEST)) {
                    logger.trace("Request already resolved to MultipartHttpServletRequest, e.g. by MultipartFilter");
                }
            }
            else if (hasMultipartException(request)) {
                logger.debug("Multipart resolution previously failed for current request - " +
                        "skipping re-resolution for undisturbed error rendering");
            }
            else {
                try {
                    return this.multipartResolver.resolveMultipart(request);
                }
                catch (MultipartException ex) {
                    if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
                        logger.debug("Multipart resolution failed for error dispatch", ex);
                        // Keep processing error dispatch with regular request handle below
                    }
                    else {
                        throw ex;
                    }
                }
            }
        }
        // If not returned before: return original request.
        return request;
    }
    //MultipartResolver接口
    public interface MultipartResolver {
    //判断是否含有文件
    boolean isMultipart(HttpServletRequest request);
    //将servletRequest解析成MultipartHttpServletRequest()
    MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;
    }
{%endcodeblock%}
- 使用:
{%codeblock lang:xml%}
    <!--设置一个文件解析器-->
   <!--配置文件上传-->
    <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <!-- 设定默认编码 -->
        <property name="defaultEncoding" value="UTF-8"></property>
        <!-- 设定文件上传的最大值为5MB，5*1024*1024 -->
        <property name="maxUploadSize" value="5242880"></property>
        <!-- 设定文件上传时写入内存的最大值，如果小于这个参数不会生成临时文件，默认为10240 -->
        <property name="maxInMemorySize" value="40960"></property>
        <!-- 上传文件的临时路径 -->
        <!-- 延迟文件解析 -->
        <property name="resolveLazily" value="true"/>
    </bean>
   <!--html-->
   <form action="/fileUpload" method="post" enctype="multipart/form-data">
    <input type="file" name="file">
    <input type="submit" value="提交">
    </form>
{%endcodeblock%}
{%codeblock lang:java%}
  @RequestMapping(value = "/fileUpload", method = RequestMethod.POST)
    public void FileUpLoad(@RequestParam(value = "file", required = false) MultipartFile file, HttpServletRequest  request) {
        System.out.println(file);
    }
{%endcodeblock%}
- 说明
在post表单提交中enctype会成为context-type
###### HandlerMapping
######

##### 一些使用方法
- 开启json处理:注意如果返回bean,要带上set|get函数
{%codeblock lang:xml%}
 <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>${jacksonVersion}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jacksonVersion}</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>${jacksonVersion}</version>
        </dependency>
        <!--springMvc配置-->
  <mvc:annotation-driven/>
{%endcodeblock%}
###### 关于MVC配置
 - 使用xml配置一个拦截器
 ```xml
 <mvc:interceptors>
 <bean id="myHandlerInterceptor" class="com.light.MyInterceptor">
 </mvc>
 ```
 - 使用注解@EnableWebMvc
  ```java
  //jdk>=1.8
  @Configuration
  public MyMvcConfig implements WebMvcConfigurer{
    default void addInterceptors(InterceptorRegistry registry) {
      registry.addInterceptor(new MyInterceptor());
    }
  }
  ```
   - DelegatingWebMvcConfiguration:真正实现了注解配置MVC的类
   {%codeblock lang:java DelegatingWebMvcConfiguration%}
   //该类是一个bean
   @Configuration(proxyBeanMethods = false)
    public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
      private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

//将用户的WebMvcConfigurer注入
@Autowired(required = false)
public void setConfigurers(List<WebMvcConfigurer> configurers) {
  if (!CollectionUtils.isEmpty(configurers)) {
    this.configurers.addWebMvcConfigurers(configurers);
  }
}
  //--------------------WebMvcConfigurationSupport----------------------
  //这里就是对于RequestMappingHandlerMapping的创建和设计,此种的mapping最终会被FrameworkServlet获取到
  @Bean
  public RequestMappingHandlerMapping requestMappingHandlerMapping(
            @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
            @Qualifier("mvcConversionService") FormattingConversionService conversionService,
            @Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

        RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
        mapping.setOrder(0);
        mapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));
        mapping.setContentNegotiationManager(contentNegotiationManager);
        mapping.setCorsConfigurations(getCorsConfigurations());

        PathMatchConfigurer configurer = getPathMatchConfigurer();

        Boolean useSuffixPatternMatch = configurer.isUseSuffixPatternMatch();
        if (useSuffixPatternMatch != null) {
            mapping.setUseSuffixPatternMatch(useSuffixPatternMatch);
        }
        Boolean useRegisteredSuffixPatternMatch = configurer.isUseRegisteredSuffixPatternMatch();
        if (useRegisteredSuffixPatternMatch != null) {
            mapping.setUseRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch);
        }
        Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
        if (useTrailingSlashMatch != null) {
            mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
        }

        UrlPathHelper pathHelper = configurer.getUrlPathHelper();
        if (pathHelper != null) {
            mapping.setUrlPathHelper(pathHelper);
        }
        PathMatcher pathMatcher = configurer.getPathMatcher();
        if (pathMatcher != null) {
            mapping.setPathMatcher(pathMatcher);
        }
        Map<String, Predicate<Class<?>>> pathPrefixes = configurer.getPathPrefixes();
        if (pathPrefixes != null) {
            mapping.setPathPrefixes(pathPrefixes);
        }

        return mapping;
    }
    }
   {%endcodeblock%}
