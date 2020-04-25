---
title: spring架构整体
date: 2020-03-11 23:18:29
tag:
- 框架
- spring
categories: java
cover: /img/spring.png
top_img: /img/post.jpg
---
### 概述
本文尝试从宏观上分析Spring Framework的组织架构
{%asset_img 4.x架构.png%}
这幅图是官网4.x给出的组织架构,[文档](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/overview.html)
#### 详细描述图
{%asset_img 模块关系.png%}
上图是根据模块间构建关系大致画的图
- 并没有表达一个模块所有的依赖,仅仅是表示了spring内部模块的构建依赖
- 为了不让图看起来太复杂,有几个模块并没有画出来
##### 分模块
###### JCL
{%asset_img JCL包.png%}
- 描述
实际上该模块通过`spi`实现了运行时决定整体日志系统,spring并没有使用`ServiceLoader`
###### CORE
{%asset_img CORE包.png%}
- 概述
  - asm包将`ASM`框架中的类直接重新打包到这里,实际上spring里cglib的实现直接调用这里接口,没有另外使用asm依赖
  - cglib包实现了cglib的一部分封装,供aop实现调用
  - lang包提供了对于jsr305的封装,以及废弃的条件编译注解
  - objenesis包提供了和jdk不同反射构造对象的能力,spring在`objenesis`框架上做了扩展
  - util包提供了花里胡哨的工具类
  - core:提供了转换器,codec,统一的io等
###### BEANS
{%asset_img BEANS包.png%}
- 概述
  - 非里层包:BeanWrap体系,PropertyValue体系
  - annotation:一个特别的注解工具类
  - facotry
    - 外部:beanfacotry定义,bean周期接口(initalizingBean...),三个基本aware接口,ObjectFactory
    - annotation:@Autowire,@Configurable,@Value,@Qualifer,以及`AGBD`,`InjectionMetadata`元信息
    - config:`BD`定义,`BF`的多种接口定义,后处理器接口,`PropertyPlaceholderConfigurer`,以及一些`FactoryBean`
    - supprot:`DLFC`定义,`RBD`定义,`Managed*`定义
    - xml:定义了xml解析器,完成xml解析的框架,提供了关于<Bean>标签处理的`parse`
  - propertyeditors:类型转换器
- 大致的知识点
{%asset_img Beans体系.png%}
###### AOP
{%asset_img AOP包.png%}
- 概述
  - aopalliance:定义了`advice`,`intercept`,`pointcut`等aop中的概念接口
  - framework:这是aop的实现核心逻辑
    - framework.adaptor:适配器,实现advisor到intercept的转换
    - autoproxy:以后处理器为逻辑,实现了spring aop的自动代理
    - 外部:由ProxyFactory为外层接口,实现了对于jdk和cglib代理的实现,这里的入口不支持`asepctj`语法
  - 外部:定义了如前置advice,后置advice,以及advisor接口
  - supprot:定义了众多的advisor或pointCut实现  
- 大致知识点
{%asset_img Aop体系.png%}  
###### INSTRUMENT
{%asset_img INSTRUMENT包.png%}
- 概述
  - InstrumentationSavingAgent:是java agent的入口类,该类实际上在spring中作为一个静态类被CONTEXT模块使用
  给需要使用`LTW`的地方提供了`Instrumentation`  
###### CONTEXT  
{%asset_img CONTEXT包.png%}
