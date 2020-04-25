---
title: maven学习二
date: 2020-04-23 21:50:59
tags:
- 构建
- 项目管理
categories: 工具
cover: /img/maven.png
top_img: /img/post.jpg
description: maven基础模型
---
# 概念模型
根据官网定义,maven分为固定三个大类的生命周期,简单理解maven构建过程就是通过`mvn`来逐步执行生命周期的过程.
lifecycle由`phase`构成,简单来说`phase`才是maven固定的执行逻辑,每个`phase`会执行绑定在其上的`goal`,而`goal`
则是由`plugin`实现的
## 生命周期lifecycle
- Clean Lifecycle
    {%asset_img clean周期.png clean周期%}
- Default Lifecycle
    {%asset_img default.png defualt周期%}
- Site LifeCycle
    {%asset_img site.png site周期%}
maven默认会在各个`phase`中绑定不同的插件,并且执行默认的`goal`,`mvn:phase`命令会执行当前生命周期中的之前的phase,
例如:`mvn install`会导致
## 插件plugin
maven本身开发并没有含有功能,只有当用户指定了plugin,由maven执行过程中调用plugin来处理对应的`phase`,因此maven编译后并不大
maven中插件实际上就是按照maven标准开发的jar,每个`plugin`包含了若干个`goal`,只有当其绑定到`phase`中时,在执行生命周期
命令时才会被调用
- mvn命令:
    - mvn phase :会触发改`pahse`所在的生命周期之前并且包含自身的`pahse`
    - mnv plugin:goal :用来调用对应插件的的`goal`,其中plugin部分表现为`gourpId:artifact:version:goal`
- 缩减调用插件goal命令
当调用maven默认的插件如`mvn clean:clean`,即调用`org.apache.maven.plugins:maven-clean-plugin: clean`可以简写,原因是maven源码
启动的时候会自动寻找groupId为`[org.apache.maven.plugins, org.codehaus.mojo]`,并且`artifact`为`maven-[pluginName]-plugin`的依赖并调用
- 自定义maven插件的命名方式
    - `artifact`一般命名为`pluginname-maven-plugin`
    - `groupId`随意,如果想要使用maven原生插件一样,则需要主动在`user/setting.xml`中添加,参考[插件命名](https://maven.apache.org/guides/plugin/guide-java-plugin-development.html#shortening-the-command-line)
        ```xml
        <pluginGroups>
          <pluginGroup>sample.plugin</pluginGroup>
        </pluginGroups>
        ```
## 常见插件
###
a's'd
