---
title: springMVC配置二
cover: /img/spring.png
top_img: /img/post.jpg
date: 2020-04-03 20:18:16
tags:
- 框架
- spring
- springMvc
categories: java
description: springMVC静态资源配置
---
此部分关于描述springmvc处理静态资源的配置,关于资源参考[springio](/2020/02/11/spring常见/#resource接口)
# default-servlet-handler
用来处理`defaultServlet`的访问
## xml
### xml解析代码分析
{%codeblock lang:java DefaultServletHandlerBeanDefinitionParser%}
class DefaultServletHandlerBeanDefinitionParser implements BeanDefinitionParser {
    @Override
    @Nullable
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        Object source = parserContext.extractSource(element);

        String defaultServletName = element.getAttribute("default-servlet-name");
    //创建一个DefaultServletHttpRequestHandler
        RootBeanDefinition defaultServletHandlerDef = new RootBeanDefinition(DefaultServletHttpRequestHandler.class);
        defaultServletHandlerDef.setSource(source);
        defaultServletHandlerDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        if (StringUtils.hasText(defaultServletName)) {
            defaultServletHandlerDef.getPropertyValues().add("defaultServletName", defaultServletName);
        }
        String defaultServletHandlerName = parserContext.getReaderContext().generateBeanName(defaultServletHandlerDef);
        parserContext.getRegistry().registerBeanDefinition(defaultServletHandlerName, defaultServletHandlerDef);
        parserContext.registerComponent(new BeanComponentDefinition(defaultServletHandlerDef, defaultServletHandlerName));

        Map<String, String> urlMap = new ManagedMap<>();
        urlMap.put("/**", defaultServletHandlerName);

    //创建SimpleUrlHandlerMapping,并设置urlMap
        RootBeanDefinition handlerMappingDef = new RootBeanDefinition(SimpleUrlHandlerMapping.class);
        handlerMappingDef.setSource(source);
        handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        handlerMappingDef.getPropertyValues().add("urlMap", urlMap);

        String handlerMappingBeanName = parserContext.getReaderContext().generateBeanName(handlerMappingDef);
        parserContext.getRegistry().registerBeanDefinition(handlerMappingBeanName, handlerMappingDef);
        parserContext.registerComponent(new BeanComponentDefinition(handlerMappingDef, handlerMappingBeanName));

        // Ensure BeanNameUrlHandlerMapping (SPR-8289) and default HandlerAdapters are not "turned off"
        MvcNamespaceUtils.registerDefaultComponents(parserContext, source);

        return null;
    }

}
{%endcodeblock%}
## 注解
## 实现源码
根据`default-servlet-handler`创建一个`SimpleUrlHandlerMapping`并包含`DefaultServletHttpRequestHandler`
- DefaultServletHttpRequestHandler:是`HttpRequestHandler`的实现之一,访问默认servlet,根据不同的web环境,defualtServlet也不同
{%codeblock lang:java DefaultServletHttpRequestHandler%}
public class DefaultServletHttpRequestHandler implements HttpRequestHandler, ServletContextAware {
  public void setServletContext(ServletContext servletContext) {
        this.servletContext = servletContext;
        if (!StringUtils.hasText(this.defaultServletName)) { //判断不同环境访问不同
            if (this.servletContext.getNamedDispatcher(COMMON_DEFAULT_SERVLET_NAME) != null) {
                this.defaultServletName = COMMON_DEFAULT_SERVLET_NAME;
            }
            else if (this.servletContext.getNamedDispatcher(GAE_DEFAULT_SERVLET_NAME) != null) {
                this.defaultServletName = GAE_DEFAULT_SERVLET_NAME;
            }
            else if (this.servletContext.getNamedDispatcher(RESIN_DEFAULT_SERVLET_NAME) != null) {
                this.defaultServletName = RESIN_DEFAULT_SERVLET_NAME;
            }
            else if (this.servletContext.getNamedDispatcher(WEBLOGIC_DEFAULT_SERVLET_NAME) != null) {
                this.defaultServletName = WEBLOGIC_DEFAULT_SERVLET_NAME;
            }
            else if (this.servletContext.getNamedDispatcher(WEBSPHERE_DEFAULT_SERVLET_NAME) != null) {
                this.defaultServletName = WEBSPHERE_DEFAULT_SERVLET_NAME;
            }
            else {
                throw new IllegalStateException("Unable to locate the default servlet for serving static content. " +
                        "Please set the 'defaultServletName' property explicitly.");
            }
        }
    }


    @Override
    public void handleRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        Assert.state(this.servletContext != null, "No ServletContext set");
        RequestDispatcher rd = this.servletContext.getNamedDispatcher(this.defaultServletName);
        if (rd == null) {
            throw new IllegalStateException("A RequestDispatcher could not be located for the default servlet '" +
                    this.defaultServletName + "'");
        }
    //forward实现
        rd.forward(request, response);
    }

}  
{%endcodeblock%}
# resources
## xml
### xml配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd">
        <mvc:resources mapping="/**" location="/**">
            <mvc:resource-chain resource-cache="true" >
                <mvc:resolvers>
                    <mvc:version-resolver>
                        <!-- 实际上这内部也支持用户自定义 -->
                        <mvc:content-version-strategy patterns="/**"/>
                        <mvc:fixed-version-strategy version="1.0" patterns="/fiexedVersion/**"/>
                    </mvc:version-resolver>
                </mvc:resolvers>
            </mvc:resource-chain>
        </mvc:resources>
</beans>
```
- 总结
    - `mvc:cache-control`:mapping表示url映射,location表示本地资源路径
    - `cache-control`:控制`CacheControl`属性,该类的使用参考[mvc缓存处理](/2019/12/26/springMVC核心/#缓存机制)
    - `resource-chain`:设置内部的资源处理链
        - `version-resolver`:用来设置`VersionResourceResolver`的配置
        - `resource-cache`:开启`CachingResourceResolver`

### xml解析源码分析
## java配置
- `ResourceHandlerRegistry`:包含`ResourceHandlerRegistration`和`ResourceChainRegistration`
- `ResourceChainRegistration`:设置`ResourceHttpRequestHandler`的资源处理链
- `ResourceChainRegistration`:对应资源处理器

```java
public void addResourceHandlers(ResourceHandlerRegistry registry) {
        ResourceHandlerRegistration resourceHandlerRegistration = registry.addResourceHandler("/**");
        resourceHandlerRegistration.addResourceLocations( "/");
        //开启缓存资源解析器
        ResourceChainRegistration resourceChainRegistration= resourceHandlerRegistration.resourceChain(true);
        //开启version资源解析器
        VersionResourceResolver versionResourceResolver = new VersionResourceResolver();
        versionResourceResolver.addFixedVersionStrategy("1.0", "/fixedVersion/**");
        versionResourceResolver.addContentVersionStrategy("/md5Version/**");
        resourceChainRegistration.addResolver(versionResourceResolver);
    }
```

这种代码可以写成连续的语法`registry.addResourceLocations( "/").resourceChain(true).addFixedVersionStrategy("1.0"."fiexedVersion")`
## 实现原理
### ResourceHttpRequestHandler
{%asset_img ResourceHttpRequestHandler.png%}
- 代码逻辑
{%codeblock lang:java ResourceHttpRequestHandler%}
private static final Log logger = LogFactory.getLog(ResourceHttpRequestHandler.class);

    private static final String URL_RESOURCE_CHARSET_PREFIX = "[charset=";

  //通过配置设置的本地资源位置
    private final List<String> locationValues = new ArrayList<>(4);
  //通过locationValues解析的资源
    private final List<Resource> locations = new ArrayList<>(4);
  //编码资源,那么也可以看出来这里解析支持仅仅是文本资源
    private final Map<Resource, Charset> locationCharsets = new HashMap<>(4);
  //资源解析器
    private final List<ResourceResolver> resourceResolvers = new ArrayList<>(4);

    private final List<ResourceTransformer> resourceTransformers = new ArrayList<>(4);
  //资源解析器链(责任链模式)
    @Nullable
    private ResourceResolverChain resolverChain;

    @Nullable
    private ResourceTransformerChain transformerChain;
  //资源转换器
    @Nullable
    private ResourceHttpMessageConverter resourceHttpMessageConverter;
  //region资源转换器
    @Nullable
    private ResourceRegionHttpMessageConverter resourceRegionHttpMessageConverter;

    @Nullable
    private ContentNegotiationManager contentNegotiationManager;

    @Nullable
    private PathExtensionContentNegotiationStrategy contentNegotiationStrategy;

    @Nullable
    private CorsConfiguration corsConfiguration;

    @Nullable
    private UrlPathHelper urlPathHelper;

  //转换#{属性},由aware接口实现
    @Nullable
    private StringValueResolver embeddedValueResolver;


    public ResourceHttpRequestHandler() {
        super(HttpMethod.GET.name(), HttpMethod.HEAD.name());
    }
  /==============================初始化==========================================
  public void afterPropertiesSet() throws Exception {
      //[1]通过location初始化文本资源
          resolveResourceLocations();
      //[1!]
          if (logger.isWarnEnabled() && CollectionUtils.isEmpty(this.locations)) {
              logger.warn("Locations list is empty. No resources will be served unless a " +
                      "custom ResourceResolver is configured as an alternative to PathResourceResolver.");
          }
      //[2!]至少添加一个path处理器
          if (this.resourceResolvers.isEmpty()) {
              this.resourceResolvers.add(new PathResourceResolver());
          }
      //[2!]
      //[3]
          initAllowedLocations();
      //[3!]  

      //[4]初始化其他必要变量
          this.resolverChain = new DefaultResourceResolverChain(this.resourceResolvers);
          this.transformerChain = new DefaultResourceTransformerChain(this.resolverChain, this.resourceTransformers);

          if (this.resourceHttpMessageConverter == null) {
              this.resourceHttpMessageConverter = new ResourceHttpMessageConverter();
          }
          if (this.resourceRegionHttpMessageConverter == null) {
              this.resourceRegionHttpMessageConverter = new ResourceRegionHttpMessageConverter();
          }

          this.contentNegotiationStrategy = initContentNegotiationStrategy();
      //[4!]
      }
    //-------------------根据locations解析资源-------------------------
    private void resolveResourceLocations() {
        if (CollectionUtils.isEmpty(this.locationValues)) {
            return;
        }
        else if (!CollectionUtils.isEmpty(this.locations)) {
            throw new IllegalArgumentException("Please set either Resource-based \"locations\" or " +
                    "String-based \"locationValues\", but not both.");
        }

        ApplicationContext applicationContext = obtainApplicationContext();
        for (String location : this.locationValues) {
            if (this.embeddedValueResolver != null) {//处理${}
                String resolvedLocation = this.embeddedValueResolver.resolveStringValue(location);
                if (resolvedLocation == null) {
                    throw new IllegalArgumentException("Location resolved to null: " + location);
                }
                location = resolvedLocation;
            }
            Charset charset = null;
            location = location.trim();
            if (location.startsWith(URL_RESOURCE_CHARSET_PREFIX)) { //如果带有[charset=xx]前缀,则保存一份带有编码的map
                int endIndex = location.indexOf(']', URL_RESOURCE_CHARSET_PREFIX.length());
                if (endIndex == -1) {
                    throw new IllegalArgumentException("Invalid charset syntax in location: " + location);
                }
                String value = location.substring(URL_RESOURCE_CHARSET_PREFIX.length(), endIndex);
                charset = Charset.forName(value);
                location = location.substring(endIndex + 1);
            }
            Resource resource = applicationContext.getResource(location);
            this.locations.add(resource);
            if (charset != null) {
                if (!(resource instanceof UrlResource)) {
                    throw new IllegalArgumentException("Unexpected charset for non-UrlResource: " + resource);
                }
                this.locationCharsets.put(resource, charset); //带有编码的map
            }
        }
    }
  //-----------------------正确解析的文本资源添加到PathResourceResolver中-----------------------------
  protected void initAllowedLocations() {
        if (CollectionUtils.isEmpty(this.locations)) {
            return;
        }
        for (int i = getResourceResolvers().size() - 1; i >= 0; i--) {
            if (getResourceResolvers().get(i) instanceof PathResourceResolver) {
                PathResourceResolver pathResolver = (PathResourceResolver) getResourceResolvers().get(i);
                if (ObjectUtils.isEmpty(pathResolver.getAllowedLocations())) {
                    pathResolver.setAllowedLocations(getLocations().toArray(new Resource[0]));
                }
                if (this.urlPathHelper != null) {
                    pathResolver.setLocationCharsets(this.locationCharsets);
                    pathResolver.setUrlPathHelper(this.urlPathHelper);
                }
                break;
            }
        }
    }
  //================================逻辑处理========================================
  public void handleRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        // For very general mappings (e.g. "/") we need to check 404 first
        Resource resource = getResource(request);
        if (resource == null) {
            logger.debug("Resource not found");
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }
    //[1!]
    //[2]options方法则返回mvc支持的方法
        if (HttpMethod.OPTIONS.matches(request.getMethod())) {
            response.setHeader("Allow", getAllowHeader());
            return;
        }
    //[2!]
        // Supported methods and required session
        checkRequest(request);

        // Header phase
        //这里根据判断资源是否过期
        if (new ServletWebRequest(request, response).checkNotModified(resource.lastModified())) {
            logger.trace("Resource not modified");
            return;
        }

        // Apply cache settings, if any
    //处理缓存机制
        prepareResponse(response);
    //[3]head方法表示仅仅返回响应头
        // Check the media type for the resource
        MediaType mediaType = getMediaType(request, resource);

        // Content phase
        if (METHOD_HEAD.equals(request.getMethod())) {
            setHeaders(response, resource, mediaType);
            return;
        }
    //[3!]

    //[4] 转化器来写出资源
        ServletServerHttpResponse outputMessage = new ServletServerHttpResponse(response);
        if (request.getHeader(HttpHeaders.RANGE) == null) { //非range资源
            Assert.state(this.resourceHttpMessageConverter != null, "Not initialized");
            setHeaders(response, resource, mediaType);
            this.resourceHttpMessageConverter.write(resource, mediaType, outputMessage);
        }
        else {//range资源
            Assert.state(this.resourceRegionHttpMessageConverter != null, "Not initialized");
            response.setHeader(HttpHeaders.ACCEPT_RANGES, "bytes");
            ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(request);
            try {
                List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
                response.setStatus(HttpServletResponse.SC_PARTIAL_CONTENT);
                this.resourceRegionHttpMessageConverter.write(
                        HttpRange.toResourceRegions(httpRanges, resource), mediaType, outputMessage);
            }
            catch (IllegalArgumentException ex) {
                response.setHeader("Content-Range", "bytes */" + resource.contentLength());
                response.sendError(HttpServletResponse.SC_REQUESTED_RANGE_NOT_SATISFIABLE);
            }
        }
    //[4!]
    }
{%endcodeblock%}
- 总结
  - 通过解析器来处理资源
  - 通过转换器写出资源
  - 处理了缓存头,options,head方法,关于处理缓存头部分参考[mvc缓存处理](/2019/12/26/springMVC核心/#缓存机制)
  - locations变量用来表示本地资源路径
### 资源处理责任链
实际内部采取责任链模式实际获取资源`Resource`
#### 责任链实现
`DefaultResourceResolverChain`是典型的责任链模型
{%codeblock lang:java %}
class DefaultResourceResolverChain implements ResourceResolverChain {

    @Nullable
    private final ResourceResolver resolver;

    @Nullable
    private final ResourceResolverChain nextChain;

    //简单来说就是通过list创建一个链表,next表示下一个节点
    public DefaultResourceResolverChain(@Nullable List<? extends ResourceResolver> resolvers) {
        resolvers = (resolvers != null ? resolvers : Collections.emptyList());
        DefaultResourceResolverChain chain = initChain(new ArrayList<>(resolvers));
        this.resolver = chain.resolver;
        this.nextChain = chain.nextChain;
    }

    private static DefaultResourceResolverChain initChain(ArrayList<? extends ResourceResolver> resolvers) {
        DefaultResourceResolverChain chain = new DefaultResourceResolverChain(null, null);
        ListIterator<? extends ResourceResolver> it = resolvers.listIterator(resolvers.size());
        //创建链条
        while (it.hasPrevious()) {
            chain = new DefaultResourceResolverChain(it.previous(), chain);
        }
        return chain;
    }

    private DefaultResourceResolverChain(@Nullable ResourceResolver resolver, @Nullable ResourceResolverChain chain) {
        Assert.isTrue((resolver == null && chain == null) || (resolver != null && chain != null),
                "Both resolver and resolver chain must be null, or neither is");
        this.resolver = resolver;
        this.nextChain = chain;
    }


    @Override
    @Nullable
    public Resource resolveResource(
            @Nullable HttpServletRequest request, String requestPath, List<? extends Resource> locations) {

        return (this.resolver != null && this.nextChain != null ?
                this.resolver.resolveResource(request, requestPath, locations, this.nextChain) : null);
    }

    @Override
    @Nullable
    public String resolveUrlPath(String resourcePath, List<? extends Resource> locations) {
        return (this.resolver != null && this.nextChain != null ?
                this.resolver.resolveUrlPath(resourcePath, locations, this.nextChain) : null);
    }

}
{%endcodeblock%}
- 总结
    - 链条顺序:和构造器`resolvers`顺序一致
#### 资源解析器
`ResourceResolver` spring内一贯的`Resolver`
{%asset_img ResourceResolver.png%}
- 抽象逻辑
{%codeblock lang:java AbstractResourceResolver%}
public abstract class AbstractResourceResolver implements ResourceResolver {

    protected final Log logger = LogFactory.getLog(getClass());
    @Override
    @Nullable
    public Resource resolveResource(@Nullable HttpServletRequest request, String requestPath,
            List<? extends Resource> locations, ResourceResolverChain chain) {
        //子类实现
        return resolveResourceInternal(request, requestPath, locations, chain);
    }

    @Override
    @Nullable
    public String resolveUrlPath(String resourceUrlPath, List<? extends Resource> locations,
            ResourceResolverChain chain) {
        //调用子类实现
        return resolveUrlPathInternal(resourceUrlPath, locations, chain);
    }


    @Nullable
    protected abstract Resource resolveResourceInternal(@Nullable HttpServletRequest request,
            String requestPath, List<? extends Resource> locations, ResourceResolverChain chain);

    @Nullable
    protected abstract String resolveUrlPathInternal(String resourceUrlPath,
            List<? extends Resource> locations, ResourceResolverChain chain);

}
{%endcodeblock%}
##### CachingResourceResolver
通过spring缓存机制实现,参考[spring缓存](/2020/04/01/spring-cache/#spring缓存)
根据`Accpect-Encoding`和请求路径判断是否命中缓存,内部`Cache`缓存`Resource`
```java
protected Resource resolveResourceInternal(@Nullable HttpServletRequest request, String requestPath,
    List<? extends Resource> locations, ResourceResolverChain chain) {

  String key = computeKey(request, requestPath);
  //尝试获取缓存
  Resource resource = this.cache.get(key, Resource.class);

  if (resource != null) {
    if (logger.isTraceEnabled()) {
      logger.trace("Resource resolved from cache");
    }
    return resource;
  }
  //nextChain获取
  resource = chain.resolveResource(request, requestPath, locations);
  if (resource != null) {
    this.cache.put(key, resource);//缓存
  }

  return resource;
}
```
##### VersionResourceResolver
通过url带上版本信息,以及`Etag`来实现新鲜度,本身`ResourceHttpRequestHandler`已经有根据修改时间来判断新鲜度了,
这个解析器实际上就是通过按照Etag来判断的功能,etag来自于固定版本和md5.
- version url格式
    - `{version}/static/myresource.js`,我把前者叫做version类型
    - `static/myresource-{MD5}.js`,后者为md5类型
- 缓存逻辑
固定版本如`1.0/static/test.js`,当第一次客户端访问,如果`VersionResourceResolver`能够返回那么,客户端就会
对该资源带上`Etag:1.0`,以后访问的时候`ResourceHttpRequestHandler`会判断一次`etag`,决定是否继续解析.
当出现版本更新时,服务器应该改变{vserion}为新的,至于资源文件则由服务器更新,页面url也要更新
MD5实际逻辑是一样
- 代码逻辑
{%codeblock lang:java VersionResourceResolver%}
public class VersionResourceResolver extends AbstractResourceResolver {
    //Ant路径匹配器
    private AntPathMatcher pathMatcher = new AntPathMatcher();
    /** Map from path pattern -> VersionStrategy. */
    //key为url路径,value为VersionStrategy
    private final Map<String, VersionStrategy> versionStrategyMap = new LinkedHashMap<>();
    //添加md5类型
    public VersionResourceResolver addContentVersionStrategy(String... pathPatterns) {
            addVersionStrategy(new ContentVersionStrategy(), pathPatterns);
            return this;
        }
        //添加version类型
        public VersionResourceResolver addFixedVersionStrategy(String version, String... pathPatterns) {
                List<String> patternsList = Arrays.asList(pathPatterns);
                List<String> prefixedPatterns = new ArrayList<>(pathPatterns.length);
                String versionPrefix = "/" + version;
                for (String pattern : patternsList) {
                    prefixedPatterns.add(pattern);
                    if (!pattern.startsWith(versionPrefix) && !patternsList.contains(versionPrefix + pattern)) {
                        prefixedPatterns.add(versionPrefix + pattern);
                    }
                }
                return addVersionStrategy(new FixedVersionStrategy(version), StringUtils.toStringArray(prefixedPatterns));
            }
//-----------------------解析逻辑----------------------------------
            protected Resource resolveResourceInternal(@Nullable HttpServletRequest request, String requestPath,
                        List<? extends Resource> locations, ResourceResolverChain chain) {
                    //[1]优先执行next
                    Resource resolved = chain.resolveResource(request, requestPath, locations);
                    if (resolved != null) {
                        return resolved;
                    }
                    //[1!]
                    //[2]根据路径获取VS,Ant路径匹配
                    VersionStrategy versionStrategy = getStrategyForPath(requestPath);
                    if (versionStrategy == null) {
                        return null;
                    }
                    //[2!]
                    //[3]尝试获取请求中存在的版本信息
                    String candidateVersion = versionStrategy.extractVersion(requestPath);
                    if (!StringUtils.hasLength(candidateVersion)) {
                        return null;
                    }
                    //[3!]
                    //[4]首先尝试获取基础版本,即不带版本信息的    
                    //从路径中删除版本信息
                    String simplePath = versionStrategy.removeVersion(requestPath, candidateVersion);
                    //获取基础资源,由于ResourceHttpRequestHandler#afterPropertiesSet的实现
                    //因此责任链会有一个兜底的PathResourceResolver
                    Resource baseResource = chain.resolveResource(request, simplePath, locations);
                    if (baseResource == null) {
                        return null;
                    }
                    //[4!]
                    //[5]根据基础资源生成版本号
                    String actualVersion = versionStrategy.getResourceVersion(baseResource);
                    if (candidateVersion.equals(actualVersion)) {//当客户端发送的版本信息和当前资源能够匹配则返回
                        return new FileNameVersionedResource(baseResource, candidateVersion);//返回VersionResource,这是一个HttpResource
                    }
                    else {
                        if (logger.isTraceEnabled()) {
                            logger.trace("Found resource for \"" + requestPath + "\", but version [" +
                                    candidateVersion + "] does not match");
                        }
                        return null;
                    }
                    //[5!]
                }
//====================内部类===============================
private class FileNameVersionedResource extends AbstractResource implements HttpResource {

        private final Resource original;

        private final String version;

        public FileNameVersionedResource(Resource original, String version) {
            this.original = original;
            this.version = version;
        }
        //这个HttpResource设置了一个etag头信息
        public HttpHeaders getResponseHeaders() {
            HttpHeaders headers = (this.original instanceof HttpResource ?
                    ((HttpResource) this.original).getResponseHeaders() : new HttpHeaders());
            headers.setETag("\"" + this.version + "\"");
            return headers;
        }
}
{%endcodeblock%}
- 总结

###### VersionStrategy
此类为`VersionResourceResolver`中的实现核心,用来返回`version`字符串,默认实现为两种
{%asset_img VersionStrategy.png%}
- 抽象逻辑
这个类设计成这样我是觉得不好
{%codeblock lang:java AbstractVersionStrategy%}
public String extractVersion(String requestPath) {
    private final VersionPathStrategy pathStrategy;//实际为内部类
        return this.pathStrategy.extractVersion(requestPath);
    }

    @Override
    public String removeVersion(String requestPath, String version) {
        return this.pathStrategy.removeVersion(requestPath, version);
    }

    @Override
    public String addVersion(String requestPath, String version) {
        return this.pathStrategy.addVersion(requestPath, version);
    }
    //用来处理`{version}/xx.资源后缀`这种类型
    protected static class PrefixVersionPathStrategy implements VersionPathStrategy {

            private final String prefix;

            public PrefixVersionPathStrategy(String version) {
                Assert.hasText(version, "Version must not be empty");
                this.prefix = version;
            }

            @Override
            @Nullable
            public String extractVersion(String requestPath) { //返回资源附带的前缀{version}部分
                return (requestPath.startsWith(this.prefix) ? this.prefix : null);
            }

            @Override
            public String removeVersion(String requestPath, String version) {
                return requestPath.substring(this.prefix.length());
            }

            @Override
            public String addVersion(String path, String version) {
                if (path.startsWith(".")) {
                    return path;
                }
                else {
                    return (this.prefix.endsWith("/") || path.startsWith("/") ?
                            this.prefix + path : this.prefix + '/' + path);
                }
            }
        }
        protected static class FileNameVersionPathStrategy implements VersionPathStrategy {

                private static final Pattern pattern = Pattern.compile("-(\\S*)\\.");

                @Override
                @Nullable
                //使用正则获取-{MD5}版本信息
                public String extractVersion(String requestPath) {
                    Matcher matcher = pattern.matcher(requestPath);
                    if (matcher.find()) {
                        String match = matcher.group(1);
                        return (match.contains("-") ? match.substring(match.lastIndexOf('-') + 1) : match);
                    }
                    else {
                        return null;
                    }
                }

                @Override
                public String removeVersion(String requestPath, String version) {
                    return StringUtils.delete(requestPath, "-" + version);
                }

                @Override
                public String addVersion(String requestPath, String version) {
                    String baseFilename = StringUtils.stripFilenameExtension(requestPath);
                    String extension = StringUtils.getFilenameExtension(requestPath);
                    return (baseFilename + '-' + version + '.' + extension);
                }
            }
{%endcodeblock%}
- ContentVersionStrategy
{%codeblock lang:java ContentVersionStrategy%}
public class ContentVersionStrategy extends AbstractVersionStrategy {

    public ContentVersionStrategy() {
        super(new FileNameVersionPathStrategy());
    }
    //生成版本信息
    @Override
    public String getResourceVersion(Resource resource) {
        try {
            byte[] content = FileCopyUtils.copyToByteArray(resource.getInputStream());
            return DigestUtils.md5DigestAsHex(content);//生成md5摘要
        }
        catch (IOException ex) {
            throw new IllegalStateException("Failed to calculate hash for " + resource, ex);
        }
    }

}
{%endcodeblock%}
- FixedVersionStrategy
{%codeblock lang:java FixedVersionStrategy%}
public class FixedVersionStrategy extends AbstractVersionStrategy {

    private final String version;


    /**
     * Create a new FixedVersionStrategy with the given version string.
     * @param version the fixed version string to use
     */
    public FixedVersionStrategy(String version) {
        super(new PrefixVersionPathStrategy(version));
        this.version = version;
    }

    //返回前缀
    @Override
    public String getResourceVersion(Resource resource) {
        return this.version;
    }

}
{%endcodeblock%}
##### GzipResourceResolver
用来处理带有`.gz`的资源拓展,并且`Accept-Encoding`必须是`gzip`
##### PathResourceResolver
这是比较核心实现,用来根据url获取`location`中的资源
