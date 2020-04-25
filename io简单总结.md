---
title: io简单总结
date: 2020-03-16 07:59:02
tags:
- jdk
categories: java
cover: /img/java.png
top_img: /img/post.jpg
---
### 概述
本文用来简单分类java io,不做源码分析
整体按照类图来分类,实际上io应当按照基础和包装类分类,基本上java io中基础流实现功能,
包装流封装基础流完成进一步功能.
#### inStream
{%asset_img in总结.png%}
jdk中in的意思表示指定来源,read函数表示从来源中按字节读取
- 直接子类
  - FileInputStream:文件读取
  - ByteArrayInputStream:从指定数组源中读取
  - PipedInputStream:和PipedOutputStream联合使用,实现原理通过共享一个数组,多线程间使用
  - SequenceInputStream:指定一组InputStream,当读取到当前流末尾则触发下一个
  - ObjectInputStream:序列化读取流
- Filter子类:典型的包装类
  - BufferedInputStream:内部含有byte[],读取时,若数组未满,触发内部真正流的读取填充内部数组
否则直接获取数组数据
  - DataInpuStream:DataInput子类,ReadBoolen等函数按对应字节读取,然后进行类型转换
#### outStream
{%asset_img out总结.png%}
- flsuh函数
  - out流中的定义是调用时保证将存在缓存的情况真正刷新到目的地,如FileOutputStream这种
实现本身的write函数就不存在缓存,flush函数实际是空白的.
  - BufferedOutputStream的刷新时机在于write时内部buff已满,或主动调用flush
- PrintStream:System.out,内部封装了Writer,因此其print函数是调用Writer中的
#### reader
{%asset_img reader总结.png%}
- reader 实际上是对于字节的转换InputStreamReader包含了解码器StreamDecoder
- BufferedReader,PipedReader都是包装类设计
#### writer
{%asset_img writer总结.png%}
#### 总结
{%asset_img 分类.png%}
