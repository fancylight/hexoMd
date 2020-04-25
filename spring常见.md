---
title: spring常见
date: 2020-02-11 11:14:49
tag:
- 框架
- spring
- spring组件
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
description: spring框架中常见的组件
---
### Bean相关
- 概念
    - BeanDefinition 是将ioc定义如xml,注解扫描的类转为成抽象描述,包含了用户定义的各种bean信息
    - BeanWrapper 是从该类抽离出的低级javaBean,包含了属性转换器,以及java 原信息
    - PropertyValue 仅仅是封装key-value 表示对象,存在于Bd内部,以及ioc属性转换过程中
    - PropertyEditor jdk提供的属性转换,A->B类型的转换,ioc逻辑中使用
#### xml解析
简而言之, 解析xml的类是BeanDefinitionParserDelegate完成的
- 标准bean标签的解析过程
    - xml例子
      ```xml
    <beans>
    <bean id="test" class="com.light.Foo">
    <description>
        this is a test
    </description>
    <property name="list">
        <list>
            <value>123</value>
            <value>123</value>
        </list>
    </property>
    <property name="files">
        <list>
            <value>d:/test/12.txt</value>
            <value>d:/test/12.txt</value>
        </list>
    </property>
    <meta key="key1" value="value1"/>
</bean>
</beans>
    ```

##### 基本元素
- 前言
解析bean的读取器为`BeanDefinitionReader`体系,并非是仅仅支持xml.
- 外部类
        {%codeblock lang:java DefaultBeanDefinitionDocumentReader%}
//该类是xml默认的 解析器实现类
protected void doRegisterBeanDefinitions(Element root) {
    //委托模式
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);

        if (this.delegate.isDefaultNamespace(root)) {
            String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                        profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
                // We cannot use Profiles.of(...) since profile expressions are not supported
                // in XML config. See SPR-12458 for details.
                if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                    }
                    return;
                }
            }
        }

        preProcessXml(root); //空,子类实现
        parseBeanDefinitions(root, this.delegate);
        postProcessXml(root); //空,子类实现

        this.delegate = parent;
}


//-------------------------------- 循环处理所有元素---------------------------------
    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    if (delegate.isDefaultNamespace(ele)) {
                        parseDefaultElement(ele, delegate); //处理不用引入如<context> 这样的默认标签
                    }
                    else {
                        delegate.parseCustomElement(ele); //处理额外的标签
                    }
                }
            }
        }
        else {
            delegate.parseCustomElement(root);
        }
    }

//-----------------------------------处理默认标签-------------------------------------------
  private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
  if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
    importBeanDefinitionResource(ele);
  }
  else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
    processAliasRegistration(ele);
  }
  else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
    processBeanDefinition(ele, delegate);
  }
  else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
    // recurse
    doRegisterBeanDefinitions(ele);
  }
}
//------------------------------------处理bean标签-----------------------------------------
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  //holder仅仅用来包含BeanDefition,内部的BeanDiftion 就是GenicBeanDeition
  //[1]  解析并创建gbdH
  BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
  //[1!]
  //[2] 注册bd到beanFacotry(Registry)中
  if (bdHolder != null) {
    bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
    try {
      // Register the final decorated instance.
      BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
    }
    catch (BeanDefinitionStoreException ex) {
      getReaderContext().error("Failed to register bean definition with name '" +
          bdHolder.getBeanName() + "'", ele, ex);
    }
    //[2!]
    //[3] 触发事件
    getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    //[3!]
  }


      {%endcodeblock%}
- 委托类
      {%codeblock lang:java BeanDefinitionParserDelegate%}
      //------------------------------------解析 并创建 gbd--------------------------------------
        public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
          return parseBeanDefinitionElement(ele, null);
        }
        @Nullable
        public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
          //[1] 处理id和name,别名,保持id唯一性
          String id = ele.getAttribute(ID_ATTRIBUTE);
          String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

          List<String> aliases = new ArrayList<>();
          if (StringUtils.hasLength(nameAttr)) {
            String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            aliases.addAll(Arrays.asList(nameArr));
          }

          String beanName = id;
          if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
            beanName = aliases.remove(0);
            if (logger.isTraceEnabled()) {
              logger.trace("No XML 'id' specified - using '" + beanName +
                  "' as bean name and " + aliases + " as aliases");
            }
          }

          if (containingBean == null) {
            checkNameUniqueness(beanName, aliases, ele);
          }
          //[1!]
          //[2]解析标签,并创建gbd
          AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
          //[2!]
          if (beanDefinition != null) {
            if (!StringUtils.hasText(beanName)) {
              try {
                if (containingBean != null) {
                  beanName = BeanDefinitionReaderUtils.generateBeanName(
                      beanDefinition, this.readerContext.getRegistry(), true);
                }
                else {
                  beanName = this.readerContext.generateBeanName(beanDefinition);
                  // Register an alias for the plain bean class name, if still possible,
                  // if the generator returned the class name plus a suffix.
                  // This is expected for Spring 1.2/2.0 backwards compatibility.
                  String beanClassName = beanDefinition.getBeanClassName();
                  if (beanClassName != null &&
                      beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                      !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                    aliases.add(beanClassName);
                  }
                }
                if (logger.isTraceEnabled()) {
                  logger.trace("Neither XML 'id' nor 'name' specified - " +
                      "using generated bean name [" + beanName + "]");
                }
              }
              catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
              }
            }
            String[] aliasesArray = StringUtils.toStringArray(aliases);
            return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
          }

          return null;
        }
        //----------------------------------实际创建gbd的代码---------------------------
        public AbstractBeanDefinition parseBeanDefinitionElement(
            Element ele, String beanName, @Nullable BeanDefinition containingBean) {

          this.parseState.push(new BeanEntry(beanName));

          String className = null;
          //class属性
          if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
            className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
          }
          String parent = null;
          //parent属性
          if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
          }

          try {
            // 简单创建gbd实例
            AbstractBeanDefinition bd = createBeanDefinition(className, parent);
            // 处理<bean> 标签中所有的属性,即attr,设置为abd(AbstractBeanDefinition) 对应的属性
            parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
            // 子类node中寻找 获取desc
            bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
            //  此处代码见下边
            parseMetaElements(ele, bd);
            // 子类node寻找<lookUp  name="" ,bean=""> 元素,添加MethodOverrride对象到bd中
            parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            // 寻找<ReplaceMethod name="" repalcer="">  元素 添加MEthodOverride对象到bd中
            parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
            // 处理<constructor-arg index= ...>
            // bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
            // valueHolder就是解析后创建的要添加到bd中的对象  
            parseConstructorArgElements(ele, bd);
            //处理子类中属性
            parsePropertyElements(ele, bd);
            //处理qualifier标签
            parseQualifierElements(ele, bd);
            //bd来源,如xml的路径  
            bd.setResource(this.readerContext.getResource());
            //
            bd.setSource(extractSource(ele));

            return bd;
          }
           //...

          return null;
        }
        //------------------------------处理meta元素-----------------------
        public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
          //从子类寻<meta>标签,获取meat的key以及value属性,作为bd#attribute
          NodeList nl = ele.getChildNodes();
          for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
              Element metaElement = (Element) node;
              //这里就能知道BeanMetadataAttributeAccessor的map是什么情况可以设置的
              String key = metaElement.getAttribute(KEY_ATTRIBUTE);
              String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
              //此处体现了BD体系中如何使用key-value
              BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
              attribute.setSource(extractSource(metaElement));
              attributeAccessor.addMetadataAttribute(attribute);
            }
          }
        }
        //----------------------------处理property----------------------------
        public void parsePropertyElement(Element ele, BeanDefinition bd) {
        String propertyName = ele.getAttribute(NAME_ATTRIBUTE); //获取该propery名
        if (!StringUtils.hasLength(propertyName)) {
            error("Tag 'property' must have a 'name' attribute", ele);
            return;
        }
        this.parseState.push(new PropertyEntry(propertyName));
        try {
            if (bd.getPropertyValues().contains(propertyName)) {
                error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
                return;
            }
      //此处是具体代码
            Object val = parsePropertyValue(ele, bd, propertyName);
      //返回值创建pv (key-vlaue) 类型
            PropertyValue pv = new PropertyValue(propertyName, val);
            parseMetaElements(ele, pv);
            pv.setSource(extractSource(ele));
      //添加到bd#pvs中,这个pvs在ioc中注值使用,非常关键
            bd.getPropertyValues().addPropertyValue(pv);
        }
        finally {
            this.parseState.pop();
        }
    }

  public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
        String elementName = (propertyName != null ?
                "<property> element for property '" + propertyName + "'" :
                "<constructor-arg> element");

        // Should only have one child element: ref, value, list, etc.
    //[1]获取当前元素(即Property) 子元素,最多最外层 只能存在一个如上
        NodeList nl = ele.getChildNodes();
        Element subElement = null;
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
                    !nodeNameEquals(node, META_ELEMENT)) {
                // Child element is what we're looking for.
                if (subElement != null) {
                    error(elementName + " must not contain more than one sub-element", ele);
                }
                else {
                    subElement = (Element) node;
                }
            }
        }
    //[1!]
    //[2] 不允许property元素带有value和ref属性, 或者任意属性并且有子元素
        boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
        boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
        if ((hasRefAttribute && hasValueAttribute) ||
                ((hasRefAttribute || hasValueAttribute) && subElement != null)) {
            error(elementName +
                    " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
        }
    //[2]
    //[3] 分情况讨论
        if (hasRefAttribute) {
            String refName = ele.getAttribute(REF_ATTRIBUTE);
            if (!StringUtils.hasText(refName)) {
                error(elementName + " contains empty 'ref' attribute", ele);
            }
            RuntimeBeanReference ref = new RuntimeBeanReference(refName);//创建一个引用
            ref.setSource(extractSource(ele));
            return ref;
        } //ref情况返回
        else if (hasValueAttribute) {
            TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
            valueHolder.setSource(extractSource(ele));
            return valueHolder;
        }//value 类型
        else if (subElement != null) {
            return parsePropertySubElement(subElement, bd); //复杂情况,即property存在子元素
        }
        else {
            // Neither child element nor "ref" or "value" attribute found.
            error(elementName + " must specify a ref or value", ele);
            return null;
        }
    [3!]
    }
  //-------------------------------处理property子元素------------------------------------
  public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
    //[1]
        if (!isDefaultNamespace(ele)) {
            return parseNestedCustomElement(ele, bd); //非标准元素,去调用custom逻辑,见下文
        }
    //[1!]
    //[2]
        else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
       //Bean标签,相当于递归调用
            BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
            if (nestedBd != null) {
                nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
            }
            return nestedBd;
        }
    //[2!]
    //[3]
        else if (nodeNameEquals(ele, REF_ELEMENT)) {
            // A generic reference to any name of any bean.
            String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
            boolean toParent = false;
            if (!StringUtils.hasLength(refName)) {
                // A reference to the id of another bean in a parent context.
                refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
                toParent = true;
                if (!StringUtils.hasLength(refName)) {
                    error("'bean' or 'parent' is required for <ref> element", ele);
                    return null;
                }
            }
            if (!StringUtils.hasText(refName)) {
                error("<ref> element contains empty target attribute", ele);
                return null;
            }
            RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
            ref.setSource(extractSource(ele));
            return ref;// <ref bean=""> 处理
        }
    //[4]
    //[5] <idref bean=""> 此处应该只能填写id
        else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
            return parseIdRefElement(ele);
        }
    //[5!]
    //[6] <value >处理
        else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
            return parseValueElement(ele, defaultValueType);
        }//[6!]
        else if (nodeNameEquals(ele, NULL_ELEMENT)) {
            // It's a distinguished null value. Let's wrap it in a TypedStringValue
            // object in order to preserve the source location.
            TypedStringValue nullHolder = new TypedStringValue(null);
            nullHolder.setSource(extractSource(ele));
            return nullHolder;
        }
    // 这几个类型 可能会继续引起子 解析,
        else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
            return parseArrayElement(ele, bd);
        }
        else if (nodeNameEquals(ele, LIST_ELEMENT)) {
            return parseListElement(ele, bd);
        }
        else if (nodeNameEquals(ele, SET_ELEMENT)) {
            return parseSetElement(ele, bd);
        }
        else if (nodeNameEquals(ele, MAP_ELEMENT)) {
            return parseMapElement(ele, bd);
        }
        else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
            return parsePropsElement(ele);
        }
        else {
            error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
            return null;
        }
    }
      {%endcodeblock%}
##### custom元素
###### NamespaceHandler体系
- 调用点
{%codeblock lang:java BeanDefinitionParserDelegate%}
    public BeanDefinition parseCustomElement(Element ele) {
        return parseCustomElement(ele, null);
    }
    @Nullable
    public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        String namespaceUri = getNamespaceURI(ele);
        if (namespaceUri == null) {
            return null;
        }
        //根据不同的元素使用不同的handler 来处理
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler == null) {
            error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        }
        return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
    }
{%endcodeblock%}
- 继承树
{%asset_img BeanDetintionParser.png%}
简单来看实际最多的是直接子类,另一类是
```java
    BeanDefinitionParser
        AbstractBeanDefinitionParser
            AbstractSingleBeanDefinitionParser
```

- 抽象逻辑
        {%codeblock lang:java AbstractBeanDefinitionParser%}
            public abstract class AbstractBeanDefinitionParser implements BeanDefinitionParser {
                public static final String ID_ATTRIBUTE = "id";
                public static final String NAME_ATTRIBUTE = "name";
                public final BeanDefinition parse(Element element, ParserContext parserContext) {
		AbstractBeanDefinition definition = parseInternal(element, parserContext); //子类实现解析bd
		if (definition != null && !parserContext.isNested()) {
			try {
                //尝试获得id
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
                //别名
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
                //注册
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) { //抽象类流出的接口
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				String msg = ex.getMessage();
				parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
				return null;
			}
		}
		return definition;
	}
        }
        {%endcodeblock%}
    - AbstractSingleBeanDefinitionParser

         该类表示此标签仅仅创建一个bd,例如`<context:property-placeholder`的结果
        {%codeblock lang:java AbstractSingleBeanDefinitionParser%}
        protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
        //bdBuidler用来创建bd,这里的getXX都是子类实现的
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
		String parentName = getParentName(element);//子类获取
		if (parentName != null) {
			builder.getRawBeanDefinition().setParentName(parentName);
		}
		Class<?> beanClass = getBeanClass(element);
		if (beanClass != null) {
			builder.getRawBeanDefinition().setBeanClass(beanClass);
		}
		else {
			String beanClassName = getBeanClassName(element);
			if (beanClassName != null) {
				builder.getRawBeanDefinition().setBeanClassName(beanClassName);
			}
		}
		builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
		BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
		if (containingBd != null) {
			// Inner bean definition must receive same scope as containing bean.
			builder.setScope(containingBd.getScope());
		}
		if (parserContext.isDefaultLazyInit()) {
			// Default-lazy-init applies to custom bean definitions as well.
			builder.setLazyInit(true);
		}
		doParse(element, parserContext, builder); //再添加一些必要的属性
		return builder.getBeanDefinition();//构建bd
	}

        {%endcodeblock%}
##### xml 和 注解实现的一些区别
- 单纯使用xml的话,gbd创建会包含所有的信息,如pvs,各种ioc属性
- 注解的话很多都要进行后处理器来处理
  - name一个String类型
      ```xml
                <property name="name">
                  zhangsan
                <property>
              ```
  - 注解实现
              ```java
              @Value("张三")
              private String name;
              ```
  - 前者会在为gbd#pvs,后者必须经过AutowiredAnnotationBeanPostProcessor(@AutoWire,@Value,@Inject)处理     

#### BeanDefinition
{%asset_img RootBeanDetinition.png RootBeanDetinition%}
spring常见的接口设计模式:
- RootBeanDetinition实质是一个AttributeAccessor和BeanDefinition
- 属于AttributeAccessor的部分由父类BeanMetadataAttributeAccessor实现
- AttributeAccessor:该接口定义如何访问源信息中的属性,在Bd体系中,通过 'BeanMetadataAttribut',作为meta 原信息体现key-value
    - AttributeAccessor
      {%codeblock lang:java AttributeAccessor%}
void setAttribute(String name, @Nullable Object value);
    Object getAttribute(String name);
    Object removeAttribute(String name);
    boolean hasAttribute(String name);
    String[] attributeNames();
      {%endcodeblock%}
    - AttributeAccessorSupport
      {%codeblock lang:java AttributeAccessorSupport%}
      //该map就是属性储存的位置,在该体系中,此map表示的是bean的一些附加信息,跟属性无关
      private final Map<String, Object> attributes = new LinkedHashMap<>();
    @Override
    public void setAttribute(String name, @Nullable Object value) {
        Assert.notNull(name, "Name must not be null");
        if (value != null) {
            this.attributes.put(name, value);
        }:
        else {
            removeAttribute(name);
        }
    }
      {%endcodeblock%}
    - BeanMetadataAttributeAccessor
      {%asset_img BeanMetadataAttributeAccessor.png%}
      {%codeblock lang:java BeanMetadataAttributeAccessor%}
      //包含了要进行配置的对象
      public interface BeanMetadataElement {

    /**
     * Return the configuration source {@code Object} for this metadata element
     * (may be {@code null}).
     */
    @Nullable
    Object getSource();

}

* Holder for a key-value style attribute that is part of a bean definition.
* Keeps track of the definition source in addition to the key-value pair.
*
* @author Juergen Hoeller
* @since 2.5
*/
//包含了source中一个属性的键值对,可以理解为pair
public class BeanMetadataAttribute implements BeanMetadataElement {

 private final String name;

 @Nullable
 private final Object value;

 @Nullable
 private Object source;
}
//
/**
 * Extension of {@link org.springframework.core.AttributeAccessorSupport},
 * holding attributes as {@link BeanMetadataAttribute} objects in order
 * to keep track of the definition source.
 * 该类用来访问source中属性,属性由BeanMetaDataAttribute包装
 * @author Juergen Hoeller
 * @since 2.5
 */
@SuppressWarnings("serial")
public class BeanMetadataAttributeAccessor extends AttributeAccessorSupport implements BeanMetadataElement {
  @Nullable
      private Object source;


      /**
       * Set the configuration source {@code Object} for this metadata element.
       * <p>The exact type of the object will depend on the configuration mechanism used.
       */
      public void setSource(@Nullable Object source) {
          this.source = source;
      }

      @Override
      @Nullable
      public Object getSource() {
          return this.source;
      }


      /**
       * Add the given BeanMetadataAttribute to this accessor's set of attributes.
       * @param attribute the BeanMetadataAttribute object to register
       */
      public void addMetadataAttribute(BeanMetadataAttribute attribute) {
          super.setAttribute(attribute.getName(), attribute);
      }
}
      {%endcodeblock%}
      {%asset_img ScannedGenericBeanDefinition.png%}
      此时的source用来说明bd的来源,该类型会在注册到`BeanFactory`时被转换为`RootBeanDefintion`类型
      图中propertyValues,是xml处理<propery>时添加的字面属性,在ioc过程中pvs要经过类型转换,以及各种处理器.
- BeanDefinition:该接口实质是描述了springBean配置
- 常见子类ScannedGenericBeanDefinition和GenericBeanDefinition
    {%codeblock lang:java BeanDefinition%}
void setScope(@Nullable String scope);
void setLazyInit(boolean lazyInit);
void setDependsOn(@Nullable String... dependsOn);
//...
    {%endcodeblock%}
#### BeanWrapper
{%asset_img BeanWrapper.png BeanWrapper%}
- PropertyEditorRegistry 和 TypeConverter组成属性转换器
##### 属性类型转换
{%asset_img Property.png PropertyEditorRegistry%}
- PropertyEditor是jdk提供的接口,用来进行属性类型转换的
    {%codeblock lang:java PropertyEditor%}
    public interface PropertyEditor {
        void setValue(Object value);
        //将转递的stirng类型进行转换
        void setAsText(String text) throws java.lang.IllegalArgumentException;
    }  
    //一般继承该接口
    public class PropertyEditorSupport implements PropertyEditor {
      private Object value;
      private Object source;
      private java.util.Vector<PropertyChangeListener> listeners;

      public PropertyEditorSupport() {
      setSource(this);
  }

  public PropertyEditorSupport(Object source) {
      if (source == null) {
         throw new NullPointerException();
      }
      setSource(source);
  }
    }
//spirng中的一个实现
public class ByteArrayPropertyEditor extends PropertyEditorSupport {

    @Override
    public void setAsText(@Nullable String text) {
        setValue(text != null ? text.getBytes() : null);
    }

    @Override
    public String getAsText() {
        byte[] value = (byte[]) getValue();
        return (value != null ? new String(value) : "");
    }

}    
    {%endcodeblock%}
  - PropertyEditor使用

      ```java
      //1.当类型String->Type转换
      //调用setText()
      //2.当其他类型->Type
      //调用setValue()
      ```

- PropertyEditorRegistrySupport使用Map存储PropertyEditor
    {%codeblock lang:java PropertyEditorRegistrySupport%}
    public class PropertyEditorRegistrySupport implements PropertyEditorRegistry {
  //spring 实现的类型转换器,先略过    
    @Nullable
    private ConversionService conversionService;
  //默认的类型转换器
  private Map<Class<?>, PropertyEditor> defaultEditors;

  private void createDefaultEditors() {
    this.defaultEditors.put(Charset.class, new CharsetEditor());
        this.defaultEditors.put(Class.class, new ClassEditor());
        this.defaultEditors.put(Class[].class, new ClassArrayEditor());
    //....添加了大量的类型转换器
  }
    {%endcodeblock%}
- TypeConverterDelegate作为委托模式用来真正处理类型转换
    {%codeblock lang:java TypeConverterSupport%}
  public abstract class TypeConverterSupport extends PropertyEditorRegistrySupport implements TypeConverter {

    @Nullable
    TypeConverterDelegate typeConverterDelegate;
  public <T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType,
            @Nullable TypeDescriptor typeDescriptor) throws TypeMismatchException {
            return this.typeConverterDelegate.convertIfNecessary(null, null, value, requiredType,
    }
}
//委托逻辑
class TypeConverterDelegate {
  /**
  *  propertyName 表示属性名
  *  oldValue 表示旧值,若该函数是pvs设值时调用一般是null
  *  newValue 表示此次要设置的值,也是要处理进行类型 转换的值
  */
public <T> T convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue, @Nullable Object newValue,
@Nullable Class<T> requiredType, @Nullable TypeDescriptor typeDescriptor) throws IllegalArgumentException {

// Custom editor for this type?
//获取自定义的属性编辑器
PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);

ConversionFailedException conversionAttemptEx = null;

// No custom editor but custom ConversionService specified?
// 尝试使用ConversionService进行转换
ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
  try {
    return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
  }
  catch (ConversionFailedException ex) {
    // fallback to default conversion logic below
    conversionAttemptEx = ex;
  }
}
}

Object convertedValue = newValue;

// Value not of required type?
if (editor != null || (requiredType != null && !ClassUtils.isAssignableValue(requiredType, convertedValue))) { //没有自定义编辑器或者 不能直接分配(即newValue需要进行向requiredType的转换)

if (typeDescriptor != null && requiredType != null && Collection.class.isAssignableFrom(requiredType) &&
    convertedValue instanceof String) {
  //此处代码逻辑 是 当属性为Collection<Class>|Collection<Eunm> 时,可以将该字符串当作CSV转化    
  TypeDescriptor elementTypeDesc = typeDescriptor.getElementTypeDescriptor();
  if (elementTypeDesc != null) {
    Class<?> elementType = elementTypeDesc.getType();
    if (Class.class == elementType || Enum.class.isAssignableFrom(elementType)) {
      convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
    }
  }
}
if (editor == null) {
  // 根据requiredType从map中获取对应的propertyEditor
  editor = findDefaultEditor(requiredType);
}
//进行转换
convertedValue = doConvertValue(oldValue, convertedValue, requiredType, editor);
}

boolean standardConversion = false;
}
//....下边的逻辑是对于不能直接适配的情况,进一步的标准处理
{%endcodeblock%}
{%asset_img doConvertValue.png doConvertValue%}
{%codeblock lang:java doConvertValue%}
//实际转换逻辑
private Object doConvertValue(@Nullable Object oldValue, @Nullable Object newValue,
            @Nullable Class<?> requiredType, @Nullable PropertyEditor editor) {

        Object convertedValue = newValue;

        if (editor != null && !(convertedValue instanceof String)) {
      //一般来说spring基本都是将string类型进行转换,但是如果此处的value并不是String,
      //那么就要调用PropertyEditor的setValue进行转换,然后再get出来,这个set逻辑要
      //用户自己实现,或者spring内部提供了对应的转换器
            try {
                editor.setValue(convertedValue);
                Object newConvertedValue = editor.getValue();
                if (newConvertedValue != convertedValue) {
                    convertedValue = newConvertedValue;
                    // Reset PropertyEditor: It already did a proper conversion.
                    // Don't use it again for a setAsText call.
                    editor = null;
                }
            }
            catch (Exception ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug("PropertyEditor [" + editor.getClass().getName() + "] does not support setValue call", ex);
                }
                // Swallow and proceed.
            }
        }

        Object returnValue = convertedValue;

        if (requiredType != null && !requiredType.isArray() && convertedValue instanceof String[]) {
          //若转换类型不为数组,并且给定的value为String数组,则将value转为成CVS
            if (logger.isTraceEnabled()) {
                logger.trace("Converting String array to comma-delimited String [" + convertedValue + "]");
            }
            convertedValue = StringUtils.arrayToCommaDelimitedString((String[]) convertedValue);
        }

    //若经过上述两操作,cV还是String,那么就可以执行Text类型转换
        if (convertedValue instanceof String) {
            if (editor != null) {
                // Use PropertyEditor's setAsText in case of a String value.
                if (logger.isTraceEnabled()) {
                    logger.trace("Converting String to [" + requiredType + "] using property editor [" + editor + "]");
                }
                String newTextValue = (String) convertedValue;
                return doConvertTextValue(oldValue, newTextValue, editor);
            }
            else if (String.class == requiredType) {
                returnValue = convertedValue;
            }
        }

        return returnValue;
    }

  private Object doConvertTextValue(@Nullable Object oldValue, String newTextValue, PropertyEditor editor) {
        try {
            editor.setValue(oldValue);
        }
        catch (Exception ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("PropertyEditor [" + editor.getClass().getName() + "] does not support setValue call", ex);
            }
            // Swallow and proceed.
        }
        editor.setAsText(newTextValue);
        return editor.getValue();
    }
}
      {%endcodeblock%}
- 例子
```java
public class Foo {

    private Sub sub;
    private List list;

    @Value("${test}")
    private String name;  //不需要转换,${test}从ev中获取
    @Value("D:/1.txt")    
    private File file;    //通过FileEditor 转换
    @Value("1,2,3,4")
    private String[] test2; //通过StringArrayEditor 直接转换
    @Value("I:/BaiduNetdisk/api-ms-win-core-console-l1-1-0.dll,I:/BaiduNetdisk/api-ms-win-core-datetime-l1-1-0.dll")
    private List<File> files; //先通过CustomCollectionEditor ,再通过FileEditor转换
    //但是默认情况下,这个转换器并不能正确处理这种类型
    //
    //private File[] files   //没有该类型转换器
```
 这里描述一下非基础类型数组|集合转换逻辑
  convertIfNecessary->判断能否进行CVS处理,如果不行则执行CustomCollectionEditor转换,结果是一个
 单个 元素list,后续逻辑中 进行遍历,再调用convertIfNecessary,那么结果一定是错的.
 - 添加自定义编辑器
 通过CustomEditorConfigurer实现,这是一个BF_post
### spring_post处理器
- BeanPostProcessor
```java
public interface BeanPostProcessor {

  default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
  default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```
- MergedBeanDefinitionPostProcessor
```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

    void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);

    default void resetBeanDefinition(String beanName) {
    }

}
```
- InstantiationAwareBeanPostProcessor
```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {
  default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }
  default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }
  default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
            throws BeansException {

        return null;
    }
}
```
- Instantiation表示实例化,即ioc反射创建BeanWrap;Initialization表示初始化,指ioc对于bean的init逻辑,如init-method
- SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference发生在早期引用获取过程
- 具体执行顺序参考[生命周期](/2019/05/21/ioc/#关于单例bean的生命周期和循环依赖问题)
### LTW
载入时织入,使用方式参考[LTW](https://www.w3xue.com/exp/article/20189/1890.html),
- 由`LoadTimeWeaverBeanDefinitionParser`创建`AspectJWeavingEnabler`以及`DefaultContextLoadTimeWeaver`
### MessageSource
该接口是spring提供的和i18n相关的基础类,使用起来并不复杂
- ResourceBulde:jdk实现的国际化类,也是messageSource的内部原理
  - 准备多种语言properties文件
    {%asset_img 多语言.png%}
    error.properties
    ```xml
    0000=error
    0001=info
    ```
    error_zh.Properties
    ```xml
    0000=错误
    test=this is a {0}
    ```
  - 使用
    ```java
    //根据Local获取不同的ResourceBulde
    ResourceBundle resourceBundle = ResourceBundle.getBundle("error",Locale.getDefault());
    String zh_test = resourceBundle.getString("test")
    //this is a {0}
    ```

- MessageFormat:jdk提供用来替换占位符的工具类
  - 接上文
    ```java
     MessageFormat messageFormat = new MessageFormat(zh_test);
     messageFormat.format(new String[]{"测试"})
     //静态调用
     MessageFormat.format("this is a {0}","测试")
     //结果都是 this is a 测试
    ```
- MessageSource
  - 定义
    ```java
    /**
    * code 表示要转换的key,如上文的0000,test
    * args 表示可以替换的占位符,如MessageFormat.format("this is a {0}",new String[]{"测试"})第二个参数
    * defaultMessage 表示未能匹配时返回的默认值
    */
    String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale);
    //无默认,不匹配则抛出异常
    String getMessage(String code, @Nullable Object[] args, Locale locale) throws NoSuchMessageException;
    //MessageSourceResolvable仅仅是封装了第一个函数的前三个参数
    String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException;
    ```
  - 使用
    ```java
          //ResourceBundleMessageSource是spring提供的实现之一
          ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
          //设置properties
          messageSource.setBasename("error");
          messageSource.setDefaultEncoding("UTF-8");
          String message = messageSource.getMessage("0001", null, Locale.ENGLISH);
          System.out.println(message);
          //error
          String str1 = messageSource.getMessage("test",new String[]{"测试"},Locale.CHINESE);
          System.out.println(str1);
          //this is a 测试
    ```
### spring框架中提供的特殊调用点
#### spring web关于web容器的初始化
- web相关:参考[dispatcherServlet](/2019/02/19/SpringMVC源码/#dispatcherServlet)
  - ContextLoaderListener:初始化MVC中ioc容器
  - SpringServletContainerInitializer:ServletContainerInitializer框架
  - WebApplicationInitializer:web初始化实际调用接口
- MVC相关
  - WebMvcConfigurer:MVC内部的设置
- spring context
  - ApplicationContextInitializer:该接口被context#refresh的调用|或context创建者调用
  ,就是整体项目的入口,如SpringApplication | FrameworkServlet
  ,就是整体项目的入口,如SpringApplication | FrameworkServlet
### springIo
#### resource接口
{%asset_img resource.png%}
{%codeblock lang:java Resource%}
public interface Resource extends InputStreamSource {
  boolean exists();
  default boolean isReadable() {
        return exists();
    }
  default boolean isOpen() {
        return false;
    }
  default boolean isFile() {
        return false;
    }
  URL getURL() throws IOException;
  URI getURI() throws IOException;
  File getFile() throws IOException;
  default ReadableByteChannel readableChannel() throws IOException {
        return Channels.newChannel(getInputStream());
    }
  long contentLength() throws IOException;
  long lastModified() throws IOException;
  Resource createRelative(String relativePath) throws IOException;
  String getFilename();
  String getDescription();
}  
{%endcodeblock%}
##### ResolvingResource
这种资源表示从url中获取,支持http协议,file,jboss的vfs协议
```java
public boolean exists() {
        try {
            URL url = getURL();
            if (ResourceUtils.isFileURL(url)) { //file: vfs协议
                // Proceed with file system resolution
                return getFile().exists();
            }
            else { //http协议
                // Try a URL connection content-length header
                URLConnection con = url.openConnection();
                customizeConnection(con);
                HttpURLConnection httpCon =
                        (con instanceof HttpURLConnection ? (HttpURLConnection) con : null);
                if (httpCon != null) {
                    int code = httpCon.getResponseCode();
                    if (code == HttpURLConnection.HTTP_OK) {
                        return true;
                    }
                    else if (code == HttpURLConnection.HTTP_NOT_FOUND) {
                        return false;
                    }
                }
                if (con.getContentLength() >= 0) {
                    return true;
                }
                if (httpCon != null) {
                    // No HTTP OK status, and no content-length header: give up
                    httpCon.disconnect();
                    return false;
                }
                else {
                    // Fall back to stream existence: can we open the stream?
                    getInputStream().close();
                    return true;
                }
            }
        }
        catch (IOException ex) {
            return false;
        }
    }
```
- ClassPathResource
```java
public InputStream getInputStream() throws IOException {
        InputStream is;
        if (this.clazz != null) {
            is = this.clazz.getResourceAsStream(this.path);
        }
        else if (this.classLoader != null) {
            is = this.classLoader.getResourceAsStream(this.path);
        }
        else {
            is = ClassLoader.getSystemResourceAsStream(this.path);
        }
        if (is == null) {
            throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
        }
        return is;
    }
```
- PathResource和FileSystemResource实现,都是依托了nio实现
##### HttpResource
这里Resource使用在`ResourceHttpRequestHandler`类中处理web请求静态资源的时候用来附件响应头
##### ContextResouce
表示从封闭的`context`中获取资源,典型的就是`ServletContext`
```java
public interface ContextResource extends Resource {

    /**
     * Return the path within the enclosing 'context'.
     * <p>This is typically path relative to a context-specific root directory,
     * e.g. a ServletContext root or a PortletContext root.
     */
    String getPathWithinContext();

}
```
###### ServletCOntextResouce
直接从`ServletContext`中获取资源
```java
public InputStream getInputStream() throws IOException {
        InputStream is = this.servletContext.getResourceAsStream(this.path);
        if (is == null) {
            throw new FileNotFoundException("Could not open " + getDescription());
        }
        return is;
    }
```
#### ResouceLoader
用来加载Resource资源的support
```java
Resource getResource(String location);
ClassLoader getClassLoader();
```
- 支持全路径:`file:C:/test.dat`
- 支持类路径:`classpath:test.dat`
- 支持相对路径:`WEB-INF/test.dat`
{%asset_img resourceLoader.png%}
实现中把context排除,那是包装类实现

##### DefaultResourceLoader
```java
public Resource getResource(String location) {
        Assert.notNull(location, "Location must not be null");

        for (ProtocolResolver protocolResolver : this.protocolResolvers) { //这是一个spi实现,spring默认没有实现
            Resource resource = protocolResolver.resolve(location, this);
            if (resource != null) {
                return resource;
            }
        }

    //[1]ClassPathResource
        if (location.startsWith("/")) {
            return getResourceByPath(location); //这个函数交由子类实现
        }
        else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
            return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
        }
    //[1!]
    //[2]UrlResource
        else {
            try {
                // Try to parse the location as a URL...
                URL url = new URL(location);
                return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            }
            catch (MalformedURLException ex) {
                // No URL -> resolve as resource path.
                return getResourceByPath(location);
            }
        }
    //[2!]
    }
  //当资源以/开头时就会调用子类实现
  protected Resource getResourceByPath(String path) {
        return new ClassPathContextResource(path, getClassLoader());
    }
  //
  protected static class ClassPathContextResource extends ClassPathResource implements ContextResource {

        public ClassPathContextResource(String path, @Nullable ClassLoader classLoader) {
            super(path, classLoader);
        }

        @Override
        public String getPathWithinContext() {
            return getPath();
        }

        @Override
        public Resource createRelative(String relativePath) {
            String pathToUse = StringUtils.applyRelativePath(getPath(), relativePath);
            return new ClassPathContextResource(pathToUse, getClassLoader());
        }
    }
```
- 总结
  - 若使用url则返回`UrlResource`
  - 若使用"classpath:"则返回`ClassPathResource`
  - 若使用"/"开头则调用`getResourceByPath`,由具体实现决定
  - 该体系内部都是实现了一个`ContextResouce`的`Resource`
###### ClassRelativeResourceLoader
- 代码逻辑
```java
private final Class<?> clazz;
    /**
     * Create a new ClassRelativeResourceLoader for the given class.
     * @param clazz the class to load resources through
     */
    public ClassRelativeResourceLoader(Class<?> clazz) {
        Assert.notNull(clazz, "Class must not be null");
        this.clazz = clazz;
        setClassLoader(clazz.getClassLoader());
    }

    @Override
    protected Resource getResourceByPath(String path) {
        return new ClassRelativeContextResource(path, this.clazz); //实际返回一个内部类
    }


    /**
     * ClassPathResource that explicitly expresses a context-relative path
     * through implementing the ContextResource interface.
     */
    private static class ClassRelativeContextResource extends ClassPathResource implements ContextResource {

        private final Class<?> clazz;

        public ClassRelativeContextResource(String path, Class<?> clazz) {
            super(path, clazz);
            this.clazz = clazz;
        }

        @Override
        public String getPathWithinContext() {
            return getPath();
        }

        @Override
        public Resource createRelative(String relativePath) {
            String pathToUse = StringUtils.applyRelativePath(getPath(), relativePath);
            return new ClassRelativeContextResource(pathToUse, this.clazz);
        }
    }

```
- 总结:开头为"/"返回classpath下文件
###### FileSystemResourceLoader
- 代码逻辑
```java
public class FileSystemResourceLoader extends DefaultResourceLoader {

    @Override
    protected Resource getResourceByPath(String path) {
        if (path.startsWith("/")) {
            path = path.substring(1);
        }
        return new FileSystemContextResource(path);
    }


    private static class FileSystemContextResource extends FileSystemResource implements ContextResource {

        public FileSystemContextResource(String path) {
            super(path);
        }

        @Override
        public String getPathWithinContext() {
            return getPath();
        }
    }

}
```
- 总结:开头为"/"返回文件系统
##### ResourcePatternResolver
支持ant风格进行获取资源
```java
public interface ResourcePatternResolver extends ResourceLoader {

    /**
     * Pseudo URL prefix for all matching resources from the class path: "classpath*:"
     * This differs from ResourceLoader's classpath URL prefix in that it
     * retrieves all matching resources for a given name (e.g. "/beans.xml"),
     * for example in the root of all deployed JAR files.
     * @see org.springframework.core.io.ResourceLoader#CLASSPATH_URL_PREFIX
     */
    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    /**
     * Resolve the given location pattern into Resource objects.
     * <p>Overlapping resource entries that point to the same physical
     * resource should be avoided, as far as possible. The result should
     * have set semantics.
     * @param locationPattern the location pattern to resolve
     * @return the corresponding Resource objects
     * @throws IOException in case of I/O errors
     */
    Resource[] getResources(String locationPattern) throws IOException;

}

```
###### PathMatchingResourcePatternResolver
- 概述
  - 支持 占位符,即:'classpath:',或者ant风格,'*',`**`,`?`
  - 支持协议"file","jar"
  - "classpath*:",表示从所有类路径下寻找
- 代码逻辑  
```java
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {
public PathMatchingResourcePatternResolver() {
  this.resourceLoader = new DefaultResourceLoader();
}

/**
 * Create a new PathMatchingResourcePatternResolver.
 * <p>ClassLoader access will happen via the thread context class loader.
 * @param resourceLoader the ResourceLoader to load root directories and
 * actual resources with
 */
public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
  Assert.notNull(resourceLoader, "ResourceLoader must not be null");
  this.resourceLoader = resourceLoader;
}

/**
 * Create a new PathMatchingResourcePatternResolver with a DefaultResourceLoader.
 * @param classLoader the ClassLoader to load classpath resources with,
 * or {@code null} for using the thread context class loader
 * at the time of actual resource access
 * @see org.springframework.core.io.DefaultResourceLoader
 */
public PathMatchingResourcePatternResolver(@Nullable ClassLoader classLoader) {
  this.resourceLoader = new DefaultResourceLoader(classLoader);
}
//包装类实现
public Resource getResource(String location) {
  return getResourceLoader().getResource(location);
}
//path匹配实现
public Resource[] getResources(String locationPattern) throws IOException {
        Assert.notNull(locationPattern, "Location pattern must not be null");
        if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) { //匹配classpath*:,即搜索全类路径
            // a class path resource (multiple resources for same name possible)
            if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) { //并非是ant路径即,后边路径中不包含*和?
                // a class path resource pattern
                return findPathMatchingResources(locationPattern);
            }
            else {
                // all class path resources with the given name
                return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));//[2]
            }
        }
        else {
            // Generally only look for a pattern after a prefix here,
            // and on Tomcat only after the "*/" separator for its "war:" protocol.
      //若无前缀则prefixEnd=0
            int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
                    locationPattern.indexOf(':') + 1);
            if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) { //不带classpath*,带ant,带协议头[3]
                // a file pattern
                return findPathMatchingResources(locationPattern);
            }
            else { //最简单的类型,如com/light/test.class,会通过当前类加载进行加载
                // a single resource with the given name
                return new Resource[] {getResourceLoader().getResource(locationPattern)};//不带classpath*,不带ant,带协议头[4]
            }
        }
    }
}
```
- 总结
```java
public void antResource() {
    PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
    try {
        //[4]
        printAllName(resolver.getResources("aop1.xml"));
        printAllName(resolver.getResources("classpath:aop1.xml"));
        printAllName(resolver.getResources("file:aop1.xml"));
        printAllName(resolver.getResources("java/lang/String.class"));
        printAllName(resolver.getResources("com/google/protobuf/AbstractMessage.class"));
        //[3]
//            printAllName(resolver.getResources("com/google/protobuf/*.class"));
        //[2]
//            printAllName(resolver.getResources("classpath*:com/**"));
        //[1]
        printAllName(resolver.getResources("classpath*:com/jcf/Test1.class"));

    } catch (IOException e) {
        e.printStackTrace();
    }
}

private void printAllName(Resource... resources) {
    for (Resource resource : resources) {
        try {
            System.out.println(resource.getURL().toString());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    System.out.println("=====================================");
}
```
输出如下
```
file:/F:/java/example/app/springTest/target/test-classes/aop1.xml
=====================================
file:/F:/java/example/app/springTest/target/test-classes/aop1.xml
=====================================
file:aop1.xml
=====================================
jrt:/java.base/java/lang/String.class
=====================================
jar:file:/C:/Users/fancylight/.m2/repository/com/google/protobuf/protobuf-java/2.6.0/protobuf-java-2.6.0.jar!/com/google/protobuf/AbstractMessage.class
=====================================
file:/F:/java/example/app/jcf/target/classes/com/jcf/Test1.class
=====================================

Process finished with exit code 0

```
- 实际上`xx:`,这部分可以是协议
- 如果实际是从类路径中获取数据,那么ide中运行,可能会出现`file`(target文件夹),'jar'(maven仓库),`jrt`(jdk包)
