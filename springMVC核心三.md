---
title: springMVC核心三
cover: /img/spring.png
top_img: /img/post.jpg
date: 2020-03-26 15:21:37
tags:
- 框架
- spring
- springMvc
categories: java
description: 描述适配器处理结束后的视图处理以及异常机制
---
关于视图,我认为`mvc`框架中的提供完整视图从处理机制,和使用`Rest`开发基本是冲突,关于`View`
部分,如模板处理,不做过多分析.
## 视图解析器
{%asset_img ViewResolver.png%}
简单来说分为三类,`ViewResolverComposite`为典型的`Composite`,一般为`DispatcherServlet`持有
- 接口定义
{%codeblock lang:java ViewResolver%}
View resolveViewName(String viewName, Locale locale) throws Exception;
{%endcodeblock%}
### BeanNameViewResolver
从`Context`中获取`View`
- 代码逻辑
{%codeblock lang:java BeanNameViewResolver%}
public View resolveViewName(String viewName, Locale locale) throws BeansException {
        ApplicationContext context = obtainApplicationContext();
        if (!context.containsBean(viewName)) {
            // Allow for ViewResolver chaining...
            return null;
        }
        if (!context.isTypeMatch(viewName, View.class)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Found bean named '" + viewName + "' but it does not implement View");
            }
            // Since we're looking into the general ApplicationContext here,
            // let's accept this as a non-match and allow for chaining as well...
            return null;
        }
        return context.getBean(viewName, View.class);
    }
{%endcodeblock%}
### ContentNegotiating
使用了`ContentNegotiationManager`,该类用来通过`Request`判断`MiedaType`
- 代码逻辑
{%codeblock lang:java ContentNegotiatingViewResolver%}
  public class ContentNegotiatingViewResolver extends WebApplicationObjectSupport
          implements ViewResolver, Ordered, InitializingBean {

            private ContentNegotiationManager contentNegotiationManager;
          //FactoryBean
          private final ContentNegotiationManagerFactoryBean cnmFactoryBean = new ContentNegotiationManagerFactoryBean();
          //内部含有的Resolver
          private List<ViewResolver> viewResolvers;
        //WebApplicationObjectSupport提供的,调用点实际上是ApplicationContextAwareProcessor#postProcessBeforeInitialization
        //即populate之后调用
        protected void initServletContext(ServletContext servletContext) {
          Collection<ViewResolver> matchingBeans =
                  BeanFactoryUtils.beansOfTypeIncludingAncestors(obtainApplicationContext(), ViewResolver.class).values();
          if (this.viewResolvers == null) { //若本ViewResolver并没有定义viewResovler,则将beanFacory其他解析器添加
              this.viewResolvers = new ArrayList<>(matchingBeans.size());
              for (ViewResolver viewResolver : matchingBeans) {
                  if (this != viewResolver) {
                      this.viewResolvers.add(viewResolver);
                  }
              }
          }
          else {
        //若执行到这里viewResolvers.!=null
        //说明有某些处理器,如InstantiationAwareBeanPostProcessor可能添加ViewResolver到这里
        //并且不是通过pvs添加,这样直接添加的是不属于beanFactory管理的
              for (int i = 0; i < this.viewResolvers.size(); i++) {
                  ViewResolver vr = this.viewResolvers.get(i);
                  if (matchingBeans.contains(vr)) { //明显就判断了一下,已经含有的不能是beanFactory中的
                      continue;
                  }
                  String name = vr.getClass().getName() + i;
          //将并非是ioc容器创造出来的解析器调用init
                  obtainApplicationContext().getAutowireCapableBeanFactory().initializeBean(vr, name);
              }

          }
          AnnotationAwareOrderComparator.sort(this.viewResolvers);
          this.cnmFactoryBean.setServletContext(servletContext);
      }
    //创造内部的ContextNegotiationMannager
    public void afterPropertiesSet() {
          if (this.contentNegotiationManager == null) {
              this.contentNegotiationManager = this.cnmFactoryBean.build();
          }
          if (this.viewResolvers == null || this.viewResolvers.isEmpty()) {
              logger.warn("No ViewResolvers configured");
          }
      }

    //视图解析
    public View resolveViewName(String viewName, Locale locale) throws Exception {
          RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
          Assert.state(attrs instanceof ServletRequestAttributes, "No current ServletRequestAttributes");
          List<MediaType> requestedMediaTypes = getMediaTypes(((ServletRequestAttributes) attrs).getRequest());
          if (requestedMediaTypes != null) {
        //遍历内部所有解析器,获取候选view
              List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);
        //选择View#getContextTpe和判断出来的MiedaType符合的视图
              View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
            if (bestView != null) {
                return bestView;
            }
        }

        String mediaTypeInfo = logger.isDebugEnabled() && requestedMediaTypes != null ?
                " given " + requestedMediaTypes.toString() : "";

        if (this.useNotAcceptableStatusCode) {
            if (logger.isDebugEnabled()) {
                logger.debug("Using 406 NOT_ACCEPTABLE" + mediaTypeInfo);
            }
            return NOT_ACCEPTABLE_VIEW;
        }
        else {
            logger.debug("View remains unresolved" + mediaTypeInfo);
            return null;
        }
    }
}      
{%endcodeblock%}
- 总结
  - 改类本质也是委托模式,包含`List<ViewResolver>`来处理
  - 目的是为了选出`View#getContexType`和`MiedaType`符合的视图
  - 该类可以通过`mvc:content-negotiation`进行配置,参考`ViewResolversBeanDefinitionParser`实现
### AbstractCachingViewResolver  
简而言之,此类是带有缓存的视图解析器,总体逻辑简单,由子类实现`loadView()`来创建视图
- 代码逻辑
{%codeblock lang:java AbstractCachingViewResolver%}
    public static final int DEFAULT_CACHE_LIMIT = 1024;

  //Fast access cache for Views, returning already cached instances without a global lock.
  //容量为1024的并发map
  //key实际上是 viewname+Local.toString(当前请求的Local字符串化,如zh,en)
private final Map<Object, View> viewAccessCache = new ConcurrentHashMap<>(DEFAULT_CACHE_LIMIT);
//Map from view key to View instance, synchronized for View creation.
//创建缓存时加锁,这里有趣的是重写removeEldestEntry
//简单来说在{@link LinkedHashMap}中说明该函数是当插入新元素时,可以选择将eldest元素是否删除,
//典型的就是当使用map作为含有limit的时候,如这里1024,超过该值就选择删除最早的元素
private final Map<Object, View> viewCreationCache =
            new LinkedHashMap<Object, View>(DEFAULT_CACHE_LIMIT, 0.75f, true) {
                @Override
                protected boolean removeEldestEntry(Map.Entry<Object, View> eldest) {
                    if (size() > getCacheLimit()) {
                        viewAccessCache.remove(eldest.getKey());//主动删除viewAccessCache缓存
            //返回true后this的eldest会被LinkHashMap删除
                        return true;
                    }
                    else {
                        return false;
                    }
                }
            };
      //构建视图逻辑
      public View resolveViewName(String viewName, Locale locale) throws Exception {
              if (!isCache()) {
                  return createView(viewName, locale);
              }
              else {
                  Object cacheKey = getCacheKey(viewName, locale);
                  View view = this.viewAccessCache.get(cacheKey);
                  if (view == null) {
                      synchronized (this.viewCreationCache) {
                //简单来看,当创建的时候将降维单线程
                //1.可以防止当CacheKey相同时,发生不必要的重复创建工作,但是这点可以通过sych(cacheKey.intern())实现
                //2.我认为是子类实现的createView未必是线程安全的,因此这么实现锁
                          view = this.viewCreationCache.get(cacheKey);
                          if (view == null) {
                              // Ask the subclass to create the View object.
                              view = createView(viewName, locale);
                              if (view == null && this.cacheUnresolved) {
                                  view = UNRESOLVED_VIEW;
                              }
                              if (view != null && this.cacheFilter.filter(view, viewName, locale)) {
                                  this.viewAccessCache.put(cacheKey, view);
                                  this.viewCreationCache.put(cacheKey, view);
                              }
                          }
                      }
                  }
                  else {
                      if (logger.isTraceEnabled()) {
                          logger.trace(formatKey(cacheKey) + "served from cache");
                      }
                  }
                  return (view != UNRESOLVED_VIEW ? view : null);
              }
          }

        protected View createView(String viewName, Locale locale) throws Exception {
        return loadView(viewName, locale);
    }
        protected abstract View loadView(String viewName, Locale locale) throws Exception;
{%endcodeblock%}
- 总结
  - 附带缓存,并且实现了线程安全
#### ResourceBundleViewResolver
该类实现上使用了`ResourceBundle`来对于不同`Local`根据不同的in18配置文件,返回不同的`View`
- 代码逻辑
{%codeblock lang:java ResourceBundleViewResolver%}
//properties默认名字
public static final String DEFAULT_BASENAME = "views";

protected View loadView(String viewName, Locale locale) throws Exception {
        BeanFactory factory = initFactory(locale);//创造不同的bf
        try {
            return factory.getBean(viewName, View.class);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Allow for ViewResolver chaining...
            return null;
        }
    }

  protected synchronized BeanFactory initFactory(Locale locale) throws BeansException {
          // Try to find cached factory for Locale:
          // Have we already encountered that Locale before?
          if (isCache()) {
              BeanFactory cachedFactory = this.localeCache.get(locale);
              if (cachedFactory != null) {
                  return cachedFactory;
              }
          }

          // Build list of ResourceBundle references for Locale.
          List<ResourceBundle> bundles = new LinkedList<>();
          for (String basename : this.basenames) {
              ResourceBundle bundle = getBundle(basename, locale);
              bundles.add(bundle);
          }

          // Try to find cached factory for ResourceBundle list:
          // even if Locale was different, same bundles might have been found.
          if (isCache()) {
              BeanFactory cachedFactory = this.bundleCache.get(bundles);
              if (cachedFactory != null) {
                  this.localeCache.put(locale, cachedFactory);
                  return cachedFactory;
              }
          }

          // Create child ApplicationContext for views.
      //通过上边获取到的ResourceBundle去读取不同的properties文件,并加载进行GAC
          GenericWebApplicationContext factory = new GenericWebApplicationContext();
          factory.setParent(getApplicationContext());
          factory.setServletContext(getServletContext());

          // Load bean definitions from resource bundle.
      //properties配置读取器
          PropertiesBeanDefinitionReader reader = new PropertiesBeanDefinitionReader(factory);
          reader.setDefaultParentBean(this.defaultParentView);
          for (ResourceBundle bundle : bundles) {
              reader.registerBeanDefinitions(bundle);
          }

          factory.refresh();

          // Cache factory for both Locale and ResourceBundle list.
          if (isCache()) {
              this.localeCache.put(locale, factory);
              this.bundleCache.put(bundles, factory);
          }

          return factory;
      }
{%endcodeblock%}
- 总结
  - 实现简单,具体如何配置参考后文
  - properties必须是`views.properties`这样的命名
  - 例子
    ```properties
    testViewName.(class)=org.springframework.web.servlet.view.InternalResourceView
    testViewName.url=/WEB-INF/index.jsp
    ```
  - 关于`properties`作为配置文件,参考`PropertiesBeanDefinitionReader`  
#### XmlViewResolver
和`ResourceBundleViewResolver`实现一致,不过是从xml中读取
- 代码逻辑
{%codeblock lang:java XmlViewResolver%}   
public static final String DEFAULT_LOCATION = "/WEB-INF/views.xml";

protected View loadView(String viewName, Locale locale) throws BeansException {
        BeanFactory factory = initFactory();
        try {
            return factory.getBean(viewName, View.class);
        }
        catch (NoSuchBeanDefinitionException ex) {
            // Allow for ViewResolver chaining...
            return null;
        }
    }

  protected synchronized BeanFactory initFactory() throws BeansException {
        if (this.cachedFactory != null) {
            return this.cachedFactory;
        }

        ApplicationContext applicationContext = obtainApplicationContext();

        Resource actualLocation = this.location;
        if (actualLocation == null) {
            actualLocation = applicationContext.getResource(DEFAULT_LOCATION);
        }

        // Create child ApplicationContext for views.
    //读取xml并创建gac
        GenericWebApplicationContext factory = new GenericWebApplicationContext();
        factory.setParent(applicationContext);
        factory.setServletContext(getServletContext());

        // Load XML resource with context-aware entity resolver.
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
        reader.setEnvironment(applicationContext.getEnvironment());
        reader.setEntityResolver(new ResourceEntityResolver(applicationContext));
        reader.loadBeanDefinitions(actualLocation);

        factory.refresh();

        if (isCache()) {
            this.cachedFactory = factory;
        }
        return factory;
    }
- 总结
  - 上述两种视图解析器的使用,用例子说明了gbc中的bean定义来源是可以多样化的
{%endcodeblock%}
#### UrlBasedViewResolver
- 代码逻辑
{%codeblock lang:java  UrlBasedViewResolver%}
public static final String REDIRECT_URL_PREFIX = "redirect:";
public static final String FORWARD_URL_PREFIX = "forward:";
//返回的视图对象
private Class<?> viewClass;
private String prefix = "";
private String suffix = "";

//重写父类,为了处理重定向和转发视图
protected View createView(String viewName, Locale locale) throws Exception {
        // If this resolver is not supposed to handle the given view,
        // return null to pass on to the next resolver in the chain.
        if (!canHandle(viewName, locale)) {
            return null;
        }
        //处理RedirectView
        // Check for special "redirect:" prefix.
        if (viewName.startsWith(REDIRECT_URL_PREFIX)) {
            String redirectUrl = viewName.substring(REDIRECT_URL_PREFIX.length());
            RedirectView view = new RedirectView(redirectUrl,
                    isRedirectContextRelative(), isRedirectHttp10Compatible());
            String[] hosts = getRedirectHosts();
            if (hosts != null) {
                view.setHosts(hosts);
            }
            return applyLifecycleMethods(REDIRECT_URL_PREFIX, view);
        }
        //创建Internal视图
        // Check for special "forward:" prefix.
        if (viewName.startsWith(FORWARD_URL_PREFIX)) {
            String forwardUrl = viewName.substring(FORWARD_URL_PREFIX.length());
            InternalResourceView view = new InternalResourceView(forwardUrl);
            return applyLifecycleMethods(FORWARD_URL_PREFIX, view);
        }

        // Else fall back to superclass implementation: calling loadView.
        return super.createView(viewName, locale);
    }

//由不同的子类决定不同的视图
protected Class<?> requiredViewClass() {
        return AbstractUrlBasedView.class;
    }
    protected View loadView(String viewName, Locale locale) throws Exception {
            AbstractUrlBasedView view = buildView(viewName);
            View result = applyLifecycleMethods(viewName, view);
            return (view.checkResource(locale) ? result : null);
        }

    //初始化
    protected View applyLifecycleMethods(String viewName, AbstractUrlBasedView view) {
        ApplicationContext context = getApplicationContext();
        if (context != null) {
            Object initialized = context.getAutowireCapableBeanFactory().initializeBean(view, viewName);
            if (initialized instanceof View) {
                return (View) initialized;
            }
        }
        return view;
    }
    //构建视图对象,并没有通过bf构建,而是反射
    protected AbstractUrlBasedView buildView(String viewName) throws Exception {
            Class<?> viewClass = getViewClass();
            Assert.state(viewClass != null, "No view class");

            //子类决定类型,为何不使用泛型来实现
            AbstractUrlBasedView view = (AbstractUrlBasedView) BeanUtils.instantiateClass(viewClass);
            view.setUrl(getPrefix() + viewName + getSuffix());
            view.setAttributesMap(getAttributesMap());

            String contentType = getContentType();
            if (contentType != null) {
                view.setContentType(contentType);
            }

            String requestContextAttribute = getRequestContextAttribute();
            if (requestContextAttribute != null) {
                view.setRequestContextAttribute(requestContextAttribute);
            }

            Boolean exposePathVariables = getExposePathVariables();
            if (exposePathVariables != null) {
                view.setExposePathVariables(exposePathVariables);
            }
            Boolean exposeContextBeansAsAttributes = getExposeContextBeansAsAttributes();
            if (exposeContextBeansAsAttributes != null) {
                view.setExposeContextBeansAsAttributes(exposeContextBeansAsAttributes);
            }
            String[] exposedContextBeanNames = getExposedContextBeanNames();
            if (exposedContextBeanNames != null) {
                view.setExposedContextBeanNames(exposedContextBeanNames);
            }

            return view;
        }
{%endcodeblock%}
- 总体
    - 支持`forward:`和`redirect:`
    - 带有`prefix`和`suffix`
    - 由子类决定`viewClass`
    - 重写`createView`来处理`rediect`视图,以及`forward`视图
##### 子类
- InternalResourceViewResolver:用来构建`InternalResourceView`
{%codeblock lang:java InternalResourceViewResolver%}
public InternalResourceViewResolver() {
        Class<?> viewClass = requiredViewClass();
        if (InternalResourceView.class == viewClass && jstlPresent) {
            viewClass = JstlView.class;
        }
        setViewClass(viewClass);
    }

protected AbstractUrlBasedView buildView(String viewName) throws Exception {
        InternalResourceView view = (InternalResourceView) super.buildView(viewName);
        if (this.alwaysInclude != null) {
            view.setAlwaysInclude(this.alwaysInclude);
        }
        view.setPreventDispatchLoop(true);
        return view;
    }
{%endcodeblock%}
- TilesViewResolver:构建`TilesView`
- ScriptTemplateViewResolver:构建`ScriptTemplateView`
- XsltViewResolver:构建`XsltView`
## 视图
接口定义
{%codeblock lang:java View%}
public interface View {
    default String getContentType() {
        return null;
    }

    void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
                throws Exception;
}
{%endcodeblock%}
{%asset_img View.png%}
### AbstactView
- 代码逻辑
{%codeblock lang:java AbstactView%}
public abstract class AbstractView extends WebApplicationObjectSupport implements View, BeanNameAware {


    public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
            HttpServletResponse response) throws Exception {

        if (logger.isDebugEnabled()) {
            logger.debug("View " + formatViewName() +
                    ", model " + (model != null ? model : Collections.emptyMap()) +
                    (this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
        }

        Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
        prepareResponse(request, response);//处理响应头
        renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
    }
    //合并属性
    protected Map<String, Object> createMergedOutputModel(@Nullable Map<String, ?> model,
            HttpServletRequest request, HttpServletResponse response) {

        //View.PATH_VARIABLES map
        @SuppressWarnings("unchecked")
        Map<String, Object> pathVars = (this.exposePathVariables ?
                (Map<String, Object>) request.getAttribute(View.PATH_VARIABLES) : null);

        // Consolidate static and dynamic model attributes.
        //staticAttributes map
        int size = this.staticAttributes.size();
        size += (model != null ? model.size() : 0);
        size += (pathVars != null ? pathVars.size() : 0);

        Map<String, Object> mergedModel = new LinkedHashMap<>(size);
        mergedModel.putAll(this.staticAttributes);
        if (pathVars != null) {
            mergedModel.putAll(pathVars);
        }
        //model map
        if (model != null) {
            mergedModel.putAll(model);
        }

        // Expose RequestContext?
        if (this.requestContextAttribute != null) {
            mergedModel.put(this.requestContextAttribute, createRequestContext(request, response, mergedModel));
        }

        return mergedModel;
    }

    protected void prepareResponse(HttpServletRequest request, HttpServletResponse response) {
        if (generatesDownloadContent()) {
            response.setHeader("Pragma", "private");
            response.setHeader("Cache-Control", "private, must-revalidate");
        }
    }

    //子类实现
    protected abstract void renderMergedOutputModel(
                Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
}
{%endcodeblock%}
#### AbstractJackson2View
代码简单,通过jackson api进行`application/json`和`application/xml`转换处理
- MappingJackson2JsonView
处理上文中合并的`modelMap`,将value转换成json
- MappingJackson2XmlView
处理上文中合并的`modelMap`,将value转换成xml
#### AbstractUrlBasedView
该部分子类和`UrlBasedViewResolver`子类相对应
##### InternalResourceView
- 代码逻辑
{%codeblock lang:java InternalResourceView%}
public class InternalResourceView extends AbstractUrlBasedView {
    protected void renderMergedOutputModel(
            Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

        // Expose the model object as request attributes.
        //model属性添加到res中
        exposeModelAsRequestAttributes(model, request);

        // Expose helpers as request attributes, if any.
        exposeHelpers(request);

        // Determine the path for the request dispatcher
        // url  = prefix  +viewName+ suffix
        String dispatcherPath = prepareForRendering(request, response);

        // Obtain a RequestDispatcher for the target resource (typically a JSP).
        RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
        if (rd == null) {
            throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
                    "]: Check that the corresponding file exists within your web application archive!");
        }

        // If already included or response already committed, perform include, else forward.
        if (useInclude(request, response)) {
            response.setContentType(getContentType());
            if (logger.isDebugEnabled()) {
                logger.debug("Including [" + getUrl() + "]");
            }
            rd.include(request, response);
        }

        else {
            // Note: The forwarded resource is supposed to determine the content type itself.
            if (logger.isDebugEnabled()) {
                logger.debug("Forwarding to [" + getUrl() + "]");
            }
            rd.forward(request, response);
        }
    }
}
{%endcodeblock%}
- 总结
    - `prefix`+viewname+'suffix'访问jsp或servlet,以及其他页面
    - 通过`RequestDispatcher`进行`include`和`forward`处理
##### RedirectView
- 代码逻辑
{%codeblock lang:java RedirectView%}
public class RedirectView extends AbstractUrlBasedView implements SmartView {
    //匹配重定义url中是否有占位符url
    private static final Pattern URI_TEMPLATE_VARIABLE_PATTERN = Pattern.compile("\\{([^/]+?)\\}");
    //
    protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
            HttpServletResponse response) throws IOException {

        String targetUrl = createTargetUrl(model,request);
        //调用一个 RequestDataValueProcessor当作触发器,默认 没有实现
        targetUrl = updateTargetUrl(targetUrl, model, request, response);

        // Save flash attributes
        //将outFlash数据放置到重定向url中
        RequestContextUtils.saveOutputFlashMap(targetUrl, request, response);

        // Redirect
        //response重定向设置
        sendRedirect(request, response, targetUrl, this.http10Compatible);
    }


    protected final String createTargetUrl(Map<String, Object> model, HttpServletRequest request)
                throws UnsupportedEncodingException {

            // Prepare target URL.
            StringBuilder targetUrl = new StringBuilder();
            String url = getUrl();
            Assert.state(url != null, "'url' not set");

            if (this.contextRelative && getUrl().startsWith("/")) {
                // Do not apply context path to relative URLs.
                targetUrl.append(getContextPath(request));
            }
            targetUrl.append(getUrl());

            String enc = this.encodingScheme;
            if (enc == null) {
                enc = request.getCharacterEncoding();
            }
            if (enc == null) {
                enc = WebUtils.DEFAULT_CHARACTER_ENCODING;
            }
            //替换重定向中的temp url
            if (this.expandUriTemplateVariables && StringUtils.hasText(targetUrl)) {
                //获取当前的temp url
                Map<String, String> variables = getCurrentRequestUriVariables(request);
                targetUrl = replaceUriTemplateVariables(targetUrl.toString(), model, variables, enc);
            }
            if (isPropagateQueryProperties()) {
                appendCurrentQueryParams(targetUrl, request);
            }
            if (this.exposeModelAttributes) {
                appendQueryProperties(targetUrl, model, enc);
            }

            return targetUrl.toString();
        }

        private String getContextPath(HttpServletRequest request) {
            String contextPath = request.getContextPath();
            while (contextPath.startsWith("//")) {
                contextPath = contextPath.substring(1);
            }
            return contextPath;
        }

        //一段一段替换
        protected StringBuilder replaceUriTemplateVariables(
            String targetUrl, Map<String, Object> model, Map<String, String> currentUriVariables, String encodingScheme)
            throws UnsupportedEncodingException {

        StringBuilder result = new StringBuilder();
        Matcher matcher = URI_TEMPLATE_VARIABLE_PATTERN.matcher(targetUrl);
        int endLastMatch = 0;
        while (matcher.find()) {
            String name = matcher.group(1);
            Object value = (model.containsKey(name) ? model.remove(name) : currentUriVariables.get(name));
            if (value == null) {
                throw new IllegalArgumentException("Model has no value for key '" + name + "'");
            }
            result.append(targetUrl.substring(endLastMatch, matcher.start()));
            result.append(UriUtils.encodePathSegment(value.toString(), encodingScheme));
            endLastMatch = matcher.end();
        }
        result.append(targetUrl.substring(endLastMatch));
        return result;
    }

    private Map<String, String> getCurrentRequestUriVariables(HttpServletRequest request) {
            String name = HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE; //temp url是在mapping中处理的
            Map<String, String> uriVars = (Map<String, String>) request.getAttribute(name);
            return (uriVars != null) ? uriVars : Collections.<String, String> emptyMap();
        }
}
{%endcodeblock%}
- 总结
    - 支持temp url
        - 例子
            ```java
                @GetMapping("redirect/{red}/test/{red2}")
                public String redirectView(RedirectAttributes attributes){
                        attributes.addAttribute("reKey","reValue");
                        return "redirect:/redirect/{red}/test/{red2}/xx";
                }
            ```
    - 会将`OutFlashMap`的数据,拼接到重定向url中,并且缓存到`FlashMapManager`

    剩下的视图不做过多分析,基本基于`rest`是不会使用的
## 异常处理器
### DispatcherServlet调用逻辑
{%codeblock lang:java DispatcherServlet%}
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
            @Nullable Object handler, Exception ex) throws Exception {

        // Success and error responses may use different content types
        request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

        // Check registered HandlerExceptionResolvers...
        ModelAndView exMv = null;
        //[1!] 处理器调用
        if (this.handlerExceptionResolvers != null) {
            for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
                exMv = resolver.resolveException(request, response, handler, ex);
                if (exMv != null) {
                    break;
                }
            }
        }
        //[1]
        //[2]处理
        if (exMv != null) {
            if (exMv.isEmpty()) {
                request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
                return null;
            }
            // We might still need view name translation for a plain error model...
            if (!exMv.hasView()) {
                String defaultViewName = getDefaultViewName(request);
                if (defaultViewName != null) {
                    exMv.setViewName(defaultViewName);
                }
            }
            if (logger.isTraceEnabled()) {
                logger.trace("Using resolved error view: " + exMv, ex);
            }
            else if (logger.isDebugEnabled()) {
                logger.debug("Using resolved error view: " + exMv);
            }
            WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
            return exMv;
        }
        //[2!]
        //[3] 若处理器没有正确返回视图,则抛出异常
        throw ex;
    }
{%endcodeblock%}
### HandlerExceptionResolver体系
{%asset_img HandlerExceptionRe.png HandlerExceptionResolver.png%}
#### HandlerExceptionResolverComposite
典型的`Composite`,`DispatcherServlet`默认并不是这个,在使用注解配置mvc的`WebMvcConfigurationSupport#handlerExceptionResolver`中才
使用了这个来配置
#### AbstractHandlerExceptionResolver
- 代码逻辑
{%codeblock lang:java AbstractHandlerExceptionResolver%}
public abstract class AbstractHandlerExceptionResolver implements HandlerExceptionResolver, Ordered {
    //Handler实例,如Controller,HandlerMethod,等
    private Set<?> mappedHandlers;
    private static final String HEADER_CACHE_CONTROL = "Cache-Control";
    @Nullable
    //Handler类对象
    private Class<?>[] mappedHandlerClasses;

    public ModelAndView resolveException(
            HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

        if (shouldApplyTo(request, handler)) {//判断是否进行异常处理
            prepareResponse(ex, response); //设置一个Cache-Control
            ModelAndView result = doResolveException(request, response, handler, ex);//子类实现
            if (result != null) {
                // Print debug message when warn logger is not enabled.
                if (logger.isDebugEnabled() && (this.warnLogger == null || !this.warnLogger.isWarnEnabled())) {
                    logger.debug("Resolved [" + ex + "]" + (result.isEmpty() ? "" : " to " + result));
                }
                // Explicitly configured warn logger in logException method.
                logException(ex, request);
            }
            return result;
        }
        else {
            return null;
        }
    }

    protected boolean shouldApplyTo(HttpServletRequest request, @Nullable Object handler) {
        if (handler != null) {//包含实例
            if (this.mappedHandlers != null && this.mappedHandlers.contains(handler)) {
                return true;
            }
            if (this.mappedHandlerClasses != null) {//包含类
                for (Class<?> handlerClass : this.mappedHandlerClasses) {
                    if (handlerClass.isInstance(handler)) {
                        return true;
                    }
                }
            }
        }
        // Else only apply if there are no explicit handler mappings.
        //若不去设置,这里一般都是这条件,全通过
        return (this.mappedHandlers == null && this.mappedHandlerClasses == null);
    }
}
{%endcodeblock%}
- 总结
    - 提供了对于`Handler`的过滤能力,spring默认实现并没有使用
##### SimpleMappingExceptionResolver
- 代码逻辑
{%codeblock lang:java SimpleMappingExceptionResolver%}
public class SimpleMappingExceptionResolver extends AbstractHandlerExceptionResolver {
        // key为异常名,value为视图名
        private Properties exceptionMappings;
        //不处理的异常
        @Nullable
        private Class<?>[] excludedExceptions;
        //默认错误视图页面
        @Nullable
        private String defaultErrorView;

        @Nullable
        private Integer defaultStatusCode;

        private Map<String, Integer> statusCodes = new HashMap<>();

        @Nullable
        private String exceptionAttribute = DEFAULT_EXCEPTION_ATTRIBUTE;

        protected ModelAndView doResolveException(
            HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

        // Expose ModelAndView for chosen error view.
        String viewName = determineViewName(ex, request);
        if (viewName != null) {
            // Apply HTTP status code for error views, if specified.
            // Only apply it if we're processing a top-level request.
            Integer statusCode = determineStatusCode(request, viewName);
            if (statusCode != null) {
                applyStatusCodeIfPossible(request, response, statusCode);
            }
            return getModelAndView(viewName, ex, request); //创建ModelAndView,并返回
        }
        else {
            return null;
        }
    }

    protected String determineViewName(Exception ex, HttpServletRequest request) {
        String viewName = null;
        if (this.excludedExceptions != null) {
            for (Class<?> excludedEx : this.excludedExceptions) { //排除的跳过
                if (excludedEx.equals(ex.getClass())) {
                    return null;
                }
            }
        }
        // Check for specific exception mappings.
        if (this.exceptionMappings != null) { //若定义了properties,则进行匹配
            viewName = findMatchingViewName(this.exceptionMappings, ex);
        }
        // Return default error view else, if defined.
        if (viewName == null && this.defaultErrorView != null) { //未定义或未匹配到,则返回默认值
            if (logger.isDebugEnabled()) {
                logger.debug("Resolving to default view '" + this.defaultErrorView + "'");
            }
            viewName = this.defaultErrorView;
        }
        return viewName;
    }

    //---------------匹配逻辑----------------
    //简单来说就是递归确定ex以及其父类有无能够和propeties中有匹配的情况,匹配就返回viewname
    protected String findMatchingViewName(Properties exceptionMappings, Exception ex) {
        String viewName = null;
        String dominantMapping = null;
        int deepest = Integer.MAX_VALUE;
        for (Enumeration<?> names = exceptionMappings.propertyNames(); names.hasMoreElements();) {
            String exceptionMapping = (String) names.nextElement();
            int depth = getDepth(exceptionMapping, ex); //递归
            if (depth >= 0 && (depth < deepest || (depth == deepest &&
                    dominantMapping != null && exceptionMapping.length() > dominantMapping.length()))) {
                deepest = depth;
                dominantMapping = exceptionMapping;
                viewName = exceptionMappings.getProperty(exceptionMapping);
            }
        }
        if (viewName != null && logger.isDebugEnabled()) {
            logger.debug("Resolving to view '" + viewName + "' based on mapping [" + dominantMapping + "]");
        }
        return viewName;
    }
    protected int getDepth(String exceptionMapping, Exception ex) {
        return getDepth(exceptionMapping, ex.getClass(), 0);
    }

    private int getDepth(String exceptionMapping, Class<?> exceptionClass, int depth) {
        if (exceptionClass.getName().contains(exceptionMapping)) {
            // Found it!
            return depth;
        }
        // If we've gone as far as we can go and haven't found it...
        if (exceptionClass == Throwable.class) {
            return -1;
        }
        return getDepth(exceptionMapping, exceptionClass.getSuperclass(), depth + 1);
    }
}

//------------状态码----------------
protected Integer determineStatusCode(HttpServletRequest request, String viewName) {
    if (this.statusCodes.containsKey(viewName)) {
        return this.statusCodes.get(viewName);
    }
    return this.defaultStatusCode;
}
{%endcodeblock%}
##### DefaultHandlerExceptionResolver
该实现将spring提供的异常一一对应,返回对应的状态码,`MVCContainer`为空
- 代码逻辑
{%codeblock lang:java DefaultHandlerExceptionResolver%}
public class DefaultHandlerExceptionResolver extends AbstractHandlerExceptionResolver {
    protected ModelAndView doResolveException(
            HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

        try {
            if (ex instanceof HttpRequestMethodNotSupportedException) {
                return handleHttpRequestMethodNotSupported(
                        (HttpRequestMethodNotSupportedException) ex, request, response, handler);
            }
            else if (ex instanceof HttpMediaTypeNotSupportedException) {
                return handleHttpMediaTypeNotSupported(
                        (HttpMediaTypeNotSupportedException) ex, request, response, handler);
            }
            else if (ex instanceof HttpMediaTypeNotAcceptableException) {
                return handleHttpMediaTypeNotAcceptable(
                        (HttpMediaTypeNotAcceptableException) ex, request, response, handler);
            }
            else if (ex instanceof MissingPathVariableException) {
                return handleMissingPathVariable(
                        (MissingPathVariableException) ex, request, response, handler);
            }
            else if (ex instanceof MissingServletRequestParameterException) {
                return handleMissingServletRequestParameter(
                        (MissingServletRequestParameterException) ex, request, response, handler);
            }
            else if (ex instanceof ServletRequestBindingException) {
                return handleServletRequestBindingException(
                        (ServletRequestBindingException) ex, request, response, handler);
            }
            else if (ex instanceof ConversionNotSupportedException) {
                return handleConversionNotSupported(
                        (ConversionNotSupportedException) ex, request, response, handler);
            }
            else if (ex instanceof TypeMismatchException) {
                return handleTypeMismatch(
                        (TypeMismatchException) ex, request, response, handler);
            }
            else if (ex instanceof HttpMessageNotReadableException) {
                return handleHttpMessageNotReadable(
                        (HttpMessageNotReadableException) ex, request, response, handler);
            }
            else if (ex instanceof HttpMessageNotWritableException) {
                return handleHttpMessageNotWritable(
                        (HttpMessageNotWritableException) ex, request, response, handler);
            }
            else if (ex instanceof MethodArgumentNotValidException) {
                return handleMethodArgumentNotValidException(
                        (MethodArgumentNotValidException) ex, request, response, handler);
            }
            else if (ex instanceof MissingServletRequestPartException) {
                return handleMissingServletRequestPartException(
                        (MissingServletRequestPartException) ex, request, response, handler);
            }
            else if (ex instanceof BindException) {
                return handleBindException((BindException) ex, request, response, handler);
            }
            else if (ex instanceof NoHandlerFoundException) {
                return handleNoHandlerFoundException(
                        (NoHandlerFoundException) ex, request, response, handler);
            }
            else if (ex instanceof AsyncRequestTimeoutException) {
                return handleAsyncRequestTimeoutException(
                        (AsyncRequestTimeoutException) ex, request, response, handler);
            }
        }
        catch (Exception handlerEx) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failure while trying to resolve exception [" + ex.getClass().getName() + "]", handlerEx);
            }
        }
        return null;
    }
    //随便看一个
    protected ModelAndView handleHttpRequestMethodNotSupported(HttpRequestMethodNotSupportedException ex,
            HttpServletRequest request, HttpServletResponse response, @Nullable Object handler) throws IOException {

        String[] supportedMethods = ex.getSupportedMethods();
        if (supportedMethods != null) {
            response.setHeader("Allow", StringUtils.arrayToDelimitedString(supportedMethods, ", "));
        }
        //405状态码
        response.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED, ex.getMessage());
        return new ModelAndView();
    }

}
{%endcodeblock%}
- 总结
    - 不同情况对应不同状态码,默认情况`DispatcherServlet`会使用这个
##### ResponseStatusExceptionResolver
- 代码逻辑
{%codeblock lang:java ResponseStatusExceptionResolver%}
public class ResponseStatusExceptionResolver extends AbstractHandlerExceptionResolver implements MessageSourceAware {
    //含有消息源
    private MessageSource messageSource;
    @Override
    public void setMessageSource(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    protected ModelAndView doResolveException(
                HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

            try {
                if (ex instanceof ResponseStatusException) { //若抛出status异常
                    return resolveResponseStatusException((ResponseStatusException) ex, request, response, handler);
                }

                ResponseStatus status = AnnotatedElementUtils.findMergedAnnotation(ex.getClass(), ResponseStatus.class);
                if (status != null) {//若异常被@SessionStatus标记
                    return resolveResponseStatus(status, request, response, handler, ex);
                }

                if (ex.getCause() instanceof Exception) {
                    return doResolveException(request, response, handler, (Exception) ex.getCause());
                }
            }
            catch (Exception resolveEx) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Failure while trying to resolve exception [" + ex.getClass().getName() + "]", resolveEx);
                }
            }
            return null;
        }
        //code和reason可以从StatusExe或@ResponseStatus中获取
        protected ModelAndView applyStatusAndReason(int statusCode, @Nullable String reason, HttpServletResponse response)
                    throws IOException {
                //response处理
                if (!StringUtils.hasLength(reason)) {
                    response.sendError(statusCode);
                }
                else {
                    String resolvedReason = (this.messageSource != null ?
                            this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale()) :
                            reason);
                    response.sendError(statusCode, resolvedReason);
                }
                return new ModelAndView();
            }
}
{%endcodeblock%}
- 总结
    - 用于`ResponseStatusException`和被`@ResponseStaues`标记的注解
    - `DispatcherServlet`默认异常处理器之一
##### ExceptionHandlerExceptionResolver
最常见的异常处理器
- 代码逻辑
{%codeblock lang:java ExceptionHandlerExceptionResolver%}
public abstract class AbstractHandlerMethodExceptionResolver extends AbstractHandlerExceptionResolver {
    //重写了applyto逻辑,保证了该处理器处理的handler一定是MethodHandler类型
    protected boolean shouldApplyTo(HttpServletRequest request, @Nullable Object handler) {
        if (handler == null) {
            return super.shouldApplyTo(request, null);
        }
        else if (handler instanceof HandlerMethod) {
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            handler = handlerMethod.getBean();
            return super.shouldApplyTo(request, handler);
        }
        else {
            return false;
    }
}
protected final ModelAndView doResolveException(
        HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

    return doResolveHandlerMethodException(request, response, (HandlerMethod) handler, ex);
}
protected abstract ModelAndView doResolveHandlerMethodException(
            HttpServletRequest request, HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception ex);
}
//子类
public class ExceptionHandlerExceptionResolver extends AbstractHandlerMethodExceptionResolver
        implements ApplicationContextAware, InitializingBean {
            //内部组件和RequestMappingAdaptor很相似
            private List<HandlerMethodArgumentResolver> customArgumentResolvers;

@Nullable
private HandlerMethodArgumentResolverComposite argumentResolvers;

@Nullable
private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;

@Nullable
private HandlerMethodReturnValueHandlerComposite returnValueHandlers;

private List<HttpMessageConverter<?>> messageConverters;

private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();

private final List<Object> responseBodyAdvice = new ArrayList<>();

private ApplicationContext applicationContext;

private final Map<Class<?>, ExceptionHandlerMethodResolver> exceptionHandlerCache =
        new ConcurrentHashMap<>(64);
//初始化确定
private final Map<ControllerAdviceBean, ExceptionHandlerMethodResolver> exceptionHandlerAdviceCache =
        new LinkedHashMap<>();
//---------------------始化-------------------------
public void afterPropertiesSet() {
        // Do this first, it may add ResponseBodyAdvice beans
        initExceptionHandlerAdviceCache();

        if (this.argumentResolvers == null) {
            List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
            this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
        }
        if (this.returnValueHandlers == null) {
            List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
            this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
        }
    }

    private void initExceptionHandlerAdviceCache() {
        if (getApplicationContext() == null) {
            return;
        }
        //获取@Controlleradvice类,并转换为ControllerAdviceBean
        List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
        for (ControllerAdviceBean adviceBean : adviceBeans) {
            Class<?> beanType = adviceBean.getBeanType();
            if (beanType == null) {
                throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
            }
            //resolver用来存方法@ExceptionHandler函数,并提供匹配函数
            ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(beanType);
            if (resolver.hasExceptionMappings()) {
                this.exceptionHandlerAdviceCache.put(adviceBean, resolver);
            }
            if (ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
                this.responseBodyAdvice.add(adviceBean);
            }
        }

        if (logger.isDebugEnabled()) {
            int handlerSize = this.exceptionHandlerAdviceCache.size();
            int adviceSize = this.responseBodyAdvice.size();
            if (handlerSize == 0 && adviceSize == 0) {
                logger.debug("ControllerAdvice beans: none");
            }
            else {
                logger.debug("ControllerAdvice beans: " +
                        handlerSize + " @ExceptionHandler, " + adviceSize + " ResponseBodyAdvice");
            }
        }
    }
}            
//-------------------处理异常----------------------
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
            HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {
        //[1]根据异常处理函数创建一个ServletIncocableHandlerMethod
        ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
        if (exceptionHandlerMethod == null) {
            return null;
        }
        //[1!]
        //[2]设置入参和回参处理器
        if (this.argumentResolvers != null) {
            exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        //[2!]

        //[3]调用invoke,这里又用到了invoke(arg...)可变参数,将exception,cause,handlerMethod传递过去
        //另个用到这种是InintFactory传递dataBinder的时候
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();

        try {
            if (logger.isDebugEnabled()) {
                logger.debug("Using @ExceptionHandler " + exceptionHandlerMethod);
            }
            Throwable cause = exception.getCause();
            if (cause != null) {
                // Expose cause as provided argument as well
                exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, cause, handlerMethod);
            }
            else {
                // Otherwise, just the given exception as-is
                exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, handlerMethod);
            }
        }
        catch (Throwable invocationEx) {
            // Any other than the original exception (or its cause) is unintended here,
            // probably an accident (e.g. failed assertion or the like).
            if (invocationEx != exception && invocationEx != exception.getCause() && logger.isWarnEnabled()) {
                logger.warn("Failure in @ExceptionHandler " + exceptionHandlerMethod, invocationEx);
            }
            // Continue with default processing of the original exception...
            return null;
        }

        if (mavContainer.isRequestHandled()) {//如果异常处理器结束了处理,则返回一个空白,基本发生于rest情况,即出现@ResponseBody注解
            return new ModelAndView();
        }
        else {//返回完整视图容器
            ModelMap model = mavContainer.getModel();
            HttpStatus status = mavContainer.getStatus();
            ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
            mav.setViewName(mavContainer.getViewName());
            if (!mavContainer.isViewReference()) {
                mav.setView((View) mavContainer.getView());
            }
            if (model instanceof RedirectAttributes) {
                Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
                RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
            }
            return mav;
        }
    }
    //--------------根据handlerMethod创建SIHM---------------------
    protected ServletInvocableHandlerMethod getExceptionHandlerMethod(
            @Nullable HandlerMethod handlerMethod, Exception exception) {

        Class<?> handlerType = null;

        if (handlerMethod != null) {//从含有本handlerMethod中寻找,并创建
            // Local exception handler methods on the controller class itself.
            // To be invoked through the proxy, even in case of an interface-based proxy.
            handlerType = handlerMethod.getBeanType();
            ExceptionHandlerMethodResolver resolver = this.exceptionHandlerCache.get(handlerType);
            if (resolver == null) {
                resolver = new ExceptionHandlerMethodResolver(handlerType);
                this.exceptionHandlerCache.put(handlerType, resolver);
            }
            Method method = resolver.resolveMethod(exception);
            if (method != null) {
                return new ServletInvocableHandlerMethod(handlerMethod.getBean(), method);
            }
            // For advice applicability check below (involving base packages, assignable types
            // and annotation presence), use target class instead of interface-based proxy.
            if (Proxy.isProxyClass(handlerType)) {
                handlerType = AopUtils.getTargetClass(handlerMethod.getBean());
            }
        }
        //从全局advice中匹配并创建
        for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {
            ControllerAdviceBean advice = entry.getKey();
            if (advice.isApplicableToBeanType(handlerType)) {
                ExceptionHandlerMethodResolver resolver = entry.getValue();
                Method method = resolver.resolveMethod(exception);
                if (method != null) {
                    return new ServletInvocableHandlerMethod(advice.resolveBean(), method);
                }
            }
        }

        return null;
    }
{%endcodeblock%}
- 总结
    - `@ExecptionHandler`的调用实际被转换成了`ServletInvokeHandlerMethod`
    - `@ExecptionHandler`可以使用在Controller,或者`@ControllerAdvice`
    - 参数可以为excption,cause,handlerMethod
    - 入参和回参则决定于该处理器中的入参和回参处理器
###### ExceptionHandlerMethodResolver
用来解析`@ExceptionHandler`
- 代码
{%codeblock lang:java ExceptionHandlerMethodResolver%}
public class ExceptionHandlerMethodResolver {
    //======================属性===========================
    public static final MethodFilter EXCEPTION_HANDLER_METHODS = method ->
            AnnotatedElementUtils.hasAnnotation(method, ExceptionHandler.class);

    private final Map<Class<? extends Throwable>, Method> mappedMethods = new HashMap<>(16);

    private final Map<Class<? extends Throwable>, Method> exceptionLookupCache = new ConcurrentReferenceHashMap<>(16);
    //====================构造初始化==========================
    public ExceptionHandlerMethodResolver(Class<?> handlerType) {
        //handlerType中获取@ExceptionHandler函数
        for (Method method : MethodIntrospector.selectMethods(handlerType, EXCEPTION_HANDLER_METHODS)) {
            for (Class<? extends Throwable> exceptionType : detectExceptionMappings(method)) {//判断该函数与其匹配的异常
                addExceptionMapping(exceptionType, method);
            }
        }
    }

    //--------------异常匹配-----------------
    private List<Class<? extends Throwable>> detectExceptionMappings(Method method) {
        List<Class<? extends Throwable>> result = new ArrayList<>();
        detectAnnotationExceptionMappings(method, result);
        if (result.isEmpty()) {//说明@ExceptionHandler中没有value值
            for (Class<?> paramType : method.getParameterTypes()) { //选择method形参作为异常匹配
                if (Throwable.class.isAssignableFrom(paramType)) {
                    result.add((Class<? extends Throwable>) paramType);
                }
            }
        }
        if (result.isEmpty()) {
            throw new IllegalStateException("No exception types mapped to " + method);
        }
        return result;
    }
  //获取方法中@ExceptionHandler#value作为异常值
    private void detectAnnotationExceptionMappings(Method method, List<Class<? extends Throwable>> result) {
        ExceptionHandler ann = AnnotatedElementUtils.findMergedAnnotation(method, ExceptionHandler.class);
        Assert.state(ann != null, "No ExceptionHandler annotation");
        result.addAll(Arrays.asList(ann.value()));
    }
    //-------------缓存--------------------
    private void addExceptionMapping(Class<? extends Throwable> exceptionType, Method method) {
        Method oldMethod = this.mappedMethods.put(exceptionType, method);
        if (oldMethod != null && !oldMethod.equals(method)) {
            throw new IllegalStateException("Ambiguous @ExceptionHandler method mapped for [" +
                    exceptionType + "]: {" + oldMethod + ", " + method + "}");
        }
    }

//===========================解析====================================
public Method resolveMethod(Exception exception) {
        return resolveMethodByThrowable(exception);
    }
    public Method resolveMethodByThrowable(Throwable exception) {
            Method method = resolveMethodByExceptionType(exception.getClass());
            if (method == null) {
                Throwable cause = exception.getCause();
                if (cause != null) {
                    method = resolveMethodByExceptionType(cause.getClass());
                }
            }
            return method;
        }
        public Method resolveMethodByExceptionType(Class<? extends Throwable> exceptionType) {
                Method method = this.exceptionLookupCache.get(exceptionType);
                if (method == null) {
                    method = getMappedMethod(exceptionType);
                    this.exceptionLookupCache.put(exceptionType, method);
                }
                return method;
            }
            //由于初始化的逻辑会导致,多个excption对应一个method,因此此处做了一个排序
            private Method getMappedMethod(Class<? extends Throwable> exceptionType) {
                    List<Class<? extends Throwable>> matches = new ArrayList<>();
                    for (Class<? extends Throwable> mappedException : this.mappedMethods.keySet()) {
                        if (mappedException.isAssignableFrom(exceptionType)) {
                            matches.add(mappedException);
                        }
                    }
                    if (!matches.isEmpty()) {
                        //排序,简单来说就看匹配的excpetion和发生的异常是否类型更加接近
                        matches.sort(new ExceptionDepthComparator(exceptionType));
                        return this.mappedMethods.get(matches.get(0));
                    }
                    else {
                        return null;
                    }
                }
}
{%endcodeblock%}
