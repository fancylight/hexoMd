---
title: springContext配置
cover: /img/post.jpg
top_img: /img/post.jpg
date: 2020-04-08 09:08:05
tags:
- 框架
- spring
- springMvc
categories: java
description: springContext配置基础设施配置
---
# compoent-scan
## 代码逻辑

{%codeblock lang:java ComponentScanBeanDefinitionParser%}
        public BeanDefinition parse(Element element, ParserContext parserContext) {
      String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
      basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
      String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

      // Actually scan for bean definitions and register them.
      //调用该Scanner来处理
      ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
      //返回在对应的classPath 正确注解的bean
      Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
      //给当前BeanFactory中添加一些后处理器,此处的功能和annotation-config类似
      registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
      return null;
    }
{%endcodeblock%}
## 总结
- ClassPathBeanDefinitionScanner:这个才是实际扫描路径并创建的sbd的类
spring处理注解@ComponentScan注解最终也要用到该类,外层是ComponentScanAnnotationParser
具体注解处理的代码分析可以参考,[springBoot学习一](/2019/09/05/springBoot学习一/#applicationcontextinitializer).
## annotation-config
- 代码逻辑
{%codeblock lang:java AnnotationConfigBeanDefinitionParser%}    
        public class AnnotationConfigBeanDefinitionParser implements BeanDefinitionParser {

    @Override
    @Nullable
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        Object source = parserContext.extractSource(element);

        // Obtain bean definitions for all relevant BeanPostProcessors.
    //AnnotationConfigUtils该类在许多springContext都用到了
        Set<BeanDefinitionHolder> processorDefinitions =
                AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);

        // Register component for the surrounding <context:annotation-config> element.
        CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
        parserContext.pushContainingComponent(compDefinition);

        // Nest the concrete beans in the surrounding component.
        for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
            parserContext.registerComponent(new BeanComponentDefinition(processorDefinition));
        }

        // Finally register the composite component.
        parserContext.popAndRegisterContainingComponent();

        return null;
    }

}
{%endcodeblock%}
对于AnnotationConfigUtils参考[context相关](/2019/09/05/springBoot学习一/#boot中context)
# load-time-weaver
- 代码逻辑
{%codeblock lang:java LoadTimeWeaverBeanDefinitionParser%}
public static final String ASPECTJ_WEAVING_ENABLER_BEAN_NAME =
        "org.springframework.context.config.internalAspectJWeavingEnabler";

private static final String ASPECTJ_WEAVING_ENABLER_CLASS_NAME =
        "org.springframework.context.weaving.AspectJWeavingEnabler";

private static final String DEFAULT_LOAD_TIME_WEAVER_CLASS_NAME =
        "org.springframework.context.weaving.DefaultContextLoadTimeWeaver";

private static final String WEAVER_CLASS_ATTRIBUTE = "weaver-class";

private static final String ASPECTJ_WEAVING_ATTRIBUTE = "aspectj-weaving";


//返回DefaultContextLoadTimeWeaver beanClass,具体逻辑要看AbstractSingleBeanDefinitionParser逻辑
@Override
protected String getBeanClassName(Element element) {
    if (element.hasAttribute(WEAVER_CLASS_ATTRIBUTE)) {
        return element.getAttribute(WEAVER_CLASS_ATTRIBUTE);
    }
    return DEFAULT_LOAD_TIME_WEAVER_CLASS_NAME;
}


//返回LTW bean Name
@Override
protected String resolveId(Element element, AbstractBeanDefinition definition, ParserContext parserContext) {
    return ConfigurableApplicationContext.LOAD_TIME_WEAVER_BEAN_NAME;
}

//解析标签,创建一个AspectJWeavingEnabler bean
@Override
protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
    builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

    if (isAspectJWeavingEnabled(element.getAttribute(ASPECTJ_WEAVING_ATTRIBUTE), parserContext)) {
        if (!parserContext.getRegistry().containsBeanDefinition(ASPECTJ_WEAVING_ENABLER_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(ASPECTJ_WEAVING_ENABLER_CLASS_NAME);
            parserContext.registerBeanComponent(
                    new BeanComponentDefinition(def, ASPECTJ_WEAVING_ENABLER_BEAN_NAME));
        }

        if (isBeanConfigurerAspectEnabled(parserContext.getReaderContext().getBeanClassLoader())) {
            new SpringConfiguredBeanDefinitionParser().parse(element, parserContext);
        }
    }
}

{%endcodeblock%}
## 总结
创建`DefaultContextLoadTimeWeaver`GBD和`AspectJWeavingEnabler`
简单来说LWT就是载入时织入,与spring aop实现点不同,后者是运行时编译
# property-
`property-placeholder`和`property-override`,前者用来提供处理bd中`${}`的功能,后者
- xml解析器
{%asset_img PropretyBDP.png%}
这是一个`AbstractSingleBeanDefinitionParser`子类,参考[NamespaceHandler](/2020/02/11/spring常见/#NamespaceHandler体系)
## property-placeholder`
### 代码逻辑
{%codeblock lang:java AbstractPropertyLoadingBeanDefinitionParser%}
abstract class AbstractPropertyLoadingBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {

    @Override
    protected boolean shouldGenerateId() {
        return true;
    }
    //处理共有属性
    @Override
    protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
        String location = element.getAttribute("location");
        if (StringUtils.hasLength(location)) {
            location = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(location);
            String[] locations = StringUtils.commaDelimitedListToStringArray(location);
            builder.addPropertyValue("locations", locations);
        }

        String propertiesRef = element.getAttribute("properties-ref");
        if (StringUtils.hasLength(propertiesRef)) {
            builder.addPropertyReference("properties", propertiesRef);
        }

        String fileEncoding = element.getAttribute("file-encoding");
        if (StringUtils.hasLength(fileEncoding)) {
            builder.addPropertyValue("fileEncoding", fileEncoding);
        }

        String order = element.getAttribute("order");
        if (StringUtils.hasLength(order)) {
            builder.addPropertyValue("order", Integer.valueOf(order));
        }

        builder.addPropertyValue("ignoreResourceNotFound",
                Boolean.valueOf(element.getAttribute("ignore-resource-not-found")));

        builder.addPropertyValue("localOverride",
                Boolean.valueOf(element.getAttribute("local-override")));

        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    }
}
//具体的属性
class PropertyPlaceholderBeanDefinitionParser extends AbstractPropertyLoadingBeanDefinitionParser {

    private static final String SYSTEM_PROPERTIES_MODE_ATTRIBUTE = "system-properties-mode";

    private static final String SYSTEM_PROPERTIES_MODE_DEFAULT = "ENVIRONMENT";


    @Override
    @SuppressWarnings("deprecation")
    protected Class<?> getBeanClass(Element element) {
        // As of Spring 3.1, the default value of system-properties-mode has changed from
        // 'FALLBACK' to 'ENVIRONMENT'. This latter value indicates that resolution of
        // placeholders against system properties is a function of the Environment and
        // its current set of PropertySources.
        if (SYSTEM_PROPERTIES_MODE_DEFAULT.equals(element.getAttribute(SYSTEM_PROPERTIES_MODE_ATTRIBUTE))) {//这个属性实际上是dtd决定的
            return PropertySourcesPlaceholderConfigurer.class; //3.0之后
        }

        // The user has explicitly specified a value for system-properties-mode: revert to
        // PropertyPlaceholderConfigurer to ensure backward compatibility with 3.0 and earlier.
        // This is deprecated; to be removed along with PropertyPlaceholderConfigurer itself.
        return org.springframework.beans.factory.config.PropertyPlaceholderConfigurer.class; //3.0之前
    }

    @Override
    protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
        super.doParse(element, parserContext, builder);

        builder.addPropertyValue("ignoreUnresolvablePlaceholders",
                Boolean.valueOf(element.getAttribute("ignore-unresolvable")));

        String systemPropertiesModeName = element.getAttribute(SYSTEM_PROPERTIES_MODE_ATTRIBUTE);
        if (StringUtils.hasLength(systemPropertiesModeName) &&
                !systemPropertiesModeName.equals(SYSTEM_PROPERTIES_MODE_DEFAULT)) {
            builder.addPropertyValue("systemPropertiesModeName", "SYSTEM_PROPERTIES_MODE_" + systemPropertiesModeName);
        }

        if (element.hasAttribute("value-separator")) {
            builder.addPropertyValue("valueSeparator", element.getAttribute("value-separator"));
        }
        if (element.hasAttribute("trim-values")) {
            builder.addPropertyValue("trimValues", element.getAttribute("trim-values"));
        }
        if (element.hasAttribute("null-value")) {
            builder.addPropertyValue("nullValue", element.getAttribute("null-value"));
        }
    }

}
{%endcodeblock%}
### 实现原理
`PropertySourcesPlaceholderConfigurer`是提供将bd的${}转化为对应属性的`BFP`
{%asset_img PropertySourcesPlaceholderConfigurer.png%}
- 代码逻辑
{%codeblock lang:java PropertySourcesPlaceholderConfigurer%}
public class PropertySourcesPlaceholderConfigurer extends PlaceholderConfigurerSupport implements EnvironmentAware {
@Nullable
private MutablePropertySources propertySources;//包含解析的properties文件属性

@Nullable
private PropertySources appliedPropertySources;

@Nullable
private Environment environment; //通过aware接口获取的
//处理器函数
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    //[1] 若ps中为空,则从env中获取
    if (this.propertySources == null) {
        this.propertySources = new MutablePropertySources();
        if (this.environment != null) {
            this.propertySources.addLast(
                new PropertySource<Environment>(ENVIRONMENT_PROPERTIES_PROPERTY_SOURCE_NAME, this.environment) {
                    @Override
                    @Nullable
                    public String getProperty(String key) {
                        return this.source.getProperty(key);
                    }
                }
            );
        }
        try {
            PropertySource<?> localPropertySource =
                    new PropertiesPropertySource(LOCAL_PROPERTIES_PROPERTY_SOURCE_NAME, mergeProperties()); //合并localtion中,即用户定义的properties顺序ing
            if (this.localOverride) {
                this.propertySources.addFirst(localPropertySource);
            }
            else {
                this.propertySources.addLast(localPropertySource);
            }
        }
        catch (IOException ex) {
            throw new BeanInitializationException("Could not load properties", ex);
        }
    }
    //[1!]
    //使用PropertyResolver来完成对于bd中${}的处理
    processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));//创建了一个PropertySourcesPropertyResolver
    this.appliedPropertySources = this.propertySources;
}
//设置resolver属性
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
        final ConfigurablePropertyResolver propertyResolver) throws BeansException {

    propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);//${
    propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);//$}
    propertyResolver.setValueSeparator(this.valueSeparator);//:
    //这个valueResolver和形参propertyResolver是关键
    StringValueResolver valueResolver = strVal -> {
        String resolved = (this.ignoreUnresolvablePlaceholders ? //若未定义该属性,则调用抛出异常的
                propertyResolver.resolvePlaceholders(strVal) :
                propertyResolver.resolveRequiredPlaceholders(strVal));
        if (this.trimValues) {
            resolved = resolved.trim();
        }
        return (resolved.equals(this.nullValue) ? null : resolved);
    };

    doProcessProperties(beanFactoryToProcess, valueResolver);//父类调用
}
//----------------------PlaceholderConfigurerSupport父类调用---------------------
protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
            StringValueResolver valueResolver) {

        BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver); //创建vistor

        String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
        for (String curName : beanNames) { //避免处理本身这个处理器
            // Check that we're not parsing our own bean definition,
            // to avoid failing on unresolvable placeholders in properties file locations.
            if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
                BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
                try {
                    visitor.visitBeanDefinition(bd);//调用vistor
                }
                catch (Exception ex) {
                    throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
                }
            }
        }

        // New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
        //处理别名
        beanFactoryToProcess.resolveAliases(valueResolver);

        // New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
        //加入到bf中,实际AutoWire..Post会调用到,来处理@Value注解
        beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
    }
}
{%endcodeblock%}
#### BeanDefinitionVisitor
该类的作用是通过内部的`StringValueResolver`来对bd的部分属性做一次处理,实际上就是说在bd中
不仅仅是pv可以使用`${}`,在这里出现的都一定程度上可以使用
- 代码逻辑
{%codeblock lang:java BeanDefinitionVisitor%}
public class BeanDefinitionVisitor {
//Value解析器
private StringValueResolver valueResolver;

public void visitBeanDefinition(BeanDefinition beanDefinition) {
    //[0]resolverStringValue类型
    visitParentName(beanDefinition);
    visitBeanClassName(beanDefinition);
    visitFactoryBeanName(beanDefinition);
    visitFactoryMethodName(beanDefinition);
    visitScope(beanDefinition);
    //[0!]
    //[1]resolveValue类型
    if (beanDefinition.hasPropertyValues()) {
        visitPropertyValues(beanDefinition.getPropertyValues());
    }
    if (beanDefinition.hasConstructorArgumentValues()) {
        ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
        visitIndexedArgumentValues(cas.getIndexedArgumentValues());
        visitGenericArgumentValues(cas.getGenericArgumentValues());
    }
    //[1!]
}
//-----------------------resolverStringValue类型--------------------------
protected String resolveStringValue(String strVal) {
        if (this.valueResolver == null) {
            throw new IllegalStateException("No StringValueResolver specified - pass a resolver " +
                    "object into the constructor or override the 'resolveStringValue' method");
        }
        String resolvedValue = this.valueResolver.resolveStringValue(strVal);
        // Return original String if not modified.
        return (strVal.equals(resolvedValue) ? strVal : resolvedValue);
    }

    protected void visitParentName(BeanDefinition beanDefinition) {
            String parentName = beanDefinition.getParentName();
            if (parentName != null) {
                String resolvedName = resolveStringValue(parentName);
                if (!parentName.equals(resolvedName)) {
                    beanDefinition.setParentName(resolvedName);
                }
            }
        }
//.........

//-----------------------resolveValue类型--------------
protected Object resolveValue(@Nullable Object value) {
        if (value instanceof BeanDefinition) {
            visitBeanDefinition((BeanDefinition) value);
        }
        else if (value instanceof BeanDefinitionHolder) {
            visitBeanDefinition(((BeanDefinitionHolder) value).getBeanDefinition());
        }
        else if (value instanceof RuntimeBeanReference) {
            RuntimeBeanReference ref = (RuntimeBeanReference) value;
            String newBeanName = resolveStringValue(ref.getBeanName());
            if (newBeanName == null) {
                return null;
            }
            if (!newBeanName.equals(ref.getBeanName())) {
                return new RuntimeBeanReference(newBeanName);
            }
        }
        else if (value instanceof RuntimeBeanNameReference) {
            RuntimeBeanNameReference ref = (RuntimeBeanNameReference) value;
            String newBeanName = resolveStringValue(ref.getBeanName());
            if (newBeanName == null) {
                return null;
            }
            if (!newBeanName.equals(ref.getBeanName())) {
                return new RuntimeBeanNameReference(newBeanName);
            }
        }
        else if (value instanceof Object[]) {
            visitArray((Object[]) value);
        }
        else if (value instanceof List) {
            visitList((List) value);
        }
        else if (value instanceof Set) {
            visitSet((Set) value);
        }
        else if (value instanceof Map) {
            visitMap((Map) value);
        }
        else if (value instanceof TypedStringValue) {
            TypedStringValue typedStringValue = (TypedStringValue) value;
            String stringValue = typedStringValue.getValue();
            if (stringValue != null) {
                String visitedString = resolveStringValue(stringValue);
                typedStringValue.setValue(visitedString);
            }
        }
        else if (value instanceof String) {
            return resolveStringValue((String) value);
        }
        return value;
    }
}
{%endcodeblock%}
#### PropertySourcesPropertyResolver
该类真正的完成了对`${}`表达式的处理
{%asset_img PropertySourcesPropertyResolver.png%}
##### PropertyResolver
将内部的`PropertySource`转换为对应的类型
```java
//包装实现,调用内部源
boolean containsProperty(String key);
String getProperty(String key);
String getProperty(String key, String defaultValue);
//解析
<T> T getProperty(String key, Class<T> targetType);
<T> T getProperty(String key, Class<T> targetType, T defaultValue);
String getRequiredProperty(String key) throws IllegalStateException;
<T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException;
//处理${}表达式
//前者抛出异常,后者不抛异常
String resolvePlaceholders(String text);
String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;
```
##### ConfigurablePropertyResolver
可以配置前后缀,分隔符,以及转换器
{%codeblock lang:java ConfigurablePropertyResolver %}
public interface ConfigurablePropertyResolver extends PropertyResolver {
//转换器
ConfigurableConversionService getConversionService();
void setConversionService(ConfigurableConversionService conversionService);
//前后缀
void setPlaceholderPrefix(String placeholderPrefix);
void setPlaceholderSuffix(String placeholderSuffix);
void setValueSeparator(@Nullable String valueSeparator);
}
{%endcodeblock%}
##### AbstractPropertyResolver
定义了内部一些属性,实现通过函数
- 代码逻辑
{%codeblock lang:java AbstractPropertyResolver%}
protected final Log logger = LogFactory.getLog(getClass());

@Nullable
private volatile ConfigurableConversionService conversionService;

//用来处理${name}占位符
@Nullable
private PropertyPlaceholderHelper nonStrictHelper;

@Nullable
private PropertyPlaceholderHelper strictHelper;

private boolean ignoreUnresolvableNestedPlaceholders = false;

private String placeholderPrefix = SystemPropertyUtils.PLACEHOLDER_PREFIX;

private String placeholderSuffix = SystemPropertyUtils.PLACEHOLDER_SUFFIX;

@Nullable
private String valueSeparator = SystemPropertyUtils.VALUE_SEPARATOR;

private final Set<String> requiredProperties = new LinkedHashSet<>();
//处理占位符函数
public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
        if (this.strictHelper == null) {
            this.strictHelper = createPlaceholderHelper(false);
        }
        return doResolvePlaceholders(text, this.strictHelper);
    }
    public String resolvePlaceholders(String text) {
        if (this.nonStrictHelper == null) {
            this.nonStrictHelper = createPlaceholderHelper(true);
        }
        return doResolvePlaceholders(text, this.nonStrictHelper);
    }
    private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
            return helper.replacePlaceholders(text, this::getPropertyAsRawString); //注意这里传递一个了子类实现的函数引用
        }
        protected <T> T convertValueIfNecessary(Object value, @Nullable Class<T> targetType) {
            if (targetType == null) {
                return (T) value;
            }
            ConversionService conversionServiceToUse = this.conversionService;
            if (conversionServiceToUse == null) {
                // Avoid initialization of shared DefaultConversionService if
                // no standard type conversion is needed in the first place...
                if (ClassUtils.isAssignableValue(targetType, value)) {
                    return (T) value;
                }
                conversionServiceToUse = DefaultConversionService.getSharedInstance();
            }
            return conversionServiceToUse.convert(value, targetType); //cs进行转换
        }
{%endcodeblock%}
##### PropertyPlaceholderHelper
处理`${}`逻辑,实际前后缀分割符都可以定义
- 嵌套:`${${}}`
- 多值:`${xxx:xxx}`
{%codeblock lang:java %}

public class PropertyPlaceholderHelper {
    protected String parseStringValue(
                String value, PlaceholderResolver placeholderResolver, @Nullable Set<String> visitedPlaceholders) { //第二个参数实际被充当与函数接口使用
            //[1]判断前缀
            int startIndex = value.indexOf(this.placeholderPrefix);
            if (startIndex == -1) {
                return value;
            }
            //[1!]
            StringBuilder result = new StringBuilder(value);
            while (startIndex != -1) {
                int endIndex = findPlaceholderEndIndex(result, startIndex); //后缀
                if (endIndex != -1) {
                    //[1] placeholderResolver若没有提供,则说明没有能力解析${}中内容
                    String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
                    String originalPlaceholder = placeholder;
                    if (visitedPlaceholders == null) {
                        visitedPlaceholders = new HashSet<>(4);
                    }
                    if (!visitedPlaceholders.add(originalPlaceholder)) {
                        throw new IllegalArgumentException(
                                "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
                    }
                    //[1]
                    // Recursive invocation, parsing placeholders contained in the placeholder key.
                    //[2] 递归处理${${}}
                    placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
                    // Now obtain the value for the fully resolved key...
                    //[3] 处理含有分割符的
                    String propVal = placeholderResolver.resolvePlaceholder(placeholder);
                    if (propVal == null && this.valueSeparator != null) {
                        int separatorIndex = placeholder.indexOf(this.valueSeparator);
                        if (separatorIndex != -1) {
                            String actualPlaceholder = placeholder.substring(0, separatorIndex);
                            String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
                            propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
                            if (propVal == null) {
                                propVal = defaultValue;
                            }
                        }
                    }
                    if (propVal != null) {
                        // Recursive invocation, parsing placeholders contained in the
                        // previously resolved placeholder value.
                        propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
                        result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
                        if (logger.isTraceEnabled()) {
                            logger.trace("Resolved placeholder '" + placeholder + "'");
                        }
                        startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
                    }
                    //[3]
                    else if (this.ignoreUnresolvablePlaceholders) { //如果没有成功转换,则判断是否忽略
                        // Proceed with unprocessed value.
                        startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
                    }
                    else {
                        throw new IllegalArgumentException("Could not resolve placeholder '" +
                                placeholder + "'" + " in value \"" + value + "\"");
                    }
                    visitedPlaceholders.remove(originalPlaceholder);
                }
                else {
                    startIndex = -1;
                }
            }
            return result.toString();
        }
}
{%endcodeblock%}
##### PropertySourcesPropertyResolver
{%codeblock lang:java PropertySourcesPropertyResolver%}
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {
private final PropertySources propertySources; //数据源
//抽象调用的,最终会被PropertyPlaceholderHelper调用
protected String getPropertyAsRawString(String key) {
        return getProperty(key, String.class, false);
    }
public String getProperty(String key) {
        return getProperty(key, String.class, true);
    }

    @Override
    @Nullable
    public <T> T getProperty(String key, Class<T> targetValueType) {
        return getProperty(key, targetValueType, true);
    }

    @Override
    @Nullable
    protected String getPropertyAsRawString(String key) {
        return getProperty(key, String.class, false);
    }
//该类实现的转换逻辑
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) { //key就是${}内部的值
        if (this.propertySources != null) {
            for (PropertySource<?> propertySource : this.propertySources) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Searching for key '" + key + "' in PropertySource '" +
                            propertySource.getName() + "'");
                }
                Object value = propertySource.getProperty(key);//从property中获取value
                if (value != null) {
                    if (resolveNestedPlaceholders && value instanceof String) {
                        value = resolveNestedPlaceholders((String) value); //由于helper的递归逻辑,如果是help调用到这里,这个value一定不是${}这样的格式
                    }
                    logKeyFound(key, propertySource, value);
                    return convertValueIfNecessary(value, targetValueType); //根据占位符处理结果,进行类型转换
                }
            }
        }
        if (logger.isTraceEnabled()) {
            logger.trace("Could not find key '" + key + "' in any property source");
        }
        return null;
    }
}
{%endcodeblock%}
##### 总结
{%asset_img 占位符逻辑.png 占位符解析逻辑%}
- 实际使用了`PropertySourcesPropertyResolver`完成了占位符逻辑处理,从property获取vlaue,然后进行String->String的转换(实际上会直接返回)
- `PropertySourcesPropertyResolver`实际可以做到更多类型转换,这里仅仅使用了功能的子集
- 关于`${}`占位符的解析逻辑就是如此实现的
### @PropertySource
该注解常见于配置类型的项目,具体用来见该类注解,简单来说该类引入properties,并在'@Configration'类中使用
### Environment
该类本身比较简单,其构建者一般都是能够触及context的对象,如`SpringApplication`,以及`ConfigurationClassPostProcessor`处理器
为了处理`@Configration`也会创建
{%asset_img StandardEnvironment.png%}
上文中提到的`PropertySourcesPropertyResolver#getProperty`实际上会在此中使用,作为包装类内部属性
## property-override
使用propertis中的beanName.value=newValue,来替换bf中的对应beanName.value的pv属性,当然中间有string到对应类型的转换
### 例子
- 配置文件
```properties
test.password=123
```
- xml文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context" xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <context:property-override location="context/over.properties"></context:property-override>
    <bean id="test" class="context.property.PropertyBean"></bean>
</beans>
```
- 输出结果
`PropertyBean{name='null', password='123'}`
### 代码逻辑
{%codeblock lang:java PropertyOverrideBeanDefinitionParser%}
class PropertyOverrideBeanDefinitionParser extends AbstractPropertyLoadingBeanDefinitionParser {

    @Override
    protected Class<?> getBeanClass(Element element) {
        return PropertyOverrideConfigurer.class;
    }

    @Override
    protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
        super.doParse(element, parserContext, builder);

        builder.addPropertyValue("ignoreInvalidKeys",
                Boolean.valueOf(element.getAttribute("ignore-unresolvable")));

    }

}
{%endcodeblock%}
### 实现原理
{%asset_img PropertyOverrideConfigurer.png%}
- 代码逻辑
{%codeblock lang:java PropertyOverrideConfigurer.png%}
public class PropertyOverrideConfigurer extends PropertyResourceConfigurer {
    //父类postProcessBeanFactory()调用
    protected void processProperties(ConfigurableListableBeanFactory beanFactory, Properties props)
            throws BeansException {

        for (Enumeration<?> names = props.propertyNames(); names.hasMoreElements();) {//遍历props文件
            String key = (String) names.nextElement();
            try {
                processKey(beanFactory, key, props.getProperty(key));
            }
            catch (BeansException ex) {
                String msg = "Could not process key '" + key + "' in PropertyOverrideConfigurer";
                if (!this.ignoreInvalidKeys) {
                    throw new BeanInitializationException(msg, ex);
                }
                if (logger.isDebugEnabled()) {
                    logger.debug(msg, ex);
                }
            }
        }
    }

    protected void processKey(ConfigurableListableBeanFactory factory, String key, String value)
            throws BeansException {

        int separatorIndex = key.indexOf(this.beanNameSeparator);
        if (separatorIndex == -1) {
            throw new BeanInitializationException("Invalid key '" + key +
                    "': expected 'beanName" + this.beanNameSeparator + "property'");
        }
        String beanName = key.substring(0, separatorIndex);//propertis文件中beanname
        String beanProperty = key.substring(separatorIndex + 1);//beanname.key
        this.beanNames.add(beanName);
        applyPropertyValue(factory, beanName, beanProperty, value); //beanmane.key=vlaue
        if (logger.isDebugEnabled()) {
            logger.debug("Property '" + key + "' set to value [" + value + "]");
        }
    }

    protected void applyPropertyValue(
                ConfigurableListableBeanFactory factory, String beanName, String property, String value) {

            BeanDefinition bd = factory.getBeanDefinition(beanName);
            BeanDefinition bdToUse = bd;
            while (bd != null) {
                bdToUse = bd;
                bd = bd.getOriginatingBeanDefinition();
            }
            PropertyValue pv = new PropertyValue(property, value); //创建新的属性
            pv.setOptional(this.ignoreInvalidKeys);
            bdToUse.getPropertyValues().addPropertyValue(pv);
        }
}
{%endcodeblock%}
