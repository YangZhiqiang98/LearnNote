#### @Transactional 注解失效场景

[一口气说出 6种 @Transactional 注解失效场景](https://mp.weixin.qq.com/s/IcDEEft7bLhnqyo5knwUdw)



#### [annotation之@Autowired、@Inject、@Resource三者区别](https://www.cnblogs.com/pjfmeng/p/7551340.html)

1、@Autowired 是 spring 自带的，@Inject 是 JSR330 规范实现的，@Resource 是 JSR250 规范实现的，需要导入不同的包

2、@Autowired、@Inject 用法基本一样，不同的是 @Autowired 有一个 request 属性

3、@Autowired、@Inject 是默认按照类型匹配的，@Resource 是按照名称匹配的

4、@Autowired 如果需要按照名称匹配需要和 @Qualifier 一起使用，@Inject 和 @Name 一起使用

##### 相关文章

[1] [annotation之@Autowired、@Inject、@Resource三者区别 - 梦天幻 - 博客园 (cnblogs.com)](https://www.cnblogs.com/pjfmeng/p/7551340.html)