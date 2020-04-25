---
title: springMVC配置
cover: /img/spring.png
top_img: /img/post.jpg
date: 2020-03-31 15:07:48
tags:
- 框架
- spring
- springMvc
categories: java
description: springMVC配置基础设施配置
---
### annotation-driven
#### xml
{%asset_img mvc_annotation_driven.png%}
##### 源码
  {%codeblock lang:java AnnotationDrivenBeanDefinitionParser%}
  class AnnotationDrivenBeanDefinitionParser implements BeanDefinitionParser {
  public BeanDefinition parse(Element element, ParserContext context) {
    //[1] 最外层ccd
        Object source = context.extractSource(element);
        XmlReaderContext readerContext = context.getReaderContext();

        CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
        context.pushContainingComponent(compDefinition);
    //[1!]
    //[2]处理RequestMappingHandlerMapping的创建
    //contentNegotiationManager的创建
        RuntimeBeanReference contentNegotiationManager = getContentNegotiationManager(element, source, context);

        RootBeanDefinition handlerMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
        handlerMappingDef.setSource(source);
        handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        handlerMappingDef.getPropertyValues().add("order", 0);
        handlerMappingDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
    //设置是否去除;,以适应矩阵变量
        if (element.hasAttribute("enable-matrix-variables")) {
            Boolean enableMatrixVariables = Boolean.valueOf(element.getAttribute("enable-matrix-variables"));
            handlerMappingDef.getPropertyValues().add("removeSemicolonContent", !enableMatrixVariables);
        }
    //设置mapping中patchMatch变量,根据子属性<mvc:path-matching>
        configurePathMatchingProperties(handlerMappingDef, element, context);
        readerContext.getRegistry().registerBeanDefinition(HANDLER_MAPPING_BEAN_NAME, handlerMappingDef);//注册mapping
    //创建一个CorsConfiguration,设置为mapping属性
        RuntimeBeanReference corsRef = MvcNamespaceUtils.registerCorsConfigurations(null, context, source);
        handlerMappingDef.getPropertyValues().add("corsConfigurations", corsRef);
    //conversion-service标签创建cs,默认情况则通过fb创建
        RuntimeBeanReference conversionService = getConversionService(element, source, context);
    //[2!]
    //[3] ConfigurableWebBindingInitializer创建
    //validator标签创建validator,默认会创建
        RuntimeBeanReference validator = getValidator(element, source, context);
        RuntimeBeanReference messageCodesResolver = getMessageCodesResolver(element);

        RootBeanDefinition bindingDef = new RootBeanDefinition(ConfigurableWebBindingInitializer.class);
        bindingDef.setSource(source);
        bindingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        bindingDef.getPropertyValues().add("conversionService", conversionService);
        bindingDef.getPropertyValues().add("validator", validator);
        bindingDef.getPropertyValues().add("messageCodesResolver", messageCodesResolver);
    //[3!]

    //[4] RequestMappingHandlerAdapter创建
    ManagedList<?> messageConverters = getMessageConverters(element, source, context);
        ManagedList<?> argumentResolvers = getArgumentResolvers(element, context);
        ManagedList<?> returnValueHandlers = getReturnValueHandlers(element, context);
        String asyncTimeout = getAsyncTimeout(element);
        RuntimeBeanReference asyncExecutor = getAsyncExecutor(element);
        ManagedList<?> callableInterceptors = getInterceptors(element, source, context, "callable-interceptors");
        ManagedList<?> deferredResultInterceptors = getInterceptors(element, source, context, "deferred-result-interceptors");

        RootBeanDefinition handlerAdapterDef = new RootBeanDefinition(RequestMappingHandlerAdapter.class);
        handlerAdapterDef.setSource(source);
        handlerAdapterDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        handlerAdapterDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
        handlerAdapterDef.getPropertyValues().add("webBindingInitializer", bindingDef);
        handlerAdapterDef.getPropertyValues().add("messageConverters", messageConverters);
        addRequestBodyAdvice(handlerAdapterDef);
        addResponseBodyAdvice(handlerAdapterDef);

        if (element.hasAttribute("ignore-default-model-on-redirect")) {
            Boolean ignoreDefaultModel = Boolean.valueOf(element.getAttribute("ignore-default-model-on-redirect"));
            handlerAdapterDef.getPropertyValues().add("ignoreDefaultModelOnRedirect", ignoreDefaultModel);
        }
        if (argumentResolvers != null) {
            handlerAdapterDef.getPropertyValues().add("customArgumentResolvers", argumentResolvers);
        }
        if (returnValueHandlers != null) {
            handlerAdapterDef.getPropertyValues().add("customReturnValueHandlers", returnValueHandlers);
        }
        if (asyncTimeout != null) {
            handlerAdapterDef.getPropertyValues().add("asyncRequestTimeout", asyncTimeout);
        }
        if (asyncExecutor != null) {
            handlerAdapterDef.getPropertyValues().add("taskExecutor", asyncExecutor);
        }

        handlerAdapterDef.getPropertyValues().add("callableInterceptors", callableInterceptors);
        handlerAdapterDef.getPropertyValues().add("deferredResultInterceptors", deferredResultInterceptors);
        readerContext.getRegistry().registerBeanDefinition(HANDLER_ADAPTER_BEAN_NAME, handlerAdapterDef);
    //[4!]
    //[5]
    RootBeanDefinition uriContributorDef =
                new RootBeanDefinition(CompositeUriComponentsContributorFactoryBean.class);
        uriContributorDef.setSource(source);
        uriContributorDef.getPropertyValues().addPropertyValue("handlerAdapter", handlerAdapterDef);
        uriContributorDef.getPropertyValues().addPropertyValue("conversionService", conversionService);
        String uriContributorName = MvcUriComponentsBuilder.MVC_URI_COMPONENTS_CONTRIBUTOR_BEAN_NAME;
        readerContext.getRegistry().registerBeanDefinition(uriContributorName, uriContributorDef);
    //[5!]
    //[6] 创建了几个拦击器
    RootBeanDefinition csInterceptorDef = new RootBeanDefinition(ConversionServiceExposingInterceptor.class);
        csInterceptorDef.setSource(source);
        csInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, conversionService);
        RootBeanDefinition mappedInterceptorDef = new RootBeanDefinition(MappedInterceptor.class);
        mappedInterceptorDef.setSource(source);
        mappedInterceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, (Object) null);
        mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(1, csInterceptorDef);
        String mappedInterceptorName = readerContext.registerWithGeneratedName(mappedInterceptorDef);

        RootBeanDefinition methodExceptionResolver = new RootBeanDefinition(ExceptionHandlerExceptionResolver.class);
        methodExceptionResolver.setSource(source);
        methodExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        methodExceptionResolver.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
        methodExceptionResolver.getPropertyValues().add("messageConverters", messageConverters);
        methodExceptionResolver.getPropertyValues().add("order", 0);
        addResponseBodyAdvice(methodExceptionResolver);
        if (argumentResolvers != null) {
            methodExceptionResolver.getPropertyValues().add("customArgumentResolvers", argumentResolvers);
        }
        if (returnValueHandlers != null) {
            methodExceptionResolver.getPropertyValues().add("customReturnValueHandlers", returnValueHandlers);
        }
        String methodExResolverName = readerContext.registerWithGeneratedName(methodExceptionResolver);

        RootBeanDefinition statusExceptionResolver = new RootBeanDefinition(ResponseStatusExceptionResolver.class);
        statusExceptionResolver.setSource(source);
        statusExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        statusExceptionResolver.getPropertyValues().add("order", 1);
        String statusExResolverName = readerContext.registerWithGeneratedName(statusExceptionResolver);

        RootBeanDefinition defaultExceptionResolver = new RootBeanDefinition(DefaultHandlerExceptionResolver.class);
        defaultExceptionResolver.setSource(source);
        defaultExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        defaultExceptionResolver.getPropertyValues().add("order", 2);
        String defaultExResolverName = readerContext.registerWithGeneratedName(defaultExceptionResolver);
    //[6]
}  
//--------------------------创建contextNegotiationManager-----------------------
private RuntimeBeanReference getContentNegotiationManager(
            Element element, @Nullable Object source, ParserContext context) {

        RuntimeBeanReference beanRef;
        if (element.hasAttribute("content-negotiation-manager")) { //标签创建,指向一个引用
            String name = element.getAttribute("content-negotiation-manager");
            beanRef = new RuntimeBeanReference(name);
        }
        else { //默认情况通过factoryBean创建,ref指向该fb
            RootBeanDefinition factoryBeanDef = new RootBeanDefinition(ContentNegotiationManagerFactoryBean.class);
            factoryBeanDef.setSource(source);
            factoryBeanDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
            factoryBeanDef.getPropertyValues().add("mediaTypes", getDefaultMediaTypes());
            String name = CONTENT_NEGOTIATION_MANAGER_BEAN_NAME;
            context.getReaderContext().getRegistry().registerBeanDefinition(name, factoryBeanDef);
            context.registerComponent(new BeanComponentDefinition(factoryBeanDef, name));
            beanRef = new RuntimeBeanReference(name);
        }
        return beanRef;
    }
//=====================MvcNamespaceUtils============================
//处理cosConfigurations配置
public static RuntimeBeanReference registerCorsConfigurations(
            @Nullable Map<String, CorsConfiguration> corsConfigurations,
            ParserContext context, @Nullable Object source) {

        if (!context.getRegistry().containsBeanDefinition(CORS_CONFIGURATION_BEAN_NAME)) { //若已经存在bd
      //将形参corsConfigurations设置为当前bd的构造器属性
            RootBeanDefinition corsDef = new RootBeanDefinition(LinkedHashMap.class);
            corsDef.setSource(source);
            corsDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
            if (corsConfigurations != null) {
                corsDef.getConstructorArgumentValues().addIndexedArgumentValue(0, corsConfigurations);
            }
            context.getReaderContext().getRegistry().registerBeanDefinition(CORS_CONFIGURATION_BEAN_NAME, corsDef);
            context.registerComponent(new BeanComponentDefinition(corsDef, CORS_CONFIGURATION_BEAN_NAME));
        }
        else if (corsConfigurations != null) { //若不存在,则创建一个cosConfigurations,并将形参作为构造器参数
            BeanDefinition corsDef = context.getRegistry().getBeanDefinition(CORS_CONFIGURATION_BEAN_NAME);
            corsDef.getConstructorArgumentValues().addIndexedArgumentValue(0, corsConfigurations);
        }
        return new RuntimeBeanReference(CORS_CONFIGURATION_BEAN_NAME);
    }  
  {%endcodeblock%}
##### 总结
###### HandlerMapping相关
- xml配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager"
                       enable-matrix-variables="true"></mvc:annotation-driven>
    <bean id="contentNegotiationManager" class="org.springframework.web.accept.ContentNegotiationManager"/>

</beans>
```
- 配置清单
  - content-negotiation-manager:设置`ContentNegotiationManager`,参考[消息处理器](/2020/03/23/springMVC核心二/#MessageConverter处理器)
  - enable-matrix-variables:`true`表示开始矩阵变量
  - corsConfigurations:这个配置实际上属于`mvc:cors`去处理
  - path-matcher:用来配置`mapping`中路径匹配器的,默认是实现是`AntPathMatch`
###### HandlerAdapter相关
- xml配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">
<mvc:annotation-driven ignore-default-model-on-redirect="true">
    <mvc:message-converters register-defaults="true">
        <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
        </bean>
    </mvc:message-converters>
    <mvc:argument-resolvers>
        <bean class="org.springframework.web.method.annotation.ModelAttributeMethodProcessor"></bean>
    </mvc:argument-resolvers>
    <mvc:return-value-handlers>
        <bean class="org.springframework.web.method.annotation.ModelAttributeMethodProcessor"/>
    </mvc:return-value-handlers>
    <mvc:async-support default-timeout="1000"/>
</mvc:annotation-driven>
</beans>
```
- 配置清单
  - ignore-default-model-on-redirect:表示重定向时直接使用`RediectModel`
  - message-converters:配置消息处理器,其中`register-defaults`默认就是true,解析器会根据`spi`来决定是否添加对应的转换器,如`Jackson`
  - argument-resolvers:配置自定义入参解析器
  - return-resolvers:自定义回参解析器
  - async-support:异步框架处理
###### BindingInitializer
- xml配置
{%codeblock lang:xml%}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">
<mvc:annotation-driven conversion-service="" message-codes-resolver="" validator="">
</beans>
{%endcodeblock%}
这个很简单,如上所示
### interceptors
用来创建拦截器
#### xml
##### 代码
{%codeblock lang:java %}
class InterceptorsBeanDefinitionParser implements BeanDefinitionParser {\
  public BeanDefinition parse(Element element, ParserContext context) {
        context.pushContainingComponent(
                new CompositeComponentDefinition(element.getTagName(), context.extractSource(element)));

        RuntimeBeanReference pathMatcherRef = null;
        if (element.hasAttribute("path-matcher")) {
            pathMatcherRef = new RuntimeBeanReference(element.getAttribute("path-matcher"));
        }
    //子类标签
        List<Element> interceptors = DomUtils.getChildElementsByTagName(element, "bean", "ref", "interceptor");
        for (Element interceptor : interceptors) {
      //最外层用MappedInterceptor包裹
            RootBeanDefinition mappedInterceptorDef = new RootBeanDefinition(MappedInterceptor.class);
            mappedInterceptorDef.setSource(context.extractSource(interceptor));
            mappedInterceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

            ManagedList<String> includePatterns = null;
            ManagedList<String> excludePatterns = null;
            Object interceptorBean;
            if ("interceptor".equals(interceptor.getLocalName())) {
                includePatterns = getIncludePatterns(interceptor, "mapping");
                excludePatterns = getIncludePatterns(interceptor, "exclude-mapping");
                Element beanElem = DomUtils.getChildElementsByTagName(interceptor, "bean", "ref").get(0);
                interceptorBean = context.getDelegate().parsePropertySubElement(beanElem, null);
            }
            else {
                interceptorBean = context.getDelegate().parsePropertySubElement(interceptor, null);
            }
            mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, includePatterns);
            mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(1, excludePatterns);
            mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(2, interceptorBean);

            if (pathMatcherRef != null) {
                mappedInterceptorDef.getPropertyValues().add("pathMatcher", pathMatcherRef);
            }

            String beanName = context.getReaderContext().registerWithGeneratedName(mappedInterceptorDef);
            context.registerComponent(new BeanComponentDefinition(mappedInterceptorDef, beanName));
        }

        context.popAndRegisterContainingComponent();
        return null;
    }

    private ManagedList<String> getIncludePatterns(Element interceptor, String elementName) {
        List<Element> paths = DomUtils.getChildElementsByTagName(interceptor, elementName);
        ManagedList<String> patterns = new ManagedList<>(paths.size());
        for (Element path : paths) {
            patterns.add(path.getAttribute("path"));
        }
        return patterns;
    }
}  
{%endcodeblock%}
##### 总结
简单来说,就是外部包裹`MappedInterceptor`,这种拦截器在`AbstractHandlerMapping`创建`Chain`的时候用来选择
```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
        HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
                (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

        String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
        for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
            if (interceptor instanceof MappedInterceptor) { //MappedInterceptor的处理方式
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
```
- 例子
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven />

    <mvc:interceptors>
        <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor"/>
        <mvc:interceptor>
            <mvc:mapping path="/**" />
            <mvc:exclude-mapping path="/admin/**" />
            <mvc:exclude-mapping path="/images/**" />
            <bean class="org.springframework.web.servlet.theme.ThemeChangeInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>

    <mvc:interceptors path-matcher="pathMatcher">
        <mvc:interceptor>
            <mvc:mapping path="/accounts/[0-9]*" />
            <bean class="org.springframework.web.servlet.handler.UserRoleAuthorizationInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>

    <bean id="pathMatcher" class="org.springframework.web.servlet.config.MvcNamespaceTests$TestPathMatcher"/>

</beans>
```
