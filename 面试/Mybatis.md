#### MyBatis缓存机制

##### 一级缓存

###### 基本原理

每个 SqlSession 中持有了 Executor，每个 Executor中 有一个 LocalCache。当用户发起查询时，MyBatis 根据当前执行的语句生成`MappedStatement`，在 Local Cache 进行查询，如果缓存命中的话，直接返回结果给用户，如果缓存没有命中的话，查询数据库，结果写入`Local Cache`，最后返回结果给用户。



###### 总结

1. MyBatis 一级缓存的生命周期和 SqlSession 一致。
2. MyBatis 一级缓存内部是一个没有容量限定的 HashMap。
3. MyBatis 的一级缓存最大范围是 SqlSession 内部，有多个 SqlSession 或者分布式的环境下，数据库写操作会引起脏数据。
4. 任何的 UPDATE, INSERT, DELETE 语句都会清空缓存。



##### 二级缓存

###### 基本原理

一级缓存中，其最大的共享范围就是一个 SqlSession 内部，如果多个 SqlSession 之间需要共享缓存，则需要使用到二级缓存。开启二级缓存后，会使用 CachingExecutor 装饰 Executor，进入一级缓存的查询流程前，先在 CachingExecutor 进行二级缓存的查询。二级缓存开启后，同一个 **namespace** 下的所有操作语句，都影响着同一个 Cache，即二级缓存被多个 SqlSession 共享，是一个全局的变量。



当开启缓存后，数据的查询执行的流程就是 二级缓存 -> 一级缓存 -> 数据库。



###### 总结

1. MyBatis 的二级缓存相对于一级缓存来说，实现了`SqlSession`之间缓存数据的共享，同时粒度更加的细，能够到`namespace`级别，通过Cache接口实现类不同的组合，对 Cache 的可控性也更强。
2. MyBatis 在多表查询时，极大可能会出现脏数据，有设计上的缺陷，安全使用二级缓存的条件比较苛刻。
3. 在分布式环境下，由于默认的 MyBatis Cache 实现都是基于本地的，分布式环境下必然会出现读取到脏数据，需要使用集中式缓存将 MyBatis 的 Cache 接口实现，有一定的开发成本，直接使用 Redis、Memcached 等分布式缓存可能成本更低，安全性也更高。

##### 相关文章

[1] [聊聊MyBatis缓存机制 - 美团技术团队 (meituan.com)](https://tech.meituan.com/2018/01/19/mybatis-cache.html)

[2] [mybatis 缓存的使用， 看这篇就够了_Homejim的博客-CSDN博客_mybatis缓存](https://blog.csdn.net/weixin_37139197/article/details/82908377)