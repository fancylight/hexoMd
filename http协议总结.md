---
title: http协议总结
cover: /img/http.png
top_img: /img/post.jpg
date: 2020-03-31 20:48:40
tags:  
- 协议
categories:
- 网络
description: 描述http一些协议用法和例子
---
### http方法
一般来说http协议ref文档中定义的都是通过服务器来实现,但并不是说存在类似编译器的东西来强制规定,都仅仅是语义定义.
- 定义
  - `get`: 用来访问资源,并且将参数放在url中
  - `post`: 一般用来发送表单,资源,数据放在请求体
  - `put`: 更新资源
  - `options`: 用来查看服务器支持的方法
  - `tarce`: 排序代理影响,返回请求状态
  - `delete`: 删除资源
  - `head`: 和`get`相似,但是仅仅返回头部
- 幂等性
简单来说幂等性指的是多次访问对于资源的影响等同于一次请求,`ref2616`如下定义:
    `Methods can also have the property of "idempotence" in that (asidefrom error or expiration issues) the side-effects of N > 0 identicalrequests is the same as for a single request`    
  - `get`,`put`,`head`,`delete`,`options`,`tarce`应该被设计为幂等性
  - `post`不是幂等性操作,它和`put`能够完成同样的功能,但是后者要被设计成幂等性操作.
### Range相关
作用用于客户端向服务器分段获取资源,典型的使用为h5播放器
- Range
参考[rfc7233](https://tools.ietf.org/html/rfc7233#section-2.3)
  - 请求头
      作为req请求头,表示向服务器请求范围资源,语法如下:
    ```
    Range: <unit>=<range-start>-
    Range: <unit>=<range-start>-<range-end>
    Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>
    Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>, <range-start>-<range-end>
    ```
      例如:`Range: bytes=200-1000, 2000-6576, 19000-`
  - 相应头    
    ```
      Accept-Range:bytes //服务器告诉浏览器自身支持partial requests
      Content-Range: bytes 200-1000/67589 //表示字节 范围/资源总长度
    ```
- 实现参考
  springMvc中[回参消息处理器](/2020/03/23/springMVC核心二/#抽象逻辑-v2)以及
### http缓存机制
参考[mdn](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ)
[博客](https://blog.techbridge.cc/2017/06/17/cache-introduction/)
#### 新鲜度
浏览器第一次访问服务器资源时,服务器会附带标签告知浏览器将资源缓存多久,在未过期前浏览器不会发出请求
- Expires(1.0协议)
`Expires: Wed, 21 Oct 2017 07:28:00 GMT`
- Cache-Control
`Cache-Control: max-age=30`
#### 服务器是否更新
##### Modified
浏览器访问后,服务器相应附带资源的修改时间,下次浏览器访问会带上该标签,若服务发现资源未修改过,那么请求直接返回,并返回`304`状态码
- Last-Modified:表示上次修改时间,响应头
`Last-Modified: 2017-01-01 13:00:00`
- If-Modified-Since:浏览器发送表示询问是否发生改变
`If-Modified-Since: 2017-01-01 13:00:00`
##### Etag
这个功能和上边一致,是为了处理服务器并未修改文件,但是如打开文件修改,撤销,保存操作,那么修改日期也会变,etag类似于md5.
- Etag:响应头
`Etag:x246dfff`
- If-None-Match:请求头,询问
 `If-None-Match:x246dfff`
#### Cache-Control
实际上这个响应头能够完成的事情比较多
- no-store:表示强制每次都获取
- no-cache:表示浏览器获取最新
  - 等价于:`cache-contorl: max-age=0 etag=xxx`,可以理解为每次都发送请求,但是根据tag来获取最新的改变
- private:表示只能缓存在浏览器私人缓存中
- public:可以被代理,cdn缓存
#### Pragma
1.0定义的一个头,等价于`Cache-Control:no-cache`,用来兼容1.0  
#### Vary
参考[vary](https://blog.csdn.net/qq_29405933/article/details/84315254)
该响应头表示这个数据对于客户端来说可能会其变化,一般要和缓存服务一起使用
```
Vary: *
Vary: <header-name>, <header-name>, ...
```
{%asset_img vary.png%}
注意多次访问实际上都是同一个url,也就是同一个接口返回针对不同的客户端不同的数据,通过`请求头`来判断客户端

- 当第一次访问,无缓存直接访问服务器,服务器设置`Vary:Context-Encoding`,缓存数据库缓存数据和响应头,并返回数据
- 第二次访问,缓存服务器根据`Context-Endcoding`来判断,还是没有命中,继续访问服务器并缓存
- 第三次访问缓存服务器命中
