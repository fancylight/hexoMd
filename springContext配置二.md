---
title: springContext配置二
cover: /img/spring.png
top_img: /img/post.jpg
date: 2020-07-28 23:56:54:w
tags:
- 框架
- spring
- springMvc
categories: java
description: springContext配置基础设施配置
---
本文继续描述spring-context模块相关的配置
## spring-configured
该配置用来配置`@Configurable`(spring-beans)注解,以及对应的处理器`aspect AnnotationBeanConfigurerAspect`,显而易见的该部分是要使用到`spring-aspectj`模块,通过
aspectj来处理被`@Configurable`的类,将其加入到ioc容器中,该部分我没有使用过
## mbean
这部分是和jmx相关的,典型的如tomcat启动过程将其内部各个组件都加入到了jmx管理,spring也提供了如此功能,暂时不做分析
## context模块中的关键注解