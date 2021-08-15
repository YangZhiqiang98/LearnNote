

### 拦截器与过滤器的区别

- 拦截器是基于 java 的反射机智的额，而过滤器是基于函数的回调。
- 拦截器不依赖于 servlet 容器，而过滤器依赖于 servlet 容器。
- 拦截器值对 action 请求起作用，而过滤器则可以对几乎所有的请求起作用。
- 拦截器可以访问 action 上下文、值、栈里面的对象，而过滤器不可以。
- 在 action 的生命周期里，拦截器可以多次被调用，而过滤器只能在容器初始化时被调用一次。
- 拦截器可以获取 IOC 容器中的各个 bean，而过滤器不行，在拦截器中注入一个 service ，可以调用业务逻辑。



### 执行时机

过滤前-拦截前-action执行-拦截后-过滤后

![image-20210815095510597](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210815095510597.png)

### 图解

![img](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20190620230043280.png)







### 主要参考

[1] [拦截器与过滤器的区别_Tang-CSDN博客_拦截器与过滤器的区别](https://blog.csdn.net/weixin_44502804/article/details/93139550)

