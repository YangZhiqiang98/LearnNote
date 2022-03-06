

### 拦截器与过滤器的区别

- `过滤器` 是基于函数回调的，`拦截器` 则是基于Java的反射机制（动态代理）实现的
- Interceptor 不依赖于 servlet 容器，而 Filter 依赖于 servlet 容器。

### 执行时机

过滤前-拦截前-action执行-拦截后-过滤后

![image-20210815095510597](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210815095510597.png)

### 图解

![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20190620230043280.png)







### 主要参考

[1] [拦截器与过滤器的区别_Tang-CSDN博客_拦截器与过滤器的区别](https://blog.csdn.net/weixin_44502804/article/details/93139550)

[2] [拦截器（Interceptor）和过滤器（Filter）的执行顺序和区别_scorpio的博客-CSDN博客_过滤器和拦截器的区别](https://blog.csdn.net/zxd1435513775/article/details/80556034)

[3] [Spring 拦截器和过滤器的区别?](https://www.zhihu.com/question/30212464/answer/1786967139)

[4] [过滤器 和 拦截器的 6个区别，别再傻傻分不清了_程序员内点事-CSDN博客_过滤器和拦截器](https://blog.csdn.net/xinzhifu1/article/details/106356958)
