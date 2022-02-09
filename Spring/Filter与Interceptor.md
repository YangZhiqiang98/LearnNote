

### 拦截器与过滤器的区别

- 拦截器是基于 java 的反射机制，而过滤器是基于函数的回调。
- 拦截器不依赖于 servlet 容器，而过滤器依赖于 servlet 容器。
- 

### 执行时机

过滤前-拦截前-action执行-拦截后-过滤后

![image-20210815095510597](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210815095510597.png)

### 图解

![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20190620230043280.png)







### 主要参考

[1] [拦截器与过滤器的区别_Tang-CSDN博客_拦截器与过滤器的区别](https://blog.csdn.net/weixin_44502804/article/details/93139550)

[2] [拦截器（Interceptor）和过滤器（Filter）的执行顺序和区别_scorpio的博客-CSDN博客_过滤器和拦截器的区别](https://blog.csdn.net/zxd1435513775/article/details/80556034)

[3] [Spring 拦截器和过滤器的区别?](https://www.zhihu.com/question/30212464/answer/1786967139)

