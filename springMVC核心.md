---
title: springMVC核心
date: 2019-12-26 12:50:38
tags:
- 框架
- spring
- springMvc
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
本文用来说明mvc中`mapping`和对应`adapter`组件
#### mapping
用来提供key-value的请求缓存视图,以及匹配url.
该对象用来初始化系统中的`handler`,以及用来匹配请求
{%asset_img Mapping.png%}
{%codeblock lang:java HandlerMapping%}
@Nullable
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
{%endcodeblock%}
- HandlerExecutionChain:用于调用拦截链,包含`Handler`
{%codeblock lang:java HandlerExecutionChain%}
public class HandlerExecutionChain {

    private static final Log logger = LogFactory.getLog(HandlerExecutionChain.class);
    //此处的handler没有使用统一接口
    private final Object handler;

    @Nullable
    private HandlerInterceptor[] interceptors;

    @Nullable
    private List<HandlerInterceptor> interceptorList;
{%endcodeblock%}
- Handler:实际上执行请求逻辑的组件
    - HttpRequestHandler
    - HandlerMethod
    - Servlet
    - Controller
- Mapping的作用和其子类
调用逻辑:
{%asset_img mapping调用逻辑.png%}
 实际上就两种类型`AbstractUrlHandlerMapping`和`AbstractHandlerMethodMapping`
{%asset_img Mapping子类.png%}
   ##### AbstractHandlerMapping
    初始化拦截器,实现普遍逻辑
  {%codeblock lang:java AbstractHandlerMapping%}
  public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport
        implements HandlerMapping, Ordered, BeanNameAware {

      //获取拦截链
      protected void initApplicationContext() throws BeansException {
        extendInterceptors(this.interceptors);
        detectMappedInterceptors(this.adaptedInterceptors);
        initInterceptors();
    }
  //核心逻辑
  public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    //该逻辑由子类实现
        Object handler = getHandlerInternal(request);
        if (handler == null) {
            handler = getDefaultHandler();
        }
        if (handler == null) {
            return null;
        }
        // Bean name or resolved handler?
        if (handler instanceof String) {
            String handlerName = (String) handler;
            handler = obtainApplicationContext().getBean(handlerName);
        }
    //此处为公共逻辑
        HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

        if (logger.isTraceEnabled()) {
            logger.trace("Mapped to " + handler);
        }
        else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
            logger.debug("Mapped to " + executionChain.getHandler());
        }

        if (CorsUtils.isCorsRequest(request)) {
            CorsConfiguration globalConfig = this.corsConfigurationSource.getCorsConfiguration(request);
            CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
            CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
            executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
        }

        return executionChain;
    }
    //------------------getHandlerExecutionChain--------------------
    protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
        //添加拦截器
        HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
                (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

        String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
        for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
            if (interceptor instanceof MappedInterceptor) {
                MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
                if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                    chain.addInterceptor(mappedInterceptor.getInterceptor());
                }
            }
            else {
                chain.addInterceptor(interceptor);
            }
        }
        return chain;
    }
  {%endcodeblock%}
 ##### AbstractUrlHandlerMapping类型
 该类型缓存key-value键值对,通过map来实现
 {%codeblock lang:java AbstractUrlHandlerMapping %}
 //表示缓存
 private final Map<String, Object> handlerMap = new LinkedHashMap<>();
 //当没有任何匹配时可以使用该handler
 private Object rootHandler;
 //该函数 供子类调用用于注册handler
 protected void registerHandler(String urlPath, Object handler)
 //url类型的配置逻辑
 protected Object getHandlerInternal(HttpServletRequest request) throws Exception {
        String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
        Object handler = lookupHandler(lookupPath, request);
      //省略
        return handler;
    }
    //配置handler
    protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
        // Direct match? 即缓存中存在
        Object handler = this.handlerMap.get(urlPath);
        if (handler != null) { //若为string类型则从bean容器中获取,并做一次处理
            // Bean name or resolved handler?
            if (handler instanceof String) {
                String handlerName = (String) handler;
                handler = obtainApplicationContext().getBean(handlerName);
            }
            validateHandler(handler, request);
            return buildPathExposingHandler(handler, urlPath, urlPath, null);
        }

        // Pattern match? 即最初缓存的可能是 带有统配符模式的url路径
        List<String> matchingPatterns = new ArrayList<>();
        for (String registeredPattern : this.handlerMap.keySet()) {
            if (getPathMatcher().match(registeredPattern, urlPath)) {
                matchingPatterns.add(registeredPattern);
            }
            else if (useTrailingSlashMatch()) {
                if (!registeredPattern.endsWith("/") && getPathMatcher().match(registeredPattern + "/", urlPath)) {
                    matchingPatterns.add(registeredPattern +"/");
                }
            }
        }

        String bestMatch = null;
        Comparator<String> patternComparator = getPathMatcher().getPatternComparator(urlPath);
        if (!matchingPatterns.isEmpty()) {
            matchingPatterns.sort(patternComparator);
            if (logger.isTraceEnabled() && matchingPatterns.size() > 1) {
                logger.trace("Matching patterns " + matchingPatterns);
            }
            bestMatch = matchingPatterns.get(0);
        }
        if (bestMatch != null) {
            //此处逻辑和直接直接相同
            handler = this.handlerMap.get(bestMatch);
            if (handler == null) {
                if (bestMatch.endsWith("/")) {
                    handler = this.handlerMap.get(bestMatch.substring(0, bestMatch.length() - 1));
                }
                if (handler == null) {
                    throw new IllegalStateException(
                            "Could not find handler for best pattern match [" + bestMatch + "]");
                }
            }
            // Bean name or resolved handler?
            if (handler instanceof String) {
                String handlerName = (String) handler;
                handler = obtainApplicationContext().getBean(handlerName);
            }
            validateHandler(handler, request);
            String pathWithinMapping = getPathMatcher().extractPathWithinPattern(bestMatch, urlPath);

            // There might be multiple 'best patterns', let's make sure we have the correct URI template variables
            // for all of them
            Map<String, String> uriTemplateVariables = new LinkedHashMap<>();
            for (String matchingPattern : matchingPatterns) {
                if (patternComparator.compare(bestMatch, matchingPattern) == 0) {
                    Map<String, String> vars = getPathMatcher().extractUriTemplateVariables(matchingPattern, urlPath);
                    Map<String, String> decodedVars = getUrlPathHelper().decodePathVariables(request, vars);
                    uriTemplateVariables.putAll(decodedVars);
                }
            }
            if (logger.isTraceEnabled() && uriTemplateVariables.size() > 0) {
                logger.trace("URI variables " + uriTemplateVariables);
            }
            return buildPathExposingHandler(handler, bestMatch, pathWithinMapping, uriTemplateVariables);
        }

        // No handler found...
        return null;
    }
 {%endcodeblock%}
 - SimpleUrlHandlerMapping
  {%codeblock lang:java SimpleUrlHandlerMapping%}
        //@Setter|Getter,使用者如果要使用该`mapping`,则需要手动添加该map
        private final Map<String, Object> urlMap = new LinkedHashMap<>();
        //初始化
        @Override
    public void initApplicationContext() throws BeansException {
        super.initApplicationContext();
        registerHandlers(this.urlMap);
    }
    protected void registerHandlers(Map<String, Object> urlMap) throws BeansException {
        if (urlMap.isEmpty()) {
            logger.trace("No patterns in " + formatMappingName());
        }
        else {
            urlMap.forEach((url, handler) -> {
                // Prepend with slash if not already present.
                if (!url.startsWith("/")) {
                    url = "/" + url;
                }
                // Remove whitespace from handler bean name.
                if (handler instanceof String) {
                    handler = ((String) handler).trim();
                }
                //调用父类注册函数
                registerHandler(url, handler);
            });
            if (logger.isDebugEnabled()) {
                List<String> patterns = new ArrayList<>();
                if (getRootHandler() != null) {
                    patterns.add("/");
                }
                if (getDefaultHandler() != null) {
                    patterns.add("/**");
                }
                patterns.addAll(getHandlerMap().keySet());
                logger.debug("Patterns " + patterns + " in " + formatMappingName());
            }
        }
    }
    {%endcodeblock%}
举个使用的例子,casServer的用法
{%codeblock lang:java CasWebAppConfiguration%}
public class CasWebAppConfiguration implements WebMvcConfigurer {
    @Bean
     protected Controller rootController() {
             return new ParameterizableViewController() {
                     @Override
                     protected ModelAndView handleRequestInternal(final HttpServletRequest request,
                                                                                                                final HttpServletResponse response) {
                             val queryString = request.getQueryString();
                             val url = request.getContextPath() + "/login"
                                     + Optional.ofNullable(queryString).map(string -> '?' + string).orElse(StringUtils.EMPTY);
                             return new ModelAndView(new RedirectView(response.encodeURL(url)));
                     }

             };
     }

     @Bean
     @Lazy
     public SimpleUrlHandlerMapping handlerMapping() {
             val mapping = new SimpleUrlHandlerMapping();

             val root = rootController();
             mapping.setOrder(1);
             mapping.setAlwaysUseFullPath(true);
             mapping.setRootHandler(root);
             val urls = new HashMap<String, Object>();
             urls.put("/", root);

             mapping.setUrlMap(urls);
             return mapping;
     }
}
{%endcodeblock%}
- BeanNameUrlHandlerMapping
{%codeblock lang:java BeanNameUrlHandlerMapping%}
public abstract class AbstractDetectingUrlHandlerMapping extends AbstractUrlHandlerMapping {
    public void initApplicationContext() throws ApplicationContextException {
        super.initApplicationContext();
        detectHandlers();
    }
    //
    protected void detectHandlers() throws BeansException {
        //获取ioc容器中的bean
    ApplicationContext applicationContext = obtainApplicationContext();
    String[] beanNames = (this.detectHandlersInAncestorContexts ?
            BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class) :
            applicationContext.getBeanNamesForType(Object.class));

    // Take any bean name that we can determine URLs for.
    for (String beanName : beanNames) {
        String[] urls = determineUrlsForHandler(beanName);
        if (!ObjectUtils.isEmpty(urls)) {
            // URL paths found: Let's consider it a handler.
            registerHandler(urls, beanName); //注册
        }
    }

    if ((logger.isDebugEnabled() && !getHandlerMap().isEmpty()) || logger.isTraceEnabled()) {
        logger.debug("Detected " + getHandlerMap().size() + " mappings in " + formatMappingName());
    }
}
}

public class BeanNameUrlHandlerMapping extends AbstractDetectingUrlHandlerMapping {
    //将以"/"开头的bean注册
    @Override
    protected String[] determineUrlsForHandler(String beanName) {
        List<String> urls = new ArrayList<>();
        if (beanName.startsWith("/")) {
            urls.add(beanName);
        }
        String[] aliases = obtainApplicationContext().getAliases(beanName);
        for (String alias : aliases) {
            if (alias.startsWith("/")) {
                urls.add(alias);
            }
        }
        return StringUtils.toStringArray(urls);
    }

}

{%endcodeblock%}
##### AbstractHandlerMethodMapping
该子类用于将Controller中的函数转换成HandlerMethod对象
{%codeblock lang:java%}
//泛型T表示每一个方法的映射对象
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
    //映射缓存
    private final MappingRegistry mappingRegistry = new MappingRegistry();
    //初始化
    public void afterPropertiesSet() {
        initHandlerMethods();
    }
    //----------------initHandlerMethods--------------------
    protected void initHandlerMethods() {
        for (String beanName : getCandidateBeanNames()) {
            if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                processCandidateBean(beanName);
            }
        }
        //打印数量
        handlerMethodsInitialized(getHandlerMethods());
    }
    //-------------processCandidateBean------------------
    protected void processCandidateBean(String beanName) {
        Class<?> beanType = null;
        try {
            beanType = obtainApplicationContext().getType(beanName);
        }
        catch (Throwable ex) {
            // An unresolvable bean type, probably from a lazy bean - let's ignore it.
            if (logger.isTraceEnabled()) {
                logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
            }
        }
        if (beanType != null && isHandler(beanType)) { //isHandler由子类实现,如RequestMappingHandlerMapping 要判断@Controller或者@RequestMapping注解
            detectHandlerMethods(beanName);
        }
    }
    //----------------detectHandlerMethods---------------
    protected void detectHandlerMethods(Object handler) {
            Class<?> handlerType = (handler instanceof String ?
                    obtainApplicationContext().getType((String) handler) : handler.getClass());

            if (handlerType != null) {
                Class<?> userType = ClassUtils.getUserClass(handlerType);

                Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
                        (MethodIntrospector.MetadataLookup<T>) method -> {
                            try {
                                //该函数由子类实现,返回泛型T,即当前userType中每个符合要求的函数的映射对象
                                return getMappingForMethod(method, userType);
                            }
                            catch (Throwable ex) {
                                throw new IllegalStateException("Invalid mapping on handler class [" +
                                        userType.getName() + "]: " + method, ex);
                            }
                        });
                if (logger.isTraceEnabled()) {
                    logger.trace(formatMappings(userType, methods));
                }
                methods.forEach((method, mapping) -> {
                    Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
                    //注册
                    //handler表示控制器对象
                    //invocableMethod表示对应函数
                    //mapping表示函数映射对象
                    registerHandlerMethod(handler, invocableMethod, mapping);
                });
            }
        }
        //获取handler
        protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
        String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
        request.setAttribute(LOOKUP_PATH, lookupPath);
        this.mappingRegistry.acquireReadLock(); //读锁
        try {
            //返回handler
            HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
            return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
        }
        finally {
            this.mappingRegistry.releaseReadLock();
        }
    }

  //获取HandlerMethod
    protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
        //Match为保存单个T和HandlerMethod的内部节点对象
        List<Match> matches = new ArrayList<>();
        List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
        if (directPathMatches != null) {
            addMatchingMappings(directPathMatches, matches, request);
        }
        if (matches.isEmpty()) {
            // No choice but to go through all mappings...
            addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
        }

        if (!matches.isEmpty()) {
            Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
            matches.sort(comparator);
            Match bestMatch = matches.get(0);
            if (matches.size() > 1) {
                if (logger.isTraceEnabled()) {
                    logger.trace(matches.size() + " matching mappings: " + matches);
                }
                if (CorsUtils.isPreFlightRequest(request)) {
                    return PREFLIGHT_AMBIGUOUS_MATCH;
                }
                Match secondBestMatch = matches.get(1);
                if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                    Method m1 = bestMatch.handlerMethod.getMethod();
                    Method m2 = secondBestMatch.handlerMethod.getMethod();
                    String uri = request.getRequestURI();
                    throw new IllegalStateException(
                            "Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
                }
            }
            request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
            handleMatch(bestMatch.mapping, lookupPath, request);
            return bestMatch.handlerMethod;
        }
        else {
            return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
        }
    }
}
{%endcodeblock%}
- 内部类MappingRegistry
 {%codeblock lang:java MappingRegistry %}
 class MappingRegistry {

        private final Map<T, MappingRegistration<T>> registry = new HashMap<>();
    //表示映射T和HanderMethod字典表
        private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();
    //url和映射T
        private final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<>();

        private final Map<String, List<HandlerMethod>> nameLookup = new ConcurrentHashMap<>();

        private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();

        private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();

        //注册handler逻辑,该函数一般在子类初始化时调用
        /**
        *@param T 表示映射对象
        *@param handler 表示如@Controller 类(即用户类,可能被代理)
        *@param method 表示方法
        */
        public void register(T mapping, Object handler, Method method) {
            // Assert that the handler method is not a suspending one.
            if (KotlinDetector.isKotlinType(method.getDeclaringClass()) && KotlinDelegate.isSuspend(method)) {
                throw new IllegalStateException("Unsupported suspending handler method detected: " + method);
            }
            this.readWriteLock.writeLock().lock();
            try {
                HandlerMethod handlerMethod = createHandlerMethod(handler, method);
                //若已经缓存则异常
                validateMethodMapping(handlerMethod, mapping);
                缓存T->hnadlerMethod
                this.mappingLookup.put(mapping, handlerMethod);
                //获取映射对象的url信息
                List<String> directUrls = getDirectUrls(mapping);
                //缓存url->T信息
                for (String url : directUrls) {
                    this.urlLookup.add(url, mapping);
                }

                String name = null;
                if (getNamingStrategy() != null) {
                    name = getNamingStrategy().getName(handlerMethod, mapping);
                    addMappingName(name, handlerMethod);
                }

                CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
                if (corsConfig != null) {
                    this.corsLookup.put(handlerMethod, corsConfig);
                }

                this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
            }
            finally {
                this.readWriteLock.writeLock().unlock();
            }
        }
 {%endcodeblock%}
 - RequestMappingInfoHandlerMapping
 {%codeblock lang:java RequestMappingInfoHandlerMapping%}
 //这就是上述映射T的实现,用来表示一个MVC请求函数信息,每个含义 易于理解
 public final class RequestMappingInfo implements RequestCondition<RequestMappingInfo> {
     @Nullable
     private final String name;

  //url路径 匹配条件
     private final PatternsRequestCondition patternsCondition;
  // http请求方法匹配条件
     private final RequestMethodsRequestCondition methodsCondition;
  //请求参数条件
     private final ParamsRequestCondition paramsCondition;
  //请求头条件
     private final HeadersRequestCondition headersCondition;

     private final ConsumesRequestCondition consumesCondition;

     private final ProducesRequestCondition producesCondition;

     private final RequestConditionHolder customConditionHolder;
    //省略...
}

 public abstract class RequestMappingInfoHandlerMapping extends AbstractHandlerMethodMapping<RequestMappingInfo> {
 }
 {%endcodeblock%}
 - RequestMappingHandlerMapping
 MVC中关于HandlerMethod的标准实现
 {%codeblock lang:java RequestMappingHandlerMapping%}
 public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
        implements MatchableHandlerMapping, EmbeddedValueResolverAware {
            //初始化
            public void afterPropertiesSet() {
        this.config = new RequestMappingInfo.BuilderConfiguration();
        this.config.setUrlPathHelper(getUrlPathHelper());
        this.config.setPathMatcher(getPathMatcher());
        this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
        this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
        this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
        this.config.setContentNegotiationManager(getContentNegotiationManager());

        super.afterPropertiesSet();
        }

        /**
        * 此函数在父类初始化时使用
        */
        @Override
    protected boolean isHandler(Class<?> beanType) {
        return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
                AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
    }
    /**
    *  父类初始化时调用
    */
    protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
        RequestMappingInfo info = createRequestMappingInfo(method);
        if (info != null) {
            RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
            if (typeInfo != null) {
                info = typeInfo.combine(info);
            }
            String prefix = getPathPrefix(handlerType);
            if (prefix != null) {
                info = RequestMappingInfo.paths(prefix).options(this.config).build().combine(info);
            }
        }
        return info;
    }
    }
 {%endcodeblock%}
#### HandlerAdapter:Handler适配器
##### 概念
设配器模式调用内部handler
{%asset_img HandlerAdapter.png HandlerAdapter%}
- HandlerAdapter:典型的适配器模式
{%codeblock lang:java HandlerAdapter%}
public interface HandlerAdapter {
    boolean supports(Object handler); //是否支持内部的handler
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception; //执行真正的逻辑
    long getLastModified(HttpServletRequest request, Object handler);
}
{%endcodeblock%}
- handle类型
    - Controller:spring提供的接口,返回MVC视图
    - HttpRequestHandler:不能返回视图
    - Servlet:不能返回视图
    - HandlerMethod:用户常见的控制器实际就是该类型
        - ResourceHttpRequestHandler:用来将请求转发给`DefaultServlet`处理静态资源
##### 2.2 不常用情况
- SimpleServletHandlerAdapter
用来处理Servlet类型,即`mapping`中handler 是Servlet
{%codeblock lang:java SimpleServletHandlerAdapter%}
public class SimpleServletHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return (handler instanceof Servlet);
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        ((Servlet) handler).service(request, response);
        return null;
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        return -1;
    }

}
{%endcodeblock%}
- SimpleControllerHandlerAdapter
用以处理`Controller`类型,spring提供的web接口
{%codeblock lang:java SimpleControllerHandlerAdapter%}
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return (handler instanceof Controller);
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        return ((Controller) handler).handleRequest(request, response);
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        if (handler instanceof LastModified) {
            return ((LastModified) handler).getLastModified(request);
        }
        return -1L;
    }

}

{%endcodeblock%}
- HttpRequestHandlerAdapter
支持`HttpRequestHandler`
{%codeblock lang:java HttpRequestHandler%}
public class HttpRequestHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return (handler instanceof HttpRequestHandler);
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        ((HttpRequestHandler) handler).handleRequest(request, response);
        return null;
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        if (handler instanceof LastModified) {
            return ((LastModified) handler).getLastModified(request);
        }
        return -1L;
    }
}
{%endcodeblock%}
- 总结
以上三种类型使用起来不太常见,`HttpRequestHandler`以及`Controller`类型spring由几个默认实现,如果spring没有提供特别的标签如
`<mvc:resource>`,用户使用一般就得创建特定的mapping,参考本文[SimpleUrlHandlerMapping](#abstracturlhandlermapping类型)cas的例子
#### RequestMappingHandlerAdapter
该适配器是最为常见的,也是大部分是时候都要使用的
- 调用模型
{%asset_img mvc调用模型.png%}
- 图中XXMethod,实际都是`HandlerMethod`封装代表了函数调用
    - `ModelMethod`:是对`@ModelAttribute`标记的函数的封装,其类型为`InvokeHandlerMethod`
    - `InitMethod`:是对`@InitBinder`标记函数的封装,其类型是`InvokeHandlerMethod`
    - `ControllerMethod`:是对此次请求匹配到的函数的封装,其类型是`ServletInvocableHandlerMethod`
##### 属性
```java
private List<HandlerMethodArgumentResolver> customArgumentResolvers;

    //入参解析器Composite
    @Nullable
    private HandlerMethodArgumentResolverComposite argumentResolvers;
  //用来创建DataBinder时,给其初始化的解析器组件
    @Nullable
    private HandlerMethodArgumentResolverComposite initBinderArgumentResolvers;
    //自定义返回值处理器
    @Nullable
    private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;
  //返回值处理器Composite
    @Nullable
    private HandlerMethodReturnValueHandlerComposite returnValueHandlers;
  //视图处理器
    @Nullable
    private List<ModelAndViewResolver> modelAndViewResolvers;
 //...
    private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();
  //消息转换器
    private List<HttpMessageConverter<?>> messageConverters;
  //...
    private List<Object> requestResponseBodyAdvice = new ArrayList<>();
  //WebDataBinder初始化
    @Nullable
    private WebBindingInitializer webBindingInitializer;
  //异步线程执行器
    private AsyncTaskExecutor taskExecutor = new SimpleAsyncTaskExecutor("MvcAsync");
  //异常超时时间
    @Nullable
    private Long asyncRequestTimeout;
  //异步Callable拦击器
    private CallableProcessingInterceptor[] callableInterceptors = new CallableProcessingInterceptor[0];
  //异步DeferredResult拦击器
    private DeferredResultProcessingInterceptor[] deferredResultInterceptors = new DeferredResultProcessingInterceptor[0];
  //....
    private ReactiveAdapterRegistry reactiveAdapterRegistry = ReactiveAdapterRegistry.getSharedInstance();
  //....  
    private boolean ignoreDefaultModelOnRedirect = false;

    private int cacheSecondsForSessionAttributeHandlers = 0;

    private boolean synchronizeOnSession = false;

    private SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();

    private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();

    @Nullable
    private ConfigurableBeanFactory beanFactory;

  //key值表示一个@Controller类

    //value表示其sesstionAttributesHandler
    private final Map<Class<?>, SessionAttributesHandler> sessionAttributesHandlerCache = new ConcurrentHashMap<>(64);
  //value表示@InitBinder在该类中的函数
    private final Map<Class<?>, Set<Method>> initBinderCache = new ConcurrentHashMap<>(64);
  // @ModelAttribute在该类中函数
    private final Map<Class<?>, Set<Method>> modelAttributeCache = new ConcurrentHashMap<>(64);

    //key表示一个@ControllerAdvice修饰的类
  // @ModelAttribute在该类中函数
    private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache = new LinkedHashMap<>();
  //value表示@InitBinder在该类中的函数
    private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache = new LinkedHashMap<>();
```
##### 内部组件
###### WebDataBinder
该组件用来bind,valid,converte,属于DataBinder体系,特别功能可以从Request中获取参数进行bind操作
{%codeblock lang:java WebDataBinder%}
        public class WebDataBinder extends DataBinder {
              //这两个数据都会在处理@ModelAttribute入参调用
              //表示前端传送的数据前缀
                private String fieldMarkerPrefix = DEFAULT_FIELD_MARKER_PREFIX;
                //表示前端传送数据的mark
                private String fieldDefaultPrefix = DEFAULT_FIELD_DEFAULT_PREFIX;

        }
{%endcodeblock%}
- preix作用
        {%codeblock lang:java 例子%}
                @InitBinder("prefix")
             public void initBinderTest(WebDataBinder dataBinder){
                   //该前缀说明,当按照入参的key找不到则按照prefix+para找
                     dataBinder.setFieldDefaultPrefix("u.");
                     ///该前缀说明,当按照入参的key找不到则按照prefix+para找,若找到,该属性置为空
                     dataBinder.setFieldMarkerPrefix("x.");
             }
             //请求参数为 ?u.name=123&x.password=xx
             @ApiOperation("用来说明dataBinder的prefix属性作用")
             @GetMapping("/prefixTest")
             public void initBinderMethodTest(@ModelAttribute("prefix") User user){
                     System.out.println(user);
                     //User{name='123', password='null'}
             }
            {%endcodeblock%}
###### SessionAttributesHandler
用来处理@SessionAttributes
{%codeblock lang:java SessionAttributesHandler%}
        //@SessionAttributes中的key
        private final Set<String> attributeNames = new HashSet<>();
  //同上type
    private final Set<Class<?>> attributeTypes = new HashSet<>();

    private final Set<String> knownAttributeNames = Collections.newSetFromMap(new ConcurrentHashMap<>(4));
  //用来向request获取|储存sesssion#attr的
    private final SessionAttributeStore sessionAttributeStore;
        //构造器
        public SessionAttributesHandler(Class<?> handlerType, SessionAttributeStore sessionAttributeStore) {
        Assert.notNull(sessionAttributeStore, "SessionAttributeStore may not be null");
        this.sessionAttributeStore = sessionAttributeStore;
    //获取handlerType中的@SessionAttributes,并且缓存name type
        SessionAttributes ann = AnnotatedElementUtils.findMergedAnnotation(handlerType, SessionAttributes.class);
        if (ann != null) {
            Collections.addAll(this.attributeNames, ann.names());
            Collections.addAll(this.attributeTypes, ann.types());
        }
        this.knownAttributeNames.addAll(this.attributeNames);
    }
{%endcodeblock%}
###### InitBinderFactory
用来创造WebDataBinder
- 关系
{%asset_img DataBinderFactory.png uml类图%}
- 逻辑
{%asset_img DataBinderFactroy逻辑.png 工厂逻辑%}
- 代码逻辑
{%codeblock lang:java DataBinderFactroy%}
            //-------------------------DefaultDataBinderFactory------------------------

            public final WebDataBinder createBinder(
                    NativeWebRequest webRequest, @Nullable Object target, String objectName) throws Exception {

                WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
                if (this.initializer != null) { //这个initializer也属于mvc配置模块的组件
                    this.initializer.initBinder(dataBinder, webRequest);
                }
                initBinder(dataBinder, webRequest);//子类实现
                return dataBinder;
            }
            //-------------------------InitBinderDataBinderFactory-------------------------

            public void initBinder(WebDataBinder dataBinder, NativeWebRequest request) throws Exception {
                for (InvocableHandlerMethod binderMethod : this.binderMethods) { //遍历InvocableHandlerMethod
                    if (isBinderMethodApplicable(binderMethod, dataBinder)) {//@InitBinder#name能够和DataBinder内部要处理的对象配置 || @InitBinder#name为空
                        //注意此处第三个参数,也就是说InitMethod,用户使用时形参中的WebDataBinder不是入参处理器提供的,而是这里提供的
                        Object returnValue = binderMethod.invokeForRequest(request, null, dataBinder);
                        if (returnValue != null) {
                            throw new IllegalStateException(
                                    "@InitBinder methods must not return a value (should be void): " + binderMethod);
                        }
                    }
                }
            }

            protected boolean isBinderMethodApplicable(HandlerMethod initBinderMethod, WebDataBinder dataBinder) {
                InitBinder ann = initBinderMethod.getMethodAnnotation(InitBinder.class);
                Assert.state(ann != null, "No InitBinder annotation");
                String[] names = ann.value();
                return (ObjectUtils.isEmpty(names) || ObjectUtils.containsElement(names, dataBinder.getObjectName()));
            }
{%endcodeblock%}
###### ModelFactory
处理@ModelAttribute(用于函数中),表示对此次mvc流程进行模型构造过程
-    属性
{%codeblock lang:java ModelFactory%}
        public final class ModelFactory {
        //封装了InvocableHandlerMethod
        private final List<ModelMethod> modelMethods = new ArrayList<>();
        //更新model时使用
        private final WebDataBinderFactory dataBinderFactory;
        //处理@SessionAttributes注解
        private final SessionAttributesHandler sessionAttributesHandler;
{%endcodeblock%}
- 逻辑
{%asset_img ModealFactory.png%}
- 代码逻辑
{%codeblock lang:java ModealFactory%}
        //HandlerMethod就是Controller的函数
        public void initModel(NativeWebRequest request, ModelAndViewContainer container, HandlerMethod handlerMethod)
            throws Exception {
        //从request中获取@SessionAttributes标识的session#attr        
        Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
        //mvc缓存
        container.mergeAttributes(sessionAttributes);
        //调用ModelMethod
        invokeModelAttributeMethods(request, container);
    //如果此次用调用的ControllerMethod中存在@ModelAttribute参数并且和@SessionAttributes匹配,
        //mvc容器在所有ModelMethod调用之后还是没有缓存
        for (String name : findSessionAttributeArguments(handlerMethod)) {
            if (!container.containsAttribute(name)) {
                Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
                if (value == null) {
                    throw new HttpSessionRequiredException("Expected session attribute '" + name + "'", name);
                }
                container.addAttribute(name, value);
            }
        }
    }
    //设法调用符合条件的ModelMethod
    private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)
            throws Exception {

        while (!this.modelMethods.isEmpty()) {
            InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();
            ModelAttribute ann = modelMethod.getMethodAnnotation(ModelAttribute.class);
            Assert.state(ann != null, "No ModelAttribute annotation");
            //这里说明若mvc已经存在对应模型,但是又遭遇到了modelMethod,
            //若modelMethod表示该属性!binding,mvc则改变对应属性binging,并跳过此次modelMethod函数
            if (container.containsAttribute(ann.name())) {
                if (!ann.binding()) {
                    container.setBindingDisabled(ann.name());
                }
                continue;
            }
      //modelMethod调用
            Object returnValue = modelMethod.invokeForRequest(request, container);
            if (!modelMethod.isVoid()){
                String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
                if (!ann.binding()) {
                    container.setBindingDisabled(returnValueName);
                }
                if (!container.containsAttribute(returnValueName)) {
                    container.addAttribute(returnValueName, returnValue);
                }
            }
        }
    }
    //匹配条件
    private ModelMethod getNextModelMethod(ModelAndViewContainer container) {

        for (ModelMethod modelMethod : this.modelMethods) {//若mvc并没有缓存请求的model则跳过
            if (modelMethod.checkDependencies(container)) {
                this.modelMethods.remove(modelMethod);
                return modelMethod;
            }
        }
        //这说明该modelMethod没有请求模型的必要,或者mvc就是没有缓存,那么就只能在入参处理时进行创造了
        ModelMethod modelMethod = this.modelMethods.get(0);
        this.modelMethods.remove(modelMethod);
        return modelMethod;
    }
    //获取视图时调用
    public void updateModel(NativeWebRequest request, ModelAndViewContainer container) throws Exception {
        ModelMap defaultModel = container.getDefaultModel();
        if (container.getSessionStatus().isComplete()){
            this.sessionAttributesHandler.cleanupAttributes(request);
        }
        else { //再给一次机会处理session,从model中获取符合key的
            this.sessionAttributesHandler.storeAttributes(request, defaultModel);
        }
        if (!container.isRequestHandled() && container.getModel() == defaultModel) {
            updateBindingResult(request, defaultModel); //属性添加BinderResult
        }
    }
    //
    private void updateBindingResult(NativeWebRequest request, ModelMap model) throws Exception {
        List<String> keyNames = new ArrayList<>(model.keySet());
        for (String name : keyNames) {
            Object value = model.get(name);
            if (value != null && isBindingCandidate(name, value)) { //符合条件
                String bindingResultKey = BindingResult.MODEL_KEY_PREFIX + name;
                //mvc未含有含bind前缀属性,即说明该属性没有经历ModelAttribute入参处理(即bindData验证等处理)
                if (!model.containsAttribute(bindingResultKey)) {
                    //缓存一个前缀+name dataBinder
                    WebDataBinder dataBinder = this.dataBinderFactory.createBinder(request, value, name);
                    model.put(bindingResultKey, dataBinder.getBindingResult());
                }
            }
        }
    }
    //该属性未添加前缀,为session候选||自定义类型
    private boolean isBindingCandidate(String attributeName, Object value) {
        if (attributeName.startsWith(BindingResult.MODEL_KEY_PREFIX)) {
            return false;
        }

        if (this.sessionAttributesHandler.isHandlerSessionAttribute(attributeName, value.getClass())) {
            return true;
        }

        return (!value.getClass().isArray() && !(value instanceof Collection) &&
                !(value instanceof Map) && !BeanUtils.isSimpleValueType(value.getClass()));
    }
//---------------------内部类-------------------------
private static class ModelMethod {

        private final InvocableHandlerMethod handlerMethod;

        private final Set<String> dependencies = new HashSet<>();

        public ModelMethod(InvocableHandlerMethod handlerMethod) {
            this.handlerMethod = handlerMethod;
            //若该ModelMethod形参函数@ModelAttrubite说明其请求模型,则加入依赖
            for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
                if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
                    this.dependencies.add(getNameForParameter(parameter));
                }
            }
        }

        public InvocableHandlerMethod getHandlerMethod() {
            return this.handlerMethod;
        }

        public boolean checkDependencies(ModelAndViewContainer mavContainer) {
            //简单来说就是看该ModelMethod形参中请求模型在mvc中是否缓存
            for (String name : this.dependencies) {
                if (!mavContainer.containsAttribute(name)) {
                    return false;
                }
            }
            return true;
        }

        @Override
        public String toString() {
            return this.handlerMethod.getMethod().toGenericString();
        }
    }
{%endcodeblock%}
###### HandlerMethod
封装调用函数
{%asset_img HandlerMethod.png HandlerMethod%}
- 逻辑

- HandlerMethod
基类,该类中提供获取形参,返回值类型信息,以及注解等能力
{%codeblock lang:java HandlerMethod%}

public class HandlerMethod {

    /** Logger that is available to subclasses. */
    protected final Log logger = LogFactory.getLog(getClass());

    private final Object bean;

    @Nullable
    private final BeanFactory beanFactory;

    private final Class<?> beanType;

    private final Method method;

    private final Method bridgedMethod;

    private final MethodParameter[] parameters;

    //说明当前函数请求处理状态
    @Nullable
    private HttpStatus responseStatus;
  //描述原因
    @Nullable
    private String responseStatusReason;

    @Nullable
    private HandlerMethod resolvedFromHandlerMethod;

    @Nullable
    private volatile List<Annotation[][]> interfaceParameterAnnotations;

    private final String description;
    //构造
    public HandlerMethod(Object bean, Method method) {
        Assert.notNull(bean, "Bean is required");
        Assert.notNull(method, "Method is required");
        this.bean = bean;
        this.beanFactory = null;
        this.beanType = ClassUtils.getUserClass(bean);
        this.method = method;
        this.bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
        this.parameters = initMethodParameters();
        evaluateResponseStatus();
        this.description = initDescription(this.beanType, this.method);
    }

    //根据定义的@ResponseStatus注解说明状态码和原因,这在子类调用中会使用到
    private void evaluateResponseStatus() {
        ResponseStatus annotation = getMethodAnnotation(ResponseStatus.class);
        if (annotation == null) {
            annotation = AnnotatedElementUtils.findMergedAnnotation(getBeanType(), ResponseStatus.class);
        }
        if (annotation != null) {
            this.responseStatus = annotation.code();
            this.responseStatusReason = annotation.reason();
        }
    }
}
{%endcodeblock%}
- InvokeHandlerMethod
{%codeblock lang:java InvocableHandlerMethod%}
        //提供给入参处理器使用来创建databinder
        private WebDataBinderFactory dataBinderFactory;
        //入参处理器
        private HandlerMethodArgumentResolverComposite resolvers = new HandlerMethodArgumentResolverComposite();
        //参数名处理
        private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();


        //调用逻辑
        public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {
        //入参处理调用        
        Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
        if (logger.isTraceEnabled()) {
            logger.trace("Arguments: " + Arrays.toString(args));
        }
        //反射调用真正的函数
        return doInvoke(args);
    }

    //入参处理
    protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {
        //形参封装        
        MethodParameter[] parameters = getMethodParameters();
        if (ObjectUtils.isEmpty(parameters)) {
            return EMPTY_ARGS;
        }

        Object[] args = new Object[parameters.length];
        for (int i = 0; i < parameters.length; i++) {
            MethodParameter parameter = parameters[i];
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            //获取默认提供的形参
            args[i] = findProvidedArgument(parameter, providedArgs);
            if (args[i] != null) {
                continue;
            }
            //调用对应的入参 处理器
            if (!this.resolvers.supportsParameter(parameter)) {
                throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
            }
            try {
                args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
            }
            catch (Exception ex) {
                // Leave stack trace for later, exception may actually be resolved and handled...
                if (logger.isDebugEnabled()) {
                    String exMsg = ex.getMessage();
                    if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
                        logger.debug(formatArgumentError(parameter, exMsg));
                    }
                }
                throw ex;
            }
        }
        return args;
    }
        {%endcodeblock%}
- ServletInvocableHandlerMethod
spring-mvc模块的实现类,具有回参处理器,并且提供了异步支持(暂时不讨论)
{%codeblock lang:java ServletInvocableHandlerMethod%}
public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
private HandlerMethodReturnValueHandlerComposite returnValueHandlers;
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {
        //InvokeHandlerMethod调用,处理入参,反射调用        
        Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
        //这里判断该method的请求状态
        setResponseStatus(webRequest);

        if (returnValue == null) {
            //未改变|| 状态码已经设置 || 请求处理结束
            if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {

                disableContentCachingIfNecessary(webRequest);
                mavContainer.setRequestHandled(true);
                return;
            }
        }
        //使用@ResponseStatus设置了reason
        else if (StringUtils.hasText(getResponseStatusReason())) {
            mavContainer.setRequestHandled(true);
            return;
        }

        mavContainer.setRequestHandled(false);
        Assert.state(this.returnValueHandlers != null, "No return value handlers");
        //回参处理
        try {
            this.returnValueHandlers.handleReturnValue(
                    returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
        }
        catch (Exception ex) {
            if (logger.isTraceEnabled()) {
                logger.trace(formatErrorForReturnValue(returnValue), ex);
            }
            throw ex;
        }
    }
}

/**
     * Set the response status according to the {@link ResponseStatus} annotation.
     */
    private void setResponseStatus(ServletWebRequest webRequest) throws IOException {
        HttpStatus status = getResponseStatus();
        if (status == null) { //若没有使用过@ResponseStatus,跳过
            return;
        }

        HttpServletResponse response = webRequest.getResponse();
        if (response != null) {
            String reason = getResponseStatusReason();
            if (StringUtils.hasText(reason)) {//若设置了原因,则调用servlet#sendError
                response.sendError(status.value(), reason);
            }
            else {
                response.setStatus(status.value());//设置相应状态码
            }
        }

        // To be picked up by RedirectView
        webRequest.getRequest().setAttribute(View.RESPONSE_STATUS_ATTRIBUTE, status);
    }
{%endcodeblock%}
###### ModelAndViewContainer
在mvc请求处理过程中作为形参一直传递模型
- 属性
{%codeblock lang:java ModelAndViewContainer%}
public class ModelAndViewContainer {

    private boolean ignoreDefaultModelOnRedirect = false;
    //视图对象
    @Nullable
    private Object view;
  //模型map
    private final ModelMap defaultModel = new BindingAwareModelMap();
    //重定向模型map
    @Nullable
    private ModelMap redirectModel;
    //是否使用重定向模型
    private boolean redirectModelScenario = false;
  //状态码
    @Nullable
    private HttpStatus status;

    private final Set<String> noBinding = new HashSet<>(4);
  //@ModelAttribute 指定不绑定的属性
    private final Set<String> bindingDisabled = new HashSet<>(4);

    private final SessionStatus sessionStatus = new SimpleSessionStatus();
  //标识请求是否结束
    private boolean requestHandled = false;
    /**
     * 使用默认model的条件是{@code redirectModelScenario=false}
     * 或者@RedirectAttributes未声明形参&&@code ignoreDefaultModelOnRedirect=false}
    */
    public ModelMap getModel() {
        if (useDefaultModel()) {
            return this.defaultModel;
        }
        else {
            if (this.redirectModel == null) {
                this.redirectModel = new ModelMap();
            }
            return this.redirectModel;
        }
    }
}
{%endcodeblock%}
- 属性调用点
`mvcContainer`在请求处理过程作为形参传递,内部许多变量都是处理组件的判断条件
    -    requestHandled
        {%asset_img mvcContainer_Handled.png%}
        - 入参处理器修改`true`会导致跳过回参处理,以及更新`bindResult`
        - 回参设置会导致跳过更新`bindResult`
        - 入参修改的仅仅有`ServletResponseMethodArgumentResolver`
这就是mvc请求处理过程中一直传递的mvc容器        
##### 适配器初始化
{%codeblock lang:java 初始化%}
public void afterPropertiesSet() {
        // Do this first, it may add ResponseBody advice beans
        //处理全局缓存
        initControllerAdviceCache();
        //入参处理器
        if (this.argumentResolvers == null) {
            List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
            this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
        }
        //InitBinder中的
        if (this.initBinderArgumentResolvers == null) {
            List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
            this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
        }
        if (this.returnValueHandlers == null) {
            List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
            this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
        }
    }
    //获取context中@ControllerAdvice类,将其中@ModelAttribute,@InitBinder,以及RequestBodyAdvice放置到对应缓存中
    private void initControllerAdviceCache() {
        if (getApplicationContext() == null) {
            return;
        }
        //获取context中@ControllerAdvice类
        List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());

        List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();

        for (ControllerAdviceBean adviceBean : adviceBeans) {
            Class<?> beanType = adviceBean.getBeanType();
            if (beanType == null) {
                throw new IllegalStateException("Unresolvable type for iControllerAdviceBean: " + adviceBean);
            }
            Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
            if (!attrMethods.isEmpty()) {
                this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
            }
            Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
            if (!binderMethods.isEmpty()) {
                this.initBinderAdviceCache.put(adviceBean, binderMethods);
            }
            //若该类还是res|req增强,则加入
            if (RequestBodyAdvice.class.isAssignableFrom(beanType) || ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
                requestResponseBodyAdviceBeans.add(adviceBean);
            }
        }

        if (!requestResponseBodyAdviceBeans.isEmpty()) {
            this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
        }

        if (logger.isDebugEnabled()) {
            int modelSize = this.modelAttributeAdviceCache.size();
            int binderSize = this.initBinderAdviceCache.size();
            int reqCount = getBodyAdviceCount(RequestBodyAdvice.class);
            int resCount = getBodyAdviceCount(ResponseBodyAdvice.class);
            if (modelSize == 0 && binderSize == 0 && reqCount == 0 && resCount == 0) {
                logger.debug("ControllerAdvice beans: none");
            }
            else {
                logger.debug("ControllerAdvice beans: " + modelSize + " @ModelAttribute, " + binderSize +
                        " @InitBinder, " + reqCount + " RequestBodyAdvice, " + resCount + " ResponseBodyAdvice");
            }
        }
    }
{%endcodeblock%}
##### 代码调用逻辑
{%codeblock lang:java RequestMappingHandlerAdapter%}
rotected ModelAndView invokeHandlerMethod(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        try {
            //两个工厂创建
            WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
                ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
          //将当前HandlerMethod转换为...
            ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
            if (this.argumentResolvers != null) {
                invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
            }
            if (this.returnValueHandlers != null) {
                invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
            }
            invocableMethod.setDataBinderFactory(binderFactory);
            invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

            ModelAndViewContainer mavContainer = new ModelAndViewContainer();
            mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
            //建立model,依次处理ModelMethod
            modelFactory.initModel(webRequest, mavContainer, invocableMethod);
            mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
     //异步框架
            AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
            asyncWebRequest.setTimeout(this.asyncRequestTimeout);

            WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
            asyncManager.setTaskExecutor(this.taskExecutor);
            asyncManager.setAsyncWebRequest(asyncWebRequest);
            asyncManager.registerCallableInterceptors(this.callableInterceptors);
            asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

            if (asyncManager.hasConcurrentResult()) {
                Object result = asyncManager.getConcurrentResult();
                mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
                asyncManager.clearConcurrentResult();
                LogFormatUtils.traceDebug(logger, traceOn -> {
                    String formatted = LogFormatUtils.formatValue(result, !traceOn);
                    return "Resume with async result [" + formatted + "]";
                });
                invocableMethod = invocableMethod.wrapConcurrentResult(result);
            }
      //ControllerMethod调用
            invocableMethod.invokeAndHandle(webRequest, mavContainer);
            if (asyncManager.isConcurrentHandlingStarted()) {
                return null;
            }
      //视图获取
            return getModelAndView(mavContainer, modelFactory, webRequest);
        }
        finally {
            webRequest.requestCompleted();
        }
    }

    //-----------------WebDataBinderFactory创建--------------------
    private WebDataBinderFactory getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {

        Class<?> handlerType = handlerMethod.getBeanType();
        //[1]尝试缓存
        Set<Method> methods = this.initBinderCache.get(handlerType);
        //[!1]
        //[2]当前ControllerMethod中获取标记了@InitBinder的函数
        if (methods == null) {
            methods = MethodIntrospector.selectMethods(handlerType, INIT_BINDER_METHODS);
            this.initBinderCache.put(handlerType, methods);
        }
        //[!2]

        //[3] 首先将增强类中的@InitBinder标记函数转换为InitMethod,再转换对应Controller中的
        List<InvocableHandlerMethod> initBinderMethods = new ArrayList<>();
        // Global methods first
        this.initBinderAdviceCache.forEach((controllerAdviceBean, methodSet) -> {
            if (controllerAdviceBean.isApplicableToBeanType(handlerType)) {
                Object bean = controllerAdviceBean.resolveBean();
                for (Method method : methodSet) {
                    initBinderMethods.add(createInitBinderMethod(bean, method));
                }
            }
        });
        for (Method method : methods) {
            Object bean = handlerMethod.getBean();
            initBinderMethods.add(createInitBinderMethod(bean, method));
        }
        //[!3]
        //创建工厂
        return createDataBinderFactory(initBinderMethods);
    }

    private InvocableHandlerMethod createInitBinderMethod(Object bean, Method method) {
        InvocableHandlerMethod binderMethod = new InvocableHandlerMethod(bean, method);
        if (this.initBinderArgumentResolvers != null) {//若这个为空,InitMethod就无法处理了处理DataBinder之外的形参?
            binderMethod.setHandlerMethodArgumentResolvers(this.initBinderArgumentResolvers);
        }
        binderMethod.setDataBinderFactory(new DefaultDataBinderFactory(this.webBindingInitializer));
        binderMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
        return binderMethod;
    }
    //---------------ModelFactory创建--------------------------------
    private ModelFactory getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
        //逻辑和上边一样,缓存->处理advice-->处理Controller中
        SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);
        Class<?> handlerType = handlerMethod.getBeanType();
        Set<Method> methods = this.modelAttributeCache.get(handlerType);
        if (methods == null) {
            methods = MethodIntrospector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
            this.modelAttributeCache.put(handlerType, methods);
        }
        List<InvocableHandlerMethod> attrMethods = new ArrayList<>();
        // Global methods first
        this.modelAttributeAdviceCache.forEach((controllerAdviceBean, methodSet) -> {
            if (controllerAdviceBean.isApplicableToBeanType(handlerType)) {
                Object bean = controllerAdviceBean.resolveBean();
                for (Method method : methodSet) {
                    attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
                }
            }
        });
        for (Method method : methods) {
            Object bean = handlerMethod.getBean();
            attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
        }
        return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
    }

    private InvocableHandlerMethod createModelAttributeMethod(WebDataBinderFactory factory, Object bean, Method method) {
        InvocableHandlerMethod attrMethod = new InvocableHandlerMethod(bean, method);
        if (this.argumentResolvers != null) {//ModelFactory拥有完整的入参处理器
            attrMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        attrMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
        attrMethod.setDataBinderFactory(factory);
        return attrMethod;
    }
    //-------------------------------------视图获取--------------------------------------------
    private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
            ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

        modelFactory.updateModel(webRequest, mavContainer);
        if (mavContainer.isRequestHandled()) {
            return null;
        }
        //重定向Model或DefaultModel
        ModelMap model = mavContainer.getModel();
        //创建ModelAndView
        ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
        if (!mavContainer.isViewReference()) {
            mav.setView((View) mavContainer.getView());
        }
        if (model instanceof RedirectAttributes) {
            //若为重定向Model,则处理FalshMap
            Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
            HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
            if (request != null) {
                RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
            }
        }
        return mav;
    }
    {%endcodeblock%}
##### invoke逻辑
此处逻辑是被父类抽象逻辑调用的
{%codeblock lang:java RequestMappingHandlerAdapter%}
protected ModelAndView handleInternal(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

        ModelAndView mav;
        checkRequest(request);

        // Execute invokeHandlerMethod in synchronized block if required.
        if (this.synchronizeOnSession) { //对于sessoin是否尝试加锁
            HttpSession session = request.getSession(false);
            if (session != null) {
                Object mutex = WebUtils.getSessionMutex(session);
                synchronized (mutex) {
                    mav = invokeHandlerMethod(request, response, handlerMethod);
                }
            }
            else {
                // No HttpSession available -> no mutex necessary
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
        else {
            // No synchronization on session demanded at all...
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
        //这里处理缓存头
        if (!response.containsHeader(HEADER_CACHE_CONTROL)) {//若存在cache-control说明invoke或者拦截器设置过了
            if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) { //若存在@SessionAttributes,则说明要专门对该session设置一个缓存时间
                applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
            }
            else {
                prepareResponse(response);//缓存处理
            }
        }
        return mav;
    }
{%endcodeblock%}
###### 缓存机制
{%asset_img Cache.png%}
参考[http缓存机制](/2020/03/31/http协议总结/#http缓存机制)
- WebContentGenerator:作为`RequestMapping`,`ResourceHttpRequestHandler`的父类
该类提供了很多关于web context生成的函数,比较复杂,这里简单分析缓存机制
{%codeblock lang:java WebContentGenerator%}
private CacheControl cacheControl;
protected final void prepareResponse(HttpServletResponse response) {
        if (this.cacheControl != null) {
            applyCacheControl(response, this.cacheControl);
        }
        else {
            applyCacheSeconds(response, this.cacheSeconds);
        }
        if (this.varyByRequestHeaders != null) {
            for (String value : getVaryRequestHeadersToAdd(response, this.varyByRequestHeaders)) {
                response.addHeader("Vary", value);
            }
        }
    }
    //-----------------处理Cache-control--------------------
    protected final void applyCacheControl(HttpServletResponse response, CacheControl cacheControl) {
        String ccValue = cacheControl.getHeaderValue();//获取缓存头
        if (ccValue != null) {
            // Set computed HTTP 1.1 Cache-Control header
            response.setHeader(HEADER_CACHE_CONTROL, ccValue); //设置缓存头

            if (response.containsHeader(HEADER_PRAGMA)) {//1.0协议头
                // Reset HTTP 1.0 Pragma header if present
                response.setHeader(HEADER_PRAGMA, "");
            }
            if (response.containsHeader(HEADER_EXPIRES)) {//1.0协议头
                // Reset HTTP 1.0 Expires header if present
                response.setHeader(HEADER_EXPIRES, "");
            }
        }
    }
    //----------------设置新鲜度标志--------------------------
    protected final void cacheForSeconds(HttpServletResponse response, int seconds) {
        cacheForSeconds(response, seconds, false);
    }
    protected final void applyCacheSeconds(HttpServletResponse response, int cacheSeconds) {
        if (this.useExpiresHeader || !this.useCacheControlHeader) { //前边是1.0,后边是1.1,spring5.2中后者为ture,前者为false
            // Deprecated HTTP 1.0 cache behavior, as in previous Spring versions
            if (cacheSeconds > 0) { //设置expires:xxx| 或者cache-control:max-age
                cacheForSeconds(response, cacheSeconds);
            }
            else if (cacheSeconds == 0) {
                preventCaching(response);
            }
        }
        else { //创建一个CacheControl,再调用aaplyCacheControl
            CacheControl cControl;
            if (cacheSeconds > 0) {
                cControl = CacheControl.maxAge(cacheSeconds, TimeUnit.SECONDS);
                if (this.alwaysMustRevalidate) {
                    cControl = cControl.mustRevalidate();
                }
            }
            else if (cacheSeconds == 0) {
                cControl = (this.useCacheControlNoStore ? CacheControl.noStore() : CacheControl.noCache());
            }
            else {
                cControl = CacheControl.empty();
            }
            applyCacheControl(response, cControl);
        }
    }
    //---------------设置新鲜度标志------------------
    protected final void cacheForSeconds(HttpServletResponse response, int seconds) {
            cacheForSeconds(response, seconds, false);
        }

        protected final void cacheForSeconds(HttpServletResponse response, int seconds, boolean mustRevalidate) {
            if (this.useExpiresHeader) {
                // HTTP 1.0 header
                response.setDateHeader(HEADER_EXPIRES, System.currentTimeMillis() + seconds * 1000L);
            }
            else if (response.containsHeader(HEADER_EXPIRES)) {
                // Reset HTTP 1.0 Expires header if present
                response.setHeader(HEADER_EXPIRES, "");
            }

            if (this.useCacheControlHeader) {
                // HTTP 1.1 header
                String headerValue = "max-age=" + seconds;
                if (mustRevalidate || this.alwaysMustRevalidate) {
                    headerValue += ", must-revalidate";
                }
                response.setHeader(HEADER_CACHE_CONTROL, headerValue);
            }

            if (response.containsHeader(HEADER_PRAGMA)) {
                // Reset HTTP 1.0 Pragma header if present
                response.setHeader(HEADER_PRAGMA, "");
            }
        }
        //time=0的情况
        protected final void preventCaching(HttpServletResponse response) {
            response.setHeader(HEADER_PRAGMA, "no-cache"); //no-cache表示获取最新更新

            if (this.useExpiresHeader) { //兼容1.0
                // HTTP 1.0 Expires header
                response.setDateHeader(HEADER_EXPIRES, 1L); //1毫秒
            }

            if (this.useCacheControlHeader) {
                // HTTP 1.1 Cache-Control header: "no-cache" is the standard value,
                // "no-store" is necessary to prevent caching on Firefox.
                response.setHeader(HEADER_CACHE_CONTROL, "no-cache");
                if (this.useCacheControlNoStore) { //表示不缓存
                    response.addHeader(HEADER_CACHE_CONTROL, "no-store");
                }
            }
        }
//==============================cacheControl实现===========================
public class CacheControl {
private String toHeaderValue() {
        StringBuilder headerValue = new StringBuilder();
        if (this.maxAge != null) {
            appendDirective(headerValue, "max-age=" + this.maxAge.getSeconds());
        }
        if (this.noCache) {
            appendDirective(headerValue, "no-cache");
        }
        if (this.noStore) {
            appendDirective(headerValue, "no-store");
        }
        if (this.mustRevalidate) {
            appendDirective(headerValue, "must-revalidate");
        }
        if (this.noTransform) {
            appendDirective(headerValue, "no-transform");
        }
        if (this.cachePublic) {
            appendDirective(headerValue, "public");
        }
        if (this.cachePrivate) {
            appendDirective(headerValue, "private");
        }
        if (this.proxyRevalidate) {
            appendDirective(headerValue, "proxy-revalidate");
        }
        if (this.sMaxAge != null) {
            appendDirective(headerValue, "s-maxage=" + this.sMaxAge.getSeconds());
        }
        if (this.staleIfError != null) {
            appendDirective(headerValue, "stale-if-error=" + this.staleIfError.getSeconds());
        }
        if (this.staleWhileRevalidate != null) {
            appendDirective(headerValue, "stale-while-revalidate=" + this.staleWhileRevalidate.getSeconds());
        }
        return headerValue.toString();
    }
}
{%endcodeblock%}
- 说明
    - `prepareResponse`的逻辑就是存在`CacheControl`则通过该属性来设置缓存,否则则设置新鲜度
        - applyCacheControl:设置Cache-control控制
        -    applyCacheSeconds:设置新鲜度
    - 在spring4.2推荐设置使用Cache-control,废弃新鲜度函数
