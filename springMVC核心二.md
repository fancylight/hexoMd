---
title: springMVC核心二
cover: /img/spring.png
top_img: /img/post.jpg
date: 2020-03-23 09:59:32
tags:
- 框架
- spring
- springMvc
categories: java
description: 描述mvc入参|回参处理
---
### 入参解析器
#### NameValue参数
改解析器用来处理NameValue注解,即含有`name`,`flag`,`defaultValue`
- 代码逻辑
{%codeblock lang:java AbstractNamedValueMethodArgumentResolver%}
  //el表达基础条件
  private final ConfigurableBeanFactory configurableBeanFactory;
  //处理el表达式
    @Nullable
    private final BeanExpressionContext expressionContext;

    private final Map<MethodParameter, NamedValueInfo> namedValueInfoCache = new ConcurrentHashMap<>(256);

  //抽象处理逻辑
  public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

        NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
        MethodParameter nestedParameter = parameter.nestedIfOptional();

    //调用el处理name属性,即说明name||defaultValue都支持el表达式
        Object resolvedName = resolveStringValue(namedValueInfo.name);
        if (resolvedName == null) {
            throw new IllegalArgumentException(
                    "Specified name must not resolve to null: [" + namedValueInfo.name + "]");
        }
    //子类处理
        Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
        if (arg == null) {
            if (namedValueInfo.defaultValue != null) {
                arg = resolveStringValue(namedValueInfo.defaultValue); //设置为默认值
            }
            else if (namedValueInfo.required && !nestedParameter.isOptional()) {
                handleMissingValue(namedValueInfo.name, nestedParameter, webRequest); //缺失处理,抛出异常
            }
            arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType()); //空值处理
        }
        else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
            arg = resolveStringValue(namedValueInfo.defaultValue);
        }

        if (binderFactory != null) { //dataBinder调用点,这里用来类型转换
            WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
            try {
                arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
            }
            catch (ConversionNotSupportedException ex) {
                throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
                        namedValueInfo.name, parameter, ex.getCause());
            }
            catch (TypeMismatchException ex) {
                throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
                        namedValueInfo.name, parameter, ex.getCause());
            }
        }
    //提供一个处理结束调用点,子类实现,一般不处理
        handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

        return arg;
    }
  //-----------------------------处理NameValueInfo---------------------
  //生成NameValueInfo
  private NamedValueInfo getNamedValueInfo(MethodParameter parameter) {
          NamedValueInfo namedValueInfo = this.namedValueInfoCache.get(parameter);
          if (namedValueInfo == null) {
              namedValueInfo = createNamedValueInfo(parameter);//子类实现
              namedValueInfo = updateNamedValueInfo(parameter, namedValueInfo);
              this.namedValueInfoCache.put(parameter, namedValueInfo);
          }
          return namedValueInfo;
      }
  //保证生成的NameValueInfo属性正确  
  private NamedValueInfo updateNamedValueInfo(MethodParameter parameter, NamedValueInfo info) {
          String name = info.name;
          if (info.name.isEmpty()) {
              name = parameter.getParameterName(); //使用形参类型名
              if (name == null) {
                  throw new IllegalArgumentException(
                          "Name for argument type [" + parameter.getNestedParameterType().getName() +
                          "] not available, and parameter name information not found in class file either.");
              }
          }
          String defaultValue = (ValueConstants.DEFAULT_NONE.equals(info.defaultValue) ? null : info.defaultValue);
          return new NamedValueInfo(name, info.required, defaultValue);
      }  
  //内部类,用来描述对应注解信息
  protected static class NamedValueInfo {

        private final String name;

        private final boolean required;

        @Nullable
        private final String defaultValue;

        public NamedValueInfo(String name, boolean required, @Nullable String defaultValue) {
            this.name = name;
            this.required = required;
            this.defaultValue = defaultValue;
        }
    }  
  //-----------------------------el表达式处理-----------------------
  private Object resolveStringValue(String value) {
        if (this.configurableBeanFactory == null) {
            return value;
        }
        String placeholdersResolved = this.configurableBeanFactory.resolveEmbeddedValue(value);
        BeanExpressionResolver exprResolver = this.configurableBeanFactory.getBeanExpressionResolver();
        if (exprResolver == null || this.expressionContext == null) {
            return value;
        }
        return exprResolver.evaluate(placeholdersResolved, this.expressionContext);
    }
  //--------------------------
{%endcodeblock%}
- 简单总结
  - `name`,`defaultValue`支持el表达式,并不是所有子类都支持,子类构造提供了`bf`才能支持
  - 子类处理miss|null情况
  - 有处理结束调用点
##### RequestHeader
- 代码逻辑
{%codeblock lang:java RequestHeaderMethodArgumentResolver%}
  public RequestHeaderMethodArgumentResolver(@Nullable ConfigurableBeanFactory beanFactory) {
        super(beanFactory); //支持el表达式
    }
  //形参条件
  public boolean supportsParameter(MethodParameter parameter) {
        return (parameter.hasParameterAnnotation(RequestHeader.class) &&
                !Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType()));
    }
  //解析
  protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
        String[] headerValues = request.getHeaderValues(name);
        if (headerValues != null) { //获取第一个
            return (headerValues.length == 1 ? headerValues[0] : headerValues);
        }
        else {
            return null;
        }
    }


  //--------------内部类----------------
  private static final class RequestHeaderNamedValueInfo extends NamedValueInfo {

        private RequestHeaderNamedValueInfo(RequestHeader annotation) { //标准实现
            super(annotation.name(), annotation.required(), annotation.defaultValue());
        }
    }
{%endcodeblock%}
- 总结
  - 形参只能为非`Map`和`Optional<Map>`
  - 支持el表达式
  - 获取请求头属性
##### RequestAttribute
`RequestAttributeMethodArgumentResolver`,代码简单略
- 总结
  - 形参支持`@RequestAttribute`修饰
  - 不支持el
  - 获取spring封装的`WebRequest`属性
##### RequestPara
`RequestParamMethodArgumentResolver`,略
- 代码逻辑
{%codeblock lang:java %}  
  public RequestParamMethodArgumentResolver(@Nullable ConfigurableBeanFactory beanFactory,
            boolean useDefaultResolution) {

        super(beanFactory);
        this.useDefaultResolution = useDefaultResolution; //是否支持simple类型
    }


  public  boolean supportsParameter(MethodParameter parameter) {
        if (parameter.hasParameterAnnotation(RequestParam.class)) {
            if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
                RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
                return (requestParam != null && StringUtils.hasText(requestParam.name()));
            }
            else {
                return true;
            }
        }
        else {
            if (parameter.hasParameterAnnotation(RequestPart.class)) {
                return false;
            }
            parameter = parameter.nestedIfOptional();
            if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
                return true;
            }
            else if (this.useDefaultResolution) {
                return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
            }
            else {
                return false;
            }
        }
    }
  //处理
  protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
        HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

        if (servletRequest != null) { //Multipart处理
            Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
            if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
                return mpArg;
            }
        }

        Object arg = null;
        MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
        if (multipartRequest != null) {
            List<MultipartFile> files = multipartRequest.getFiles(name);
            if (!files.isEmpty()) {
                arg = (files.size() == 1 ? files.get(0) : files);
            }
        }
        if (arg == null) { //获取请求参数第一个
            String[] paramValues = request.getParameterValues(name);
            if (paramValues != null) {
                arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
            }
        }
        return arg;
    }
{%endcodeblock%}
- 总结
  - 支持el表达式
  - 形参条件
    - `@RequestPara`修饰的`Map`,但是不能有name属性
    - `MultipartFile`或者`Part`不带有`@RequestPart`注解
    - useDefaultResolution==true时,可以使用不带`@RequestPara`的simple类型
##### CookieValue    
`ServletCookieValueMethodArgumentResolver` 略
- 总结
  - 支持el
  - 形参一般为`Cookie`或`String`
##### MatrixVariable  
`MatrixVariableMethodArgumentResolver`略
- 总结
  - 不支持el
  - `@MatrixVariable`修改的`Map`,且含有name属性
  - 要使用这个配置类,实际上是为了开启`Mapping`中去除矩阵变量的分隔符`.`
    ```java
    protected void configurePathMatch(PathMatchConfigurer configurer) {
        UrlPathHelper urlPathHelper=new UrlPathHelper();
        urlPathHelper.setRemoveSemicolonContent(false);
        configurer.setUrlPathHelper(urlPathHelper);
    }
    ```
##### SessionAttribute
`SessionAttributeMethodArgumentResolver`,略
- 总结    
  - 不支持el
  - `@SessionAttribute`修饰,获取session属性
  - 和`@SessionAttributes`区分开
##### ExpressionValue
- 代码逻辑
{%codeblock lang:java ExpressionValueMethodArgumentResolver%}
public class ExpressionValueMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver {

    /**
     * Create a new {@link ExpressionValueMethodArgumentResolver} instance.
     * @param beanFactory a bean factory to use for resolving  ${...}
     * placeholder and #{...} SpEL expressions in default values;
     * or {@code null} if default values are not expected to contain expressions
     */
    public ExpressionValueMethodArgumentResolver(@Nullable ConfigurableBeanFactory beanFactory) {
        super(beanFactory);
    }


    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(Value.class);
    }

    @Override
    protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
        Value ann = parameter.getParameterAnnotation(Value.class);
        Assert.state(ann != null, "No Value annotation");
        return new ExpressionValueNamedValueInfo(ann);
    }

    @Override
    @Nullable
    protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
        // No name to resolve
    //父类解析,实际入参处理返回的就是@Value el处理结果
        return null;
    }

    @Override
    protected void handleMissingValue(String name, MethodParameter parameter) throws ServletException {
        throw new UnsupportedOperationException("@Value is never required: " + parameter.getMethod());
    }

  //由于@Value并不是NamedValue,此处如此生成了name vlaue required
    private static final class ExpressionValueNamedValueInfo extends NamedValueInfo {

        private ExpressionValueNamedValueInfo(Value annotation) {
            super("@Value", false, annotation.value());
        }
    }

}
{%endcodeblock%}

- 总结
  - 和ioc过程中@Value使用一致
##### PathVariable
`PathVariableMethodArgumentResolver`
- 总结
  - 形参`@PathVariable`修饰
    - `Map`带有name属性
    - `String`
#### ModelAttribute注解
该注解作为形参使用时说明要从`MvcContainer`中尝试获取`model`,若没有则会进入创建流程
改处理器同时也是一个`返回值处理器`,完成了`获取model`,`创建model`,`验证`的功能
- 关系
{%asset_img ModelAttribute处理器.png ModelAttribute处理器%}
##### 默认实现
- 代码逻辑
{%codeblock lang:java ModelAttributeMethodProcessor%}
  public ModelAttributeMethodProcessor(boolean annotationNotRequired) {
    //表示是否能够处理不带@ModelAttribute注解的形参
        this.annotationNotRequired = annotationNotRequired;
    }
  public boolean supportsParameter(MethodParameter parameter) {
        return (parameter.hasParameterAnnotation(ModelAttribute.class)
                (this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
    }
  //处理逻辑
  public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

        Assert.state(mavContainer != null, "ModelAttributeMethodProcessor requires ModelAndViewContainer");
        Assert.state(binderFactory != null, "ModelAttributeMethodProcessor requires WebDataBinderFactory");

    //获取@ModelAttribute#name,若不存在则获取类型名  
        String name = ModelFactory.getNameForParameter(parameter);
        ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
        if (ann != null) { //绑定情况
            mavContainer.setBinding(name, ann.binding());
        }

        Object attribute = null;
        BindingResult bindingResult = null;

        if (mavContainer.containsAttribute(name)) { //若缓存了则直接获取
            attribute = mavContainer.getModel().get(name);
        }
        else { //创造
            // Create attribute instance
            try {
                attribute = createAttribute(name, parameter, binderFactory, webRequest);
            }
            catch (BindException ex) {
                if (isBindExceptionRequired(parameter)) {
                    // No BindingResult parameter -> fail with BindException
                    throw ex;
                }
                // Otherwise, expose null/empty value and associated BindingResult
                if (parameter.getParameterType() == Optional.class) {
                    attribute = Optional.empty();
                }
                bindingResult = ex.getBindingResult();
            }
        }

        if (bindingResult == null) { //说明创造attr过程没有异常
            // Bean property binding and validation;
            // skipped in case of binding failure on construction.
      //binding和验证
            WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name); //此处的databinder中是有attr
            if (binder.getTarget() != null) {
                if (!mavContainer.isBindingDisabled(name)) { //mvc唯一判断binding状态位置,若false则跳过改attr的binding阶段
          //WebRequestDataBinder的绑定阶段
          //会将请求参数生成pvs,若多文件处理pvs则会附带Mutipart||Part 的pv
          //然后进行bind操作
                    bindRequestParameters(binder, webRequest);
                }
                validateIfApplicable(binder, parameter);//验证阶段
                if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                    throw new BindException(binder.getBindingResult());
                }
            }
            // Value type adaptation, also covering java.util.Optional
            if (!parameter.getParameterType().isInstance(attribute)) {//是否还要进行类型转换
                attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
            }
            bindingResult = binder.getBindingResult();
        }

        // Add resolved attribute and BindingResult at the end of the mode
    //这个map会包含两个entry
    // name,attrValue
    // name+bindingResult#prefix,BindingResult
    // 第二个属性是用来给处理形参BindingResult的
        Map<String, Object> bindingResultModel = bindingResult.getModel();
        mavContainer.removeAttributes(bindingResultModel);
        mavContainer.addAllAttributes(bindingResultModel);

        return attribute;
    }

  //-------------------------创造attr----------------------------
  protected Object createAttribute(String attributeName, MethodParameter parameter,
            WebDataBinderFactory binderFactory, NativeWebRequest webRequest) throws Exception {

        MethodParameter nestedParameter = parameter.nestedIfOptional();
        Class<?> clazz = nestedParameter.getNestedParameterType();
    //获取构造器,简单来说若Bean中只有一个构造器则返回这个构造器
    //若构造器多于2个则尝试获取无参构造器
    //若无法获取无参,则异常
        Constructor<?> ctor = BeanUtils.findPrimaryConstructor(clazz);
        if (ctor == null) {
            Constructor<?>[] ctors = clazz.getConstructors();
            if (ctors.length == 1) {
                ctor = ctors[0];
            }
            else {
                try {
                    ctor = clazz.getDeclaredConstructor();
                }
                catch (NoSuchMethodException ex) {
                    throw new IllegalStateException("No primary or default constructor found for " + clazz, ex);
                }
            }
        }

        Object attribute = constructAttribute(ctor, attributeName, parameter, binderFactory, webRequest);
        if (parameter != nestedParameter) {
            attribute = Optional.of(attribute);
        }
        return attribute;
    }
  //利用构造器创建属性
  protected Object constructAttribute(Constructor<?> ctor, String attributeName, MethodParameter parameter,
            WebDataBinderFactory binderFactory, NativeWebRequest webRequest) throws Exception {
    //[1]废弃函数,默认返回null    
        Object constructed = constructAttribute(ctor, attributeName, binderFactory, webRequest);
        if (constructed != null) {
            return constructed;
        }
    //[1!]
    //[2]无参构造
        if (ctor.getParameterCount() == 0) {
            // A single default constructor -> clearly a standard JavaBeans arrangement.
            return BeanUtils.instantiateClass(ctor);
        }
    //[2!]
        // A single data class constructor -> resolve constructor arguments from request parameters.
    //从请求中获取和构造器形参能够匹配上的属性,并且转换设置到对应属性上
    //[3] 处理构造器上参数名称
        ConstructorProperties cp = ctor.getAnnotation(ConstructorProperties.class);
        String[] paramNames = (cp != null ? cp.value() : parameterNameDiscoverer.getParameterNames(ctor));
        Assert.state(paramNames != null, () -> "Cannot resolve parameter names for constructor " + ctor);
        Class<?>[] paramTypes = ctor.getParameterTypes();
        Assert.state(paramNames.length == paramTypes.length,
                () -> "Invalid number of parameter names: " + paramNames.length + " for constructor " + ctor);

        Object[] args = new Object[paramTypes.length];
        WebDataBinder binder = binderFactory.createBinder(webRequest, null, attributeName); //注意此处的DataBinder中是没有Object的
    //dataBinder中的两个前缀
        String fieldDefaultPrefix = binder.getFieldDefaultPrefix();
        String fieldMarkerPrefix = binder.getFieldMarkerPrefix();
        boolean bindingFailure = false;
        Set<String> failedParams = new HashSet<>(4);
     //[3!]

     //[4] 从请求中获取对应参数值,并进行转换
        for (int i = 0; i < paramNames.length; i++) {
            String paramName = paramNames[i];
            Class<?> paramType = paramTypes[i];
            Object value = webRequest.getParameterValues(paramName);
            if (value == null) {
                if (fieldDefaultPrefix != null) {
                    value = webRequest.getParameter(fieldDefaultPrefix + paramName);
                }
                if (value == null && fieldMarkerPrefix != null) {
                    if (webRequest.getParameter(fieldMarkerPrefix + paramName) != null) {
                        value = binder.getEmptyValue(paramType);
                    }
                }
            }
            try {
                MethodParameter methodParam = new FieldAwareConstructorParameter(ctor, i, paramName);
                if (value == null && methodParam.isOptional()) {
                    args[i] = (methodParam.getParameterType() == Optional.class ? Optional.empty() : null);
                }
                else {
                    args[i] = binder.convertIfNecessary(value, paramType, methodParam);
                }
            }
            catch (TypeMismatchException ex) {
                ex.initPropertyName(paramName);
                args[i] = value;
                failedParams.add(paramName);
                binder.getBindingResult().recordFieldValue(paramName, paramType, value);
                binder.getBindingErrorProcessor().processPropertyAccessException(ex, binder.getBindingResult());
                bindingFailure = true;
            }
        }
    //[4!]
    //[5]出现异常后处理binder,并依次对构造器参数进行验证
        if (bindingFailure) {
            BindingResult result = binder.getBindingResult();
            for (int i = 0; i < paramNames.length; i++) {
                String paramName = paramNames[i];
                if (!failedParams.contains(paramName)) {
                    Object value = args[i];
                    result.recordFieldValue(paramName, paramTypes[i], value);
                    validateValueIfApplicable(binder, parameter, ctor.getDeclaringClass(), paramName, value);
                }
            }
            throw new BindException(result);
        }
    //[5!]
    //反射构造attr对象
        return BeanUtils.instantiateClass(ctor, args);
    }
{%endcodeblock%}
- 总结
  - 典型的使用`@ModelAttribute`||或自定义类型
    - 从请求参数填充属性
    - 可以进行`验证`
##### Servlet实现
- 代码逻辑
{%codeblock lang:java ServletModelAttributeMethodProcessor%}
public class ServletModelAttributeMethodProcessor extends ModelAttributeMethodProcessor {
  //子类的逻辑
  protected final Object createAttribute(String attributeName, MethodParameter parameter,
            WebDataBinderFactory binderFactory, NativeWebRequest request) throws Exception {

        String value = getRequestValueForAttribute(attributeName, request);
        if (value != null) { //若能够获得字符串值,则尝试从字符串到value的一次转换
            Object attribute = createAttributeFromRequestValue(
                    value, attributeName, parameter, binderFactory, request);
            if (attribute != null) {
                return attribute;
            }
        }
    //若没有字符串->Object转换的机会,那么逻辑就和普通的无区别
        return super.createAttribute(attributeName, parameter, binderFactory, request);
    }
}  

//尝试从请求和填充path中直接获取能和 name匹配的字符串值
protected String getRequestValueForAttribute(String attributeName, NativeWebRequest request) {
        Map<String, String> variables = getUriTemplateVariables(request);
        String variableValue = variables.get(attributeName);
        if (StringUtils.hasText(variableValue)) {
            return variableValue;
        }
        String parameterValue = request.getParameter(attributeName);
        if (StringUtils.hasText(parameterValue)) {
            return parameterValue;
        }
        return null;
    }

//这里就需要自定义ConversionService来进行这一步的转换操作
protected Object createAttributeFromRequestValue(String sourceValue, String attributeName,
            MethodParameter parameter, WebDataBinderFactory binderFactory, NativeWebRequest request)
            throws Exception {

        DataBinder binder = binderFactory.createBinder(request, null, attributeName);
        ConversionService conversionService = binder.getConversionService();
        if (conversionService != null) {
            TypeDescriptor source = TypeDescriptor.valueOf(String.class);
            TypeDescriptor target = new TypeDescriptor(parameter);
            if (conversionService.canConvert(source, target)) {
                return binder.convertIfNecessary(sourceValue, parameter.getParameterType(), parameter);
            }
        }
        return null;
    }  
{%endcodeblock%}
- 总结
  - 特有的定义从请求中获取`String`
  - 用户需定义`ConversionService`来完善功能
#### MessageConverter处理器
该类解析器在处理入参时使用用了`HttpMessageConverter`,用户可以自定义
- 转换器定义
{%codeblock lang:java HttpMessageConverter%}
public interface HttpMessageConverter<T> {
  //读取
  boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
  //写入
  boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);
  //支持的mime类型
  List<MediaType> getSupportedMediaTypes();
  //从请求中获取信息并转换
  T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException;
  //t->out中写出,一般用于回参处理    
  void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
              throws IOException, HttpMessageNotWritableException;    
}
{%endcodeblock%}
##### 抽象逻辑
- 代码逻辑
{%codeblock lang:java AbstractMessageConverterMethodArgumentResolver%}
public abstract class AbstractMessageConverterMethodArgumentResolver implements HandlerMethodArgumentResolver {
  private static final Set<HttpMethod> SUPPORTED_METHODS =
            EnumSet.of(HttpMethod.POST, HttpMethod.PUT, HttpMethod.PATCH);

    private static final Object NO_VALUE = new Object();


    protected final Log logger = LogFactory.getLog(getClass());
  //转换器
    protected final List<HttpMessageConverter<?>> messageConverters;
  //支持的mime类型
    protected final List<MediaType> allSupportedMediaTypes;
  //请求增强类,跟注解@RequestAdvice和@ResponseAdvice有关
    private final RequestResponseBodyAdviceChain advice;
  //构造
  public AbstractMessageConverterMethodArgumentResolver(List<HttpMessageConverter<?>> converters,
            @Nullable List<Object> requestResponseBodyAdvice) {

        Assert.notEmpty(converters, "'messageConverters' must not be empty");
        this.messageConverters = converters;
        this.allSupportedMediaTypes = getAllSupportedMediaTypes(converters);
        this.advice = new RequestResponseBodyAdviceChain(requestResponseBodyAdvice);
    }
}  

//转换器中获取支持的MediaType
private static List<MediaType> getAllSupportedMediaTypes(List<HttpMessageConverter<?>> messageConverters) {
        Set<MediaType> allSupportedMediaTypes = new LinkedHashSet<>();
        for (HttpMessageConverter<?> messageConverter : messageConverters) {
            allSupportedMediaTypes.addAll(messageConverter.getSupportedMediaTypes());
        }
        List<MediaType> result = new ArrayList<>(allSupportedMediaTypes);
        MediaType.sortBySpecificity(result);
        return Collections.unmodifiableList(result);
    }
//--------------------------------请求中读取并转换--------------------------
protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter,
            Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
    //包含请求头和请求体数据    
        HttpInputMessage inputMessage = createInputMessage(webRequest);
        return readWithMessageConverters(inputMessage, parameter, paramType);
    }  
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
            Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

    //[1]请求MediaType    
        MediaType contentType;
        boolean noContentType = false;
        try {
            contentType = inputMessage.getHeaders().getContentType();
        }
        catch (InvalidMediaTypeException ex) {
            throw new HttpMediaTypeNotSupportedException(ex.getMessage());
        }
        if (contentType == null) {
            noContentType = true;
            contentType = MediaType.APPLICATION_OCTET_STREAM;
        }
    //[1!]
        Class<?> contextClass = parameter.getContainingClass();
        Class<T> targetClass = (targetType instanceof Class ? (Class<T>) targetType : null);
        if (targetClass == null) {
            ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
            targetClass = (Class<T>) resolvableType.resolve();
        }

        HttpMethod httpMethod = (inputMessage instanceof HttpRequest ? ((HttpRequest) inputMessage).getMethod() : null);
        Object body = NO_VALUE;

        EmptyBodyCheckingHttpInputMessage message;
        try {
            message = new EmptyBodyCheckingHttpInputMessage(inputMessage);
      //[2]遍历转换器
            for (HttpMessageConverter<?> converter : this.messageConverters) {
                Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
                GenericHttpMessageConverter<?> genericConverter =
                        (converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
                if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
                        (targetClass != null && converter.canRead(targetClass, contentType))) {
                    if (message.hasBody()) { //请求有请求体,则开始处理
                        HttpInputMessage msgToUse =
                                getAdvice().beforeBodyRead(message, parameter, targetType, converterType); //增强点
                        body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
                                ((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));//读取转换
                        body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);//增强点
                    }
                    else {
                        body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);//空body处理
                    }
                    break;
                }
            }
        }
        catch (IOException ex) {
            throw new HttpMessageNotReadableException("I/O error while reading input message", ex, inputMessage);
        }
   //[2!]

   //[3]返回处理体
        if (body == NO_VALUE) {
            if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) ||
                    (noContentType && !message.hasBody())) {
                return null;
            }
            throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
        }

        MediaType selectedContentType = contentType;
        Object theBody = body;
        LogFormatUtils.traceDebug(logger, traceOn -> {
            String formatted = LogFormatUtils.formatValue(theBody, !traceOn);
            return "Read \"" + selectedContentType + "\" to [" + formatted + "]";
        });

        return body;
    //[3!]
    }
{%endcodeblock%}
- 总结
  - 由构造器指定转换器列表
  - 拥有`RequestAdvice`和`ResponseAdvice`调用点
  - http请求必须含有请求体
  - 这种转换器都是从body获取数据,因此就不是用来处理能够简单获取key-value的数据,可以和`@ModelAttribute`
  处理器做一下对比
##### RequestPart
- 代码逻辑
  {%codeblock lang:java RequestPartMethodArgumentResolver%}
  public class RequestPartMethodArgumentResolver extends AbstractMessageConverterMethodArgumentResolver {
    //这里的逻辑和RequestPara是相反的
    public boolean supportsParameter(MethodParameter parameter) {
        if (parameter.hasParameterAnnotation(RequestPart.class)) {
            return true;
        }
        else {
            if (parameter.hasParameterAnnotation(RequestParam.class)) {
                return false;
            }
            return MultipartResolutionDelegate.isMultipartArgument(parameter.nestedIfOptional());
        }
    }
  }
  //处理
  public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest request, @Nullable WebDataBinderFactory binderFactory) throws Exception {

        HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
        Assert.state(servletRequest != null, "No HttpServletRequest");

        RequestPart requestPart = parameter.getParameterAnnotation(RequestPart.class);
        boolean isRequired = ((requestPart == null || requestPart.required()) && !parameter.isOptional());

        String name = getPartName(parameter, requestPart);
        parameter = parameter.nestedIfOptional();
        Object arg = null;
    //交由多文件代理处理,这里逻辑和@RequestPara一致
        Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
        if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
            arg = mpArg;
        }
        else { //若没法处理则交予处理器处理
            try {
                HttpInputMessage inputMessage = new RequestPartServletServerHttpRequest(servletRequest, name);
                arg = readWithMessageConverters(inputMessage, parameter, parameter.getNestedGenericParameterType());
                if (binderFactory != null) {
                    WebDataBinder binder = binderFactory.createBinder(request, arg, name);
                    if (arg != null) {
                        validateIfApplicable(binder, parameter);
                        if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                            throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
                        }
                    }
                    if (mavContainer != null) {
                        mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
                    }
                }
            }
            catch (MissingServletRequestPartException | MultipartException ex) {
                if (isRequired) {
                    throw ex;
                }
            }
        }

        if (arg == null && isRequired) {
            if (!MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
                throw new MultipartException("Current request is not a multipart request");
            }
            else {
                throw new MissingServletRequestPartException(name);
            }
        }
        return adaptArgumentIfNecessary(arg, parameter);
    }
  {%endcodeblock%}
- 总结
  - 带上`@ReuqestPart`和`@RequestPara`区分
##### RequestResponse
转换器也是回参处理器,常见的json转换,xml转换都是这个完成的
- 代码逻辑
{%codeblock lang:java %}
public boolean supportsReturnType(MethodParameter returnType) {
        return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
                returnType.hasMethodAnnotation(ResponseBody.class));
    }

    /**
     * Throws MethodArgumentNotValidException if validation fails.
     * @throws HttpMessageNotReadableException if {@link RequestBody#required()}
     * is {@code true} and there is no body content or if there is no suitable
     * converter to read the content with.
     */
    @Override
    public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

        parameter = parameter.nestedIfOptional();
        Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
        String name = Conventions.getVariableNameForParameter(parameter);

        if (binderFactory != null) {
            WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
            if (arg != null) {
                validateIfApplicable(binder, parameter);
                if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                    throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
                }
            }
            if (mavContainer != null) {
                mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
            }
        }

        return adaptArgumentIfNecessary(arg, parameter);
    }
{%endcodeblock%}
- 总结
  - 形参为`@Requestbody`
  - 逻辑简单
  ##### HttpEntity
  转换器也是回参处理器
  - 总结
    - `HttpEntity`和`RequestEntity`支持
  ### 回参处理器
  #### 模型处理
  ##### ModelAttribute
  - 代码逻辑
  {%codeblock lang:java ModelAttributeMethodProcessor%}
  public boolean supportsReturnType(MethodParameter returnType) {
          return (returnType.hasMethodAnnotation(ModelAttribute.class) ||
                  (this.annotationNotRequired && !BeanUtils.isSimpleProperty(returnType.getParameterType())));
      }

      /**
       * Add non-null return values to the {@link ModelAndViewContainer}.
       */
      @Override
      public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

          if (returnValue != null) {
              String name = ModelFactory.getNameForReturnValue(returnValue, returnType);
              mavContainer.addAttribute(name, returnValue);
          }
      }
  {%endcodeblock%}
  - 总结
    - 该Method有@ModelAttribute修饰||或者采用自定义类型生效
    - 将该返回值加入`mvcContainer`
    ##### MapMethod
    - 代码逻辑
    {%codeblock lang:java MapMethodProcessor%}    
    public boolean supportsReturnType(MethodParameter returnType) {
            return Map.class.isAssignableFrom(returnType.getParameterType());
        }

        @Override
        @SuppressWarnings({"rawtypes", "unchecked"})
        public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
                ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

            if (returnValue instanceof Map){
                mavContainer.addAllAttributes((Map) returnValue);
            }
            else if (returnValue != null) {
                // should not happen
                throw new UnsupportedOperationException("Unexpected return type: " +
                        returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
            }
        }
    {%endcodeblock%}
    - 总结
      - 返回值必须为`Map`子类
      - 符合则加入`mvcContainer`  
  #### 视图处理   
  ##### ViewNameMethod
  - 代码逻辑
  {%codeblock lang:java ViewNameMethodReturnValueHandler%}  
  public boolean supportsReturnType(MethodParameter returnType) {
          Class<?> paramType = returnType.getParameterType();
          return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
      }

      @Override
      public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

          if (returnValue instanceof CharSequence) {
              String viewName = returnValue.toString();
              mavContainer.setViewName(viewName);
              if (isRedirectViewName(viewName)) {
                  mavContainer.setRedirectModelScenario(true);
              }
          }
          else if (returnValue != null) {
              // should not happen
              throw new UnsupportedOperationException("Unexpected return type: " +
                      returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
          }
      }
  {%endcodeblock%}
  - 总结
      - 返回值必须是`CharSequence`子类||void
      - 处理方式是给`mvcContainer#viewName`设置
  ##### ViewMethod
  - 代码逻辑
  {%codeblock lang:java ViewMethodReturnValueHandler%}
  public class ViewMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

      @Override
      public boolean supportsReturnType(MethodParameter returnType) {
          return View.class.isAssignableFrom(returnType.getParameterType());
      }

      @Override
      public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

          if (returnValue instanceof View) {
              View view = (View) returnValue;
              mavContainer.setView(view);
              if (view instanceof SmartView && ((SmartView) view).isRedirectView()) {
                  mavContainer.setRedirectModelScenario(true);
              }
          }
          else if (returnValue != null) {
              // should not happen
              throw new UnsupportedOperationException("Unexpected return type: " +
                      returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
          }
      }

  }

  {%endcodeblock%}
  - 总结
    - 返回值要求为`View`对象
    - 设置`MvcContainer#View`属性
  #####  ModelAndViewMethod
  - 代码逻辑
  {%codeblock lang:java ModelAndViewMethodReturnValueHandler%}   
  public class ModelAndViewMethodReturnValueHandler implements HandlerMethodReturnValueHandler {
    public boolean supportsReturnType(MethodParameter returnType) {
          return ModelAndView.class.isAssignableFrom(returnType.getParameterType());
      }

      @Override
      public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

          if (returnValue == null) {
              mavContainer.setRequestHandled(true);
              return;
          }

          ModelAndView mav = (ModelAndView) returnValue;
          if (mav.isReference()) {
              String viewName = mav.getViewName();
              mavContainer.setViewName(viewName);
              if (viewName != null && isRedirectViewName(viewName)) {
                  mavContainer.setRedirectModelScenario(true);
              }
          }
          else {
              View view = mav.getView();
              mavContainer.setView(view);
              if (view instanceof SmartView && ((SmartView) view).isRedirectView()) {
                  mavContainer.setRedirectModelScenario(true);
              }
          }
          mavContainer.setStatus(mav.getStatus());
          mavContainer.addAllAttributes(mav.getModel());
      }
  }
  {%endcodeblock%}
  - 总结
    - 形参`ModelAndView`
    - 支持重定向类型
  ##### ModelAndViewResolver
  - 代码逻辑
  {%codeblock lang:java ModelAndViewResolverMethodReturnValueHandler%}  
  public class ModelAndViewResolverMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

      @Nullable
      private final List<ModelAndViewResolver> mavResolvers;

      private final ModelAttributeMethodProcessor modelAttributeProcessor = new ModelAttributeMethodProcessor(true);


      /**
       * Create a new instance.
       */
      public ModelAndViewResolverMethodReturnValueHandler(@Nullable List<ModelAndViewResolver> mavResolvers) {
          this.mavResolvers = mavResolvers;
      }


      /**
       * Always returns {@code true}. See class-level note.
       */
      @Override
      public boolean supportsReturnType(MethodParameter returnType) {
          return true;
      }

      @Override
      public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
              ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

          if (this.mavResolvers != null) {
              for (ModelAndViewResolver mavResolver : this.mavResolvers) {
                  Class<?> handlerType = returnType.getContainingClass();
                  Method method = returnType.getMethod();
                  Assert.state(method != null, "No handler method");
                  ExtendedModelMap model = (ExtendedModelMap) mavContainer.getModel();
                  ModelAndView mav = mavResolver.resolveModelAndView(method, handlerType, returnValue, model, webRequest);
                  if (mav != ModelAndViewResolver.UNRESOLVED) {
                      mavContainer.addAllAttributes(mav.getModel());
                      mavContainer.setViewName(mav.getViewName());
                      if (!mav.isReference()) {
                          mavContainer.setView(mav.getView());
                      }
                      return;
                  }
              }
          }

          // No suitable ModelAndViewResolver...
          if (this.modelAttributeProcessor.supportsReturnType(returnType)) {
              this.modelAttributeProcessor.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
          }
          else {
              throw new UnsupportedOperationException("Unexpected return type: " +
                      returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
          }
      }

  }

  {%endcodeblock%}
  - 总结
    - 回参随意的原因是该回参处理在list的末尾,一般不会调用到这里
    - 用户来实现`ModelAndViewResolver`
  #### HttpMessageConverter
  这是mvc中较为常用的类型,通过`HttpMessageConverter`来处理转换类型
  ##### 抽象逻辑
  {%codeblock lang:java AbstractMessageConverterMethodProcessor%}  

  //子类一般会重写,来体统inputMessage和outputMessage
  protected <T> void writeWithMessageConverters(T value, MethodParameter returnType, NativeWebRequest webRequest)
            throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

        ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
        ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
        writeWithMessageConverters(value, returnType, inputMessage, outputMessage);
    }

  protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
            ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
            throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

        Object body;
        Class<?> valueType;
        Type targetType;

    //[1]确定类型
        if (value instanceof CharSequence) {
            body = value.toString();
            valueType = String.class;
            targetType = String.class;
        }
        else {
            body = value;
            valueType = getReturnValueType(body, returnType);
            targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
        }
    //[1!]

    //[2] 专门用来处理 Accept-Ranges
        if (isResourceType(value, returnType)) {
            outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
            if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
                    outputMessage.getServletResponse().getStatus() == 200) {
                Resource resource = (Resource) value;
                try {
                    List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
                    outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
                    body = HttpRange.toResourceRegions(httpRanges, resource);
                    valueType = body.getClass();
                    targetType = RESOURCE_REGION_LIST_TYPE;
                }
                catch (IllegalArgumentException ex) {
                    outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
                    outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
                }
            }
        }
    //[2!]

    //[3] 尝试从Response#header获取类型,若没有则通过ContentNegotiationManager解析
        MediaType selectedMediaType = null;
        MediaType contentType = outputMessage.getHeaders().getContentType();
        boolean isContentTypePreset = contentType != null && contentType.isConcrete();
        if (isContentTypePreset) {
            if (logger.isDebugEnabled()) {
                logger.debug("Found 'Content-Type:" + contentType + "' in response");
            }
            selectedMediaType = contentType;
        }
        else {
            HttpServletRequest request = inputMessage.getServletRequest();
      //req#Accept
            List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);
      //@RequestMapping#produce 中获取
            List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

            if (body != null && producibleTypes.isEmpty()) {
                throw new HttpMessageNotWritableException(
                        "No converter found for return value of type: " + valueType);
            }
            List<MediaType> mediaTypesToUse = new ArrayList<>();
            for (MediaType requestedType : acceptableTypes) {
                for (MediaType producibleType : producibleTypes) {
                    if (requestedType.isCompatibleWith(producibleType)) {
                        mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
                    }
                }
            }
            if (mediaTypesToUse.isEmpty()) {
                if (body != null) {
                    throw new HttpMediaTypeNotAcceptableException(producibleTypes);
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
                }
                return;
            }
      //[3!]
      //[4]获取最合适的
            MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
            for (MediaType mediaType : mediaTypesToUse) {
                if (mediaType.isConcrete()) {
                    selectedMediaType = mediaType;
                    break;
                }
                else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
                    selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
                    break;
                }
            }

            if (logger.isDebugEnabled()) {
                logger.debug("Using '" + selectedMediaType + "', given " +
                        acceptableTypes + " and supported " + producibleTypes);
            }
        }
    //[4!]
    //[5] 调用HttpMessageConverter进行处理,并附带了advice#beforeBodyRead调用
        if (selectedMediaType != null) {
            selectedMediaType = selectedMediaType.removeQualityValue();
            for (HttpMessageConverter<?> converter : this.messageConverters) {
                GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
                        (GenericHttpMessageConverter<?>) converter : null);
                if (genericConverter != null ?
                        ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
                        converter.canWrite(valueType, selectedMediaType)) {
                    body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
                            (Class<? extends HttpMessageConverter<?>>) converter.getClass(),
                            inputMessage, outputMessage);
                    if (body != null) {
                        Object theBody = body;
                        LogFormatUtils.traceDebug(logger, traceOn ->
                                "Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
                        addContentDispositionHeader(inputMessage, outputMessage);
                        if (genericConverter != null) {
                            genericConverter.write(body, targetType, selectedMediaType, outputMessage);
                        }
                        else {
                            ((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
                        }
                    }
                    else {
                        if (logger.isDebugEnabled()) {
                            logger.debug("Nothing to write: null body");
                        }
                    }
                    return; //处理结束
                }
            }
        }
    //[5!]

    //[6!] 若进行到这里则说明没有匹配任何处理器
        if (body != null) {
            Set<MediaType> producibleMediaTypes =
                    (Set<MediaType>) inputMessage.getServletRequest()
                            .getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

            if (isContentTypePreset || !CollectionUtils.isEmpty(producibleMediaTypes)) {
                throw new HttpMessageNotWritableException(
                        "No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
            }
            throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
        }
    }
  //[6]
  {%endcodeblock%}
  - 总结
    - 返回数据类型由`ContentNegotiationManager`和`@RequestMapping#produce`提供
    - 提供了处理`context-range`的默认方法
    - 提供了`context-dispoition`的方法
    - 执行转换前会执行`advice#beforeBodyWrite`  
  ##### RequestResponseBody
  - 代码逻辑
    {%codeblock lang:java RequestResponseBodyMethodProcessor%}
    public boolean supportsReturnType(MethodParameter returnType) {
            return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
                    returnType.hasMethodAnnotation(ResponseBody.class));
        }

    public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
            ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
            throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

        mavContainer.setRequestHandled(true);//处理结束标志
        ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
        ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

        // Try even with null return value. ResponseBodyAdvice could get involved.
        writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
    }
    {%endcodeblock%}
  - 总结
      - controller函数上有`@RequestBody`||控制器有`@RequestBody`,如`@RestController`
      - 后续不会发生更新`bindingResult`
  ##### HttpEntity
  - 代码逻辑
  {%codeblock lang:java HttpEntityMethodProcessor%}
  public boolean supportsReturnType(MethodParameter returnType) {
        return (HttpEntity.class.isAssignableFrom(returnType.getParameterType()) &&
                !RequestEntity.class.isAssignableFrom(returnType.getParameterType()));
    }

  public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
            ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    //[1]设置标志,创建in||out    
        mavContainer.setRequestHandled(true);
        if (returnValue == null) {
            return;
        }

        ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
        ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
    //[1!]
        Assert.isInstanceOf(HttpEntity.class, returnValue);
        HttpEntity<?> responseEntity = (HttpEntity<?>) returnValue;
    //[2]将Entity中的header设置到out中
        HttpHeaders outputHeaders = outputMessage.getHeaders();
        HttpHeaders entityHeaders = responseEntity.getHeaders();
        if (!entityHeaders.isEmpty()) {
            entityHeaders.forEach((key, value) -> {
                if (HttpHeaders.VARY.equals(key) && outputHeaders.containsKey(HttpHeaders.VARY)) { //此处涉及到了VARY头
                    List<String> values = getVaryRequestHeadersToAdd(outputHeaders, entityHeaders);
                    if (!values.isEmpty()) {
                        outputHeaders.setVary(values);
                    }
                }
                else {
                    outputHeaders.put(key, value);
                }
            });
        }
    //[2]
    //[3]处理200和3xx请求
        if (responseEntity instanceof ResponseEntity) {
            int returnStatus = ((ResponseEntity<?>) responseEntity).getStatusCodeValue();
            outputMessage.getServletResponse().setStatus(returnStatus);
            if (returnStatus == 200) {
                if (SAFE_METHODS.contains(inputMessage.getMethod())
                        && isResourceNotModified(inputMessage, outputMessage)) { //get或者head请求,并且资源未改变
                    // Ensure headers are flushed, no body should be written.
                    outputMessage.flush();
                    ShallowEtagHeaderFilter.disableContentCaching(inputMessage.getServletRequest());
                    // Skip call to converters, as they may update the body.
                    return;
                }
            }
            else if (returnStatus / 100 == 3) { //处理重定向属性
                String location = outputHeaders.getFirst("location");
                if (location != null) {
                    saveFlashAttributes(mavContainer, webRequest, location);
                }
            }
        }
    //[3!]
    //[4]body作为value,调用转换器代码
        // Try even with null body. ResponseBodyAdvice could get involved.
        writeWithMessageConverters(responseEntity.getBody(), returnType, inputMessage, outputMessage);

        // Ensure headers are flushed even if no body was written.
        outputMessage.flush();
    //[4!]
    }
  {%endcodeblock%}
  - 总结
    - 返回值为`HttpEntity`但不也是`RequestHttpEntity`
    - 将`Entity#body`进行类型转换
  #### 异步和其他
  异步处理暂时不做分析
  ##### HttpHeaders
  - 代码逻辑
  {%codeblock lang:java HttpHeadersReturnValueHandler%}
  @Override
  public boolean supportsReturnType(MethodParameter returnType) {
    return HttpHeaders.class.isAssignableFrom(returnType.getParameterType());
  }

  @Override
  @SuppressWarnings("resource")
  public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    mavContainer.setRequestHandled(true);

    Assert.state(returnValue instanceof HttpHeaders, "HttpHeaders expected");
    HttpHeaders headers = (HttpHeaders) returnValue;

    if (!headers.isEmpty()) {
      HttpServletResponse servletResponse = webRequest.getNativeResponse(HttpServletResponse.class);
      Assert.state(servletResponse != null, "No HttpServletResponse");
      ServletServerHttpResponse outputMessage = new ServletServerHttpResponse(servletResponse);
      outputMessage.getHeaders().putAll(headers);
      outputMessage.getBody();  // flush headers
    }
  }

  public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
            ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    [1]  设置标志,创建input|output  
        mavContainer.setRequestHandled(true);
        if (returnValue == null) {
            return;
        }

        ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
        ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

        Assert.isInstanceOf(HttpEntity.class, returnValue);
        HttpEntity<?> responseEntity = (HttpEntity<?>) returnValue;
    //[1!]
    //[2]保证将返回值的header放置到outMessage中,并且此处处理了`VARY`
        HttpHeaders outputHeaders = outputMessage.getHeaders();
        HttpHeaders entityHeaders = responseEntity.getHeaders();
        if (!entityHeaders.isEmpty()) {
            entityHeaders.forEach((key, value) -> {
                if (HttpHeaders.VARY.equals(key) && outputHeaders.containsKey(HttpHeaders.VARY)) {
                    List<String> values = getVaryRequestHeadersToAdd(outputHeaders, entityHeaders);
                    if (!values.isEmpty()) {
                        outputHeaders.setVary(values);
                    }
                }
                else {
                    outputHeaders.put(key, value);
                }
            });
        }
    //[2!]

    //[3]
        if (responseEntity instanceof ResponseEntity) {
            int returnStatus = ((ResponseEntity<?>) responseEntity).getStatusCodeValue();
            outputMessage.getServletResponse().setStatus(returnStatus);
            if (returnStatus == 200) {//直接写出,不进行转换操作
                if (SAFE_METHODS.contains(inputMessage.getMethod())
                        && isResourceNotModified(inputMessage, outputMessage)) {
                    // Ensure headers are flushed, no body should be written.
                    outputMessage.flush();
                    ShallowEtagHeaderFilter.disableContentCaching(inputMessage.getServletRequest());
                    // Skip call to converters, as they may update the body.
                    return;
                }
            }
            else if (returnStatus / 100 == 3) { //重定向操作
                String location = outputHeaders.getFirst("location");
                if (location != null) {
                    saveFlashAttributes(mavContainer, webRequest, location);
                }
            }
        }
    /[3!]
    //[4]转换并保证header会被正确的返回
        // Try even with null body. ResponseBodyAdvice could get involved.
        writeWithMessageConverters(responseEntity.getBody(), returnType, inputMessage, outputMessage);

        // Ensure headers are flushed even if no body was written.
        outputMessage.flush();
    //[4!]
    }

  {%endcodeblock%}
  - 总结
    - 返回值`HttpHeaders`
    - 处理设置`RequestHandled=true`
