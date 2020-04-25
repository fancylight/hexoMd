---
title: spring
date: 2019-05-14 09:10:52
tags:
- 框架
- spring
- 使用
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
### spring
#### resource资源接口
{%codeblock lang:java 测试%}
//只用管三个实现 ClassPathResource FileSystemResource ServletContextResource
//构造器
public ClassPathResource(String path) {
        this(path, (ClassLoader) null);
    }


public FileSystemResource(String path) {
        Assert.notNull(path, "Path must not be null");
        this.path = StringUtils.cleanPath(path);
        this.file = new File(path);
        this.filePath = this.file.toPath();
    }


public ServletContextResource(ServletContext servletContext, String path) {
        // check ServletContext
        Assert.notNull(servletContext, "Cannot resolve ServletContextResource without ServletContext");
        this.servletContext = servletContext;

        // check path
        Assert.notNull(path, "Path is required");
        String pathToUse = StringUtils.cleanPath(path);
        if (!pathToUse.startsWith("/")) {
            pathToUse = "/" + pathToUse;
        }
        this.path = pathToUse;
    }
//分别从classpath  系统文件  web路径获取数据
{%endcodeblock%}
- ant风格,使用ResourceLoader子类才使用的
  - *匹配任意字符
  - **匹配多个路径
  - ?匹配一个字符
{%codeblock lang:java %}
    /**
     * classpath**表示匹配所有的类路径,类路径可以通过idea启动项目时查看-classpath
     *对于maven项目可以知道,父-子项目的关系,仅仅是子项目会包含父项目的dependence,如果要包含父项目要主动dependence
     * 该函数的启动命令为
     * "C:\Program Files\Java\jdk-12\bin\java.exe" 省略一部分 -classpath
     * F:\code\javaProject\app\springTest\target\test-classes;F:\code\javaProject\app\springTest\target\classes;F:\code\javaProject\app\jcf\target\classes;
     * 上边就包含了本项目和依赖的同级子项目
     */
    @Test
    public void antResource(){
        try {
            Resource resources[]= new PathMatchingResourcePatternResolver().getResources("classpath*:test.xml");
            for (Resource resource:resources)
                System.out.println(resource.exists());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
{%endcodeblock%}
|前缀|实例|对应资源|
|---|---|----|
|classpath:|classpath:test.xml|classpath路径|
|file:|file:d:/1.txt|系统文件,可以使用相对路径|
|http://|http://localhost:8080/test.xml|url|
|ftp://|..|ftp服务器|
|无前缀|com/smart/xx.xml||
#### IOC
##### BeanFactory体系
###### 工厂模式
简单来说工厂模式就是通过特定的工厂类来创建对象,bean工厂可以理解为对所有bean进行创建的类,工厂模式可见设计模式部分.
{%codeblock lang:java 使用%}
  public void beanFactoryTest(){
        var  resource= new PathMatchingResourcePatternResolver().getResource("classpath:spTest.xml");
        Assert.notNull(resource,"未找到");
        var factory=new DefaultListableBeanFactory();
        var xmlReader=new XmlBeanDefinitionReader(factory);
        xmlReader.loadBeanDefinitions(resource);
        System.out.println(factory.getBean("car",Car.class));
        System.out.println(factory.getBean("car",Car.class));
        System.out.println(factory.getBean("car",Car.class));
    }
{%endcodeblock%}
###### BeanFactory和Context的关联
- 从关系上看是包含关系
###### IOC中各种处理器
- aware接口 | BeanPostProcessor 接口 | init |des
- InstantiationAwareBeanPostProcessor 该接口实际是BeanPostProcessor的子类
BeanFactory会在添加BeanPost的时候对InstantiationAwareBeanPostProcessor和DestructionAwareBeanPostProcessor做一个标记,供创建bean的时候使用
之所以其这个名字,是因为该处理器会在bean构造前后调用
```java
//在bean实例化之前调用,若该函数返回非null,则表示该bean实例化结束,不进行后续处理
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }
//若postProcessBeforeInstantiation返回了非null,则立即调用
//若before没有创建bean,则该函数会在bean构造之后调用,返回值true表示该bean没有创建完毕
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }
//该函数会在构造,自动包装之后调用,pvs表示bean的属性,通过改变该值,可以来替换注入的值
@Nullable
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
            throws BeansException {

        return null;
    }
```
- DestructionAwareBeanPostProcessor
##### IOC特性
###### 定义
- 标签:bean:name:和id的区别是前者可以随意命名,后者要遵循规范
    ```xml
    <bean id="car" class="ioc.xml.Car"
    name="./12car" scope="prototype"
    /> //定义一个bean
    ```
###### 属性
- 标签: P 或者property
- 通过set方法注入,value表示属性名,value表示直接值,ref表示引用其他bean

    ```xml
        <!--set赋值-->
    <bean id="ref" class="ioc.xml.Ref"/>
    <bean id="car" class="ioc.xml.Car" >
        <property name="name" value="123"/>
        <property name="count" value="22"/>
        <property name="ref" ref="ref"/>
    </bean>
    ```
- 通过构造器,构造器分别可以通过参数名,参数位置,参数类型进行设值并构造对象      
    ```xml
    <bean id="car2" class="ioc.xml.Car">
        <constructor-arg name="name" value="car2"/>
        <constructor-arg index="1" value="10"/>
        <constructor-arg type="ioc.xml.Ref" ref="ref"/>
    </bean>
    ```
###### 工厂对象
- 标签: factory-method |factory-bean | FactoryBean接口
    {%codeblock lang:xml%}
        <!--工厂构建-->
        <!--非静态工厂-->
        <bean id="factory" class="ioc.xml.CarFactory" p:name="car3" p:count="12" p:ref-ref="ref"/>
        <bean id="car3" factory-bean="factory" factory-method="factoryCar"/>
        <!--静态工厂-->
        <bean id="car4" class="ioc.xml.CarFactory" factory-method="staticFactoryCar"/>
        <!--实现BeanFactory的泛型工厂-->
        <bean id="car5" class="ioc.xml.MyBeanFactory"/>
    {%endcodeblock%}
- 关于工厂模式,我们可以在要通过spring某些本身就是工厂模式的类的情况下使用
- FactoryBean接口是spring提供的接口,实现该接口的类用于创建bean,通过带有&前缀获取bean可以获取到该实例本身
###### 注值protype使用
- 字面量:即prototype中使用value,当出现特殊需要转义的字符如`<`使用`&lt`,或者使用`<![CDATA[数据]]>`来填写
- 引用:通过ref来引用其他bean,如果使用p标签则无法指定parent,用来引用父Context中的bean
  ```xml
  //pSpring.xml
  <bean id="p" class="ioc.xml.Ref"></bean>
  //xmlSpring.xml
    <bean id="car6" class="ioc.xml.Car">
        <property name="ref">
            <ref parent="p"></ref>
        </property>
    </bean>
  //可以引用父Context中的bean
  ```
- 内部bean:类似于匿名bean,这种bean不能被别人引用   

  {%codeblock lang:xml%}
  <!--内部bean-->
    <bean id="car7" class="ioc.xml.Car" p:name="car7">
        <property name="ref">
            <bean class="ioc.xml.Ref"></bean>
        </property>
    </bean>
  {%endcodeblock%}
- null值,通过<null/>标签  
    ```xml
      <!--null值-->
        <bean id="car8" class="ioc.xml.Car" p:name="car8">
            <property name="ref">
                <null></null>
            </property>
        </bean>
    ```
- 级联:即可以对bean内部某个域的属性进行赋值,前提是该域被创建,可以多级级联
    ```xml
      <!--级联-->
    <bean id="car9" class="ioc.xml.Car" p:name="car9">
        <property name="cascade.a" value="12"/>
    </bean>
    //car
    private Cascade cascade=new Cascade();
    ```
- 集合类属性
  - List和set:list属性的值可以为匿名bean,引用bean,或者字面量,Set类型类似
  - Map和properties,稍微注意一下语法就好了
  - 集合支持通过merger属性和指定了parent的bean进行属性合并
    - value-type,可以指定当前集合类型,实际不指定也没有问题,仅仅区别在于spring源码类型转换过程
  {%codeblock lang:xml%}
    <bean id="ref" class="ioc.xml.Ref"/>
    <bean id="col" class="ioc.xml.CollectionIoc">
        <!--list-->
        <property name="list">
            <list>
                <bean class="ioc.xml.Ref"></bean>
                <ref bean="ref"></ref>
            </list>
        </property>
        <!--map-->
        <property name="map">
            <map>
                <entry key="1" value-ref="ref"></entry>
                <entry>
                    <key>
                        <value>1</value>
                    </key>
                    <bean class="ioc.xml.Ref"></bean>
                </entry>
            </map>
        </property>
        <!--properties-->
        <property name="properties">
            <props>
                <prop key="1">ceshi</prop>
            </props>
        </property>
    </bean>
  {%endcodeblock%}
- 集合bean  
通过util标签可以配置集合类型的bean
      {%codeblock lang:xml%}
     <util:set id="setBean" set-class="java.util.HashSet">

        <ref bean="ref"></ref>
    </util:set>
  {%endcodeblock%}
  {%asset_img bean对象.png%}
这个图中说明的是作为属性的标签,在能够填入属性的位置就这些,再加上entery,配合spring语法很好理解
###### 自动装配
autowire=  byName byType constructor autodetect,这是xml配置的属性和注解配置是两个方式
    ```xml
    <!--自动装配-->
    <bean id="car10" class="ioc.xml.Car"  autowire="byName"></bean>
    ```
###### 方法注入
通过CGlib进行方法增强来完成注入
- lookup:用来解决单例中每次调用获取新的域
```
<bean id="ref2" class="ioc.xml.Ref" scope="prototype"/>
    <bean id="car11" class="ioc.xml.Car">
        <lookup-method name="getRef" bean="ref2"></lookup-method>
    </bean>
```  
- replaceed-method,实现spring提供的MethodReplacer接口
    ```xml
      <!--方法替换-->
        <bean id="repalce" class="ioc.xml.MyReplace"></bean>
        <bean id="car12" class="ioc.xml.Car">
            <replaced-method name="getRef" replacer="repalce"></replaced-method>
        </bean>
    ```
###### bean之间的关系
- 父子继承|依赖|引用
    ```xml
     <!--bean关系-->
    <!--继承-->
    <bean id="carP" class="ioc.xml.Car" abstract="true" p:count="10" p:ref-ref="ref"/>
    <bean id="car13" class="ioc.xml.Car" parent="carP" p:name="car13"/>
    <!--依赖-->
    <bean id="car14" class="ioc.xml.Car" depends-on="car13"/>
    ```
###### 补充
- idref:充当bean的String属性,表示其他bean的id,如果不存在该id,则getBean过程抛出异常
    ```xml
      <bean id="teacher" class="test.teacher"/>
        <bean id="student" class="test.teacher"    >
            <!--teacherId属性为String类型-->
            <property name="teacherId">
                <idRef bean="tacher">
            <property>
        </bean>

    ```
