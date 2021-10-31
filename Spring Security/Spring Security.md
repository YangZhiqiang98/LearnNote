# 一、Spring Security 简介

### 1.1 核心功能

- authentication(认证)：
- authorization（授权）

### 1.2 整体架构

​	在Spring Security中，认证（authentication）和授权（Authorization）是分开的，无论使用什么样的认证方式，都不会影响授权。这种独立的好处就时Spring Security可以非常方便 地整合一些外部的认证方案。

#### 1.2.1 认证

​	在Spring Security中，用户的认证信息主要由Authentication的实现类来保存。Authentication的接口定义如下：

​	



































# 参考文章

[1] [Spring Security Reference](https://docs.spring.io/spring-security/site/docs/5.4.6/reference/html5/#authentication)

[2] https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI1NDY0MTkzNQ==&action=getalbum&album_id=1319828555819286528&scene=173&from_msgid=2247488106&from_itemidx=1&count=3&nolastread=1&uin=&key=&devicetype=Windows+10+x64&version=6302019a&lang=zh_CN&ascene=0&fontgear=3

[3] 《深入浅出spring security》

[4] 

