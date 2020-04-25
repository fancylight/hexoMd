---
title: maven学习
date: 2019-08-11 18:28:43
tags:
- 构建
- 项目管理
categories: 工具
cover: img/maven.png
top_img: /img/post.jpg
description: maven依赖传递
---
### 概述
关于maven的使用和概念不是本文要探讨的问题,这里解决的问题是maven依赖关系和继承传递相关问题
#### 继承和多模块
- 继承:即模块B对于模块A使用`<parent></parent>`
    - 子模块可以使用父模块的依赖
    - 子模块可以使用父模块的插件配置
    - 子模块可以使用父模块的属性
- 多模块:指模块A使用`Module`
    - 当mvn执行父模块会依次执行子模块
#### 依赖传递
||compile|provided|test|runtime|
|--|---|---|---|---|
|compile|compile|-|-|runtime|
|provided|provided|-|x|provided|
|test|test|provided|-|test|
|runtime|runtime|-|-|runtime|
- 存在两个模块A,B,横向表头表示A模块中任意依赖的`scope`,纵向表头表示B模块对于A模块的`scope`,其他值表示B模块对于A传递过来模块的`scope`情况
##### scope补充
- system:此处类型是用来引入本地的jar,如下引用工程下`lib`目录中的jar依赖
```xml
<dependency>  
    <groupId>htmlunit</groupId>  
    <artifactId>htmlunit</artifactId>  
    <version>2.21-OSGi</version>  
    <scope>system</scope>  
    <systemPath>${project.basedir}/libs/htmlunit-2.21-OSGi.jar</systemPath>  
</dependency>
```
- import:使用在type为pom的类型,用来引入其定义完善的依赖
将maven继承和java继承类比都是单继承,import可以当作java接口来理解,可以实现多继承
```xml
<dependency>
<groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-parent</artifactId>
       <version>2.1.9.RELEASE</version>
        <scope>import</scope>
</dependency>
```
#### optional和exclude
简单来说optional用来控制是否传递该依赖,`true`表示不会直接传递该依赖
1. 创建一个pom工程作为前置准备

{%codeblock lang:cmd 依赖关系%}
   A     B
   |_____|    
      |
      C			 		  
通过install命令将当前模块打包成pom,放置到本地仓库中	  
{%endcodeblock%}
{%codeblock lang:xml C.pom%}		 
	    <dependencies>
        <dependency>
            <groupId>com.light</groupId>
            <artifactId>A</artifactId>
            <version>1.0-SNAPSHOT</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.light</groupId>
            <artifactId>B</artifactId>
            <version>1.0-SNAPSHOT</version>
            <optional>true</optional>
        </dependency>
    </dependencies>		 
{%endcodeblock%}
{%codeblock lang:java 代码%}
//模块A
package com.light.a;

public class ATest {
    public static void aTest() {
        System.out.println("atest");
    }
}

//模块B
package com.light.b;

public class BTest {
    public static void bTest(){
        System.out.println("bTest");
    }
}

//模块C
import com.light.a.ATest;
import com.light.b.BTest;

public class CTest {
    public static void cTest1() {
        System.out.println("cTest use");
        ATest.aTest();
    }

    public static void cTest2() {
        System.out.println("cTest use");
        BTest.bTest();
    }
}


{%endcodeblock%}
2. 将这个工程当作第三方依赖,构建一个新的pom项目
{%codeblock lang:xml mavenTest2.pom%}
<?xml version="1.0" encoding="UTF-8"?>
<project>

    <groupId>com.light</groupId>
    <artifactId>mavenTest2</artifactId>
    <version>1.0-SNAPSHOT</version>
    <dependencies>
        <dependency>
            <groupId>com.light</groupId>
            <artifactId>C</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>com.light</groupId>
            <artifactId>A</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
</project>
{%endcodeblock%}
该工程来验证代码
{%codeblock lang:java 代码%}
public class Test {
    public static void main(String[] args) {
        CTest.cTest1();
        CTest.cTest2();
    }
}
//结果
//cTest use
//atest
//cTest use
//Exception in thread "main" java.lang.NoClassDefFoundError: BTest
//查看java执行命令

//"C:\Program Files\Java\jdk-12\bin\java.exe"   
// classpath "F:\code\mavenTest2\target\classes;
// C:\Users\Administrator\.m2\repository\com\light\C\1.0-SNAPSHOT\C-1.0-SNAPSHOT.jar;
//C:\Users\Administrator\.m2\repository\com\light\A\1.0-SNAPSHOT\A-1.0-SNAPSHOT.jar;
//...
{%endcodeblock%}
这个例子明确说明几点
- javac命令的理解:
	- 仅仅会处理目标(可以理解为当前项目)的依赖关系,只要满足直接依赖关系就能够通过编译.
	- 如该例子中mavenTest2-->C 并没有对于A,B的依赖,因此即使没有A,B的依赖依旧能够通过编译,在运行时抛出异常
	- 这里更加说明了javac编译过程对于非本类的信息仅仅是记录了符号引用,具体可以通过javap -v去研究
- maven工具本质:
	- 还是围绕javac 和java本身进行一些外围处理,该例子中mavenTest2的直接引用是C,由于使用了`optional`,因此
	- 通过maven执行的java命令是没有指定B的classPath
- 带有optional的依赖pom被其他项目引用时,是不会主动下载引用的,典型的spring中log4j的使用方式,使用optional和类加载器实现(貌似没有调用spi)
{%codeblock lang:xml spring-jcl.pom%}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.springframework</groupId>
  <artifactId>spring-jcl</artifactId>
  <version>5.1.0.RELEASE</version>
  <name>Spring Commons Logging Bridge</name>
  <description>Spring Commons Logging Bridge</description>
  <url>https://github.com/spring-projects/spring-framework</url>
  <organization>
    <name>Spring IO</name>
    <url>http://projects.spring.io/spring-framework</url>
  </organization>
  <licenses>
    <license>
      <name>Apache License, Version 2.0</name>
      <url>http://www.apache.org/licenses/LICENSE-2.0</url>
      <distribution>repo</distribution>
    </license>
  </licenses>
  <developers>
    <developer>
      <id>jhoeller</id>
      <name>Juergen Hoeller</name>
      <email>jhoeller@pivotal.io</email>
    </developer>
  </developers>
  <scm>
    <connection>scm:git:git://github.com/spring-projects/spring-framework</connection>
    <developerConnection>scm:git:git://github.com/spring-projects/spring-framework</developerConnection>
    <url>https://github.com/spring-projects/spring-framework</url>
  </scm>
  <issueManagement>
    <system>Jira</system>
    <url>https://jira.springsource.org/browse/SPR</url>
  </issueManagement>
  <dependencies>
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-api</artifactId>
      <version>2.11.1</version>
      <scope>compile</scope>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.25</version>
      <scope>compile</scope>
      <optional>true</optional>
    </dependency>
  </dependencies>
</project>
{%endcodeblock%}
这个pom是spring项目大部分pom引用的顶层,简要的说,关注`<optional>true</optional>`表示`log4j2`和`slf4j`都是不向下传递的,
对于spring构建者来说,他们完成了编译工作,至此:
1. 当我们使用maven构建spring项目时,spring中带有`<optional>true</optional>`的依赖不会被引入到项目中,如果我们不引入日志api,则在spring中不会有日志输出
2. 可以仔细观察idea的依赖,maven不会主动下载'<optional>true</optional>'这样的依赖.
具体spring如何使用spi方式(实际上不是spi api)动态确定log框架
{%codeblock lang:java LogAdapter.java%}
package org.apache.commons.logging;

import java.io.Serializable;
import java.util.logging.LogRecord;

import org.apache.logging.log4j.Level;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.spi.ExtendedLogger;
import org.apache.logging.log4j.spi.LoggerContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.spi.LocationAwareLogger;

/**
 * Spring's common JCL adapter behind {@link LogFactory} and {@link LogFactoryService}.
 * Detects the presence of Log4j 2.x / SLF4J, falling back to {@code java.util.logging}.
 *
 * @author Juergen Hoeller
 * @since 5.1
 */
final class LogAdapter {

	private static LogApi logApi = LogApi.JUL;
	//简单来说spring没有直接使用spi即ServiceLoader.java,这个东西比较简单,以后再说
	//spirng采用了直接使用 类加载器来加载可能存在的log框架
	static {
		ClassLoader cl = LogAdapter.class.getClassLoader();
		try {
			// Try Log4j 2.x API
			cl.loadClass("org.apache.logging.log4j.spi.ExtendedLogger");
			logApi = LogApi.LOG4J;
		}
		catch (ClassNotFoundException ex1) {//该异常不是运行时异常,在类加载时抛出
			try {
				// Try SLF4J 1.7 SPI
				cl.loadClass("org.slf4j.spi.LocationAwareLogger");
				logApi = LogApi.SLF4J_LAL;
			}
			catch (ClassNotFoundException ex2) {
				try {
					// Try SLF4J 1.7 API
					cl.loadClass("org.slf4j.Logger");
					logApi = LogApi.SLF4J;
				}
				catch (ClassNotFoundException ex3) {
					// Keep java.util.logging as default
				}
			}
		}
	}
{%endcodeblock%}
- <optional>的使用场景就是上述的例子
- <exclude> 就是给任意pom拒绝某个依赖的能力
