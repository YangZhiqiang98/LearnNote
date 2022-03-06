## 基础篇

### 一条 SQL 查询语句是如何执行的？

#### 讲课内容

下面是 MySQL 的基本架构示意图，从中可以清楚地看到 SQL 语句在 MySQL 的各个功能模块中的执行过程。



![MySQL 的基本架构示意图](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220216214134.webp)



大体来说，MySQL 可以分为 Server 层和存储引擎层两部分。

- Server 层包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。
- 存储引擎层负责 **数据的存储和提取**。其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎。现在最常用的存储引擎是 InnoDB，它从 MySQL 5.5.5 版本开始成为了默认存储引擎。

> InnoDB 和 MyISAM 区别：
>
> 1、InnoDB 支持事务，而 MyISAM 不支持事务
>
> 2、InnoDB 支持行级锁，而 MyISAM 支持表级锁 
>
> 3、InnoDB 支持 MVCC, 而 MyISAM 不支持 
>
> 4、InnoDB 支持外键，而 MyISAM 不支持 
>
> 5、InnoDB 不支持全文索引，而 MyISAM 支持。(MySQL5.6以后，InnoDB 已经可以支持全文索引了)



##### 连接器

**连接器负责跟客户端建立连接、获取权限、维持和管理连接**。

连接完成后，如果你没有后续的动作，这个连接就处于空闲状态，你可以在 `show processlist` 命令中看到它。

![](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20220216220953.webp)



客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数 `wait_timeout` 控制的，默认值是 8 小时(28800 秒)。

```mysql
 show variables like 'wait_timeout';
```

数据库里面，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。



建立连接的过程通常是比较复杂的，所以建议在使用中要尽量减少建立连接的动作，也就是 **尽量使用长连接**（通过数据库连接池维护一些长连接）。



但是全部使用长连接后，可能会出现有些时候 MySQL 占用内存涨得特别快，这是因为 MySQL 在执行过程中临时使用的 **内存是管理在连接对象里面的**。这些资源会在连接断开的时候才释放。所以如果长连接累积下来，可能导致内存占用太大，被系统强行杀掉（**OOM**），从现象看就是 MySQL **异常重启** 了。



解决方法： 

1、**定期断开长连接**。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连。

2、如果你用的是 MySQL 5.7 或更新版本，可以在每次执行一个比较大的操作后，通过执行 **`mysql_reset_connection` 来重新初始化连接资源**。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。



> [MySQL :: MySQL 8.0 C API Developer Guide :: 5.4.62 mysql_reset_connection()](https://dev.mysql.com/doc/c-api/8.0/en/mysql-reset-connection.html)
>
> mysql_reset_connection() 影响以下与会话状态相关的信息：
>
> - 回滚活跃事务并重新设置自动提交模式 ·释放所有的表锁
> - 关闭或删除所有的临时表
> - 重新初始化会话的系统变量值
> - 丢失用户定义的变量设置
> - 释放 prepared 语句
> - 关闭 handler 变量
> - 将 last_insert_id() 的值设置为 0
> - 释放 get_lock() 获取的锁
> - 清空通过 mysql_bind_param() 调用定义的当前查询属性 返回值： 0 表示成功，非 0 表示发生了错误。



##### 查询缓存

连接建立完成后，执行逻辑就会来到第二步：查询缓存。



MySQL 拿到一个查询请求后，会先到查询缓存看看，之前是不是执行过这条语句。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。如果你的查询能够直接在这个缓存中找到 key，那么这个 value 就会被直接返回给客户端。



> The query cache is deprecated as of MySQL 5.7.20, and is removed in MySQL 8.0.
>
> 查询缓存在

#### 总结问题

