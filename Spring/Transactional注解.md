## @Transactional 详解

### 概述

- @Transactional 是声明式[事务管理](https://blog.csdn.net/mingyundezuoan/article/details/79167332) 编程中使用的注解
- 添加位置：接口实现类或者接口实现方法上，而不是接口类中。
  - Spring团队的建议是你在具体的类（或类的方法）上使用 @Transactional 注解，而不要使用在类所要实现的任何接口上。你当然可以在接口上使用 @Transactional 注解，但是这将只能当你设置了基于接口的代理时它才生效。因为注解是不能继承的，这就意味着如果你正在使用基于类的代理时，那么事务的设置将不能被基于类的代理所识别，而且对象也将不会被事务代理所包装（将被确认为严重的）。因此，请接受Spring团队的建议并且在具体的类上使用 @Transactional 注解。
- 访问权限：public的方法才有用
  - @Transactional 注解应该只被应用在public方法上，这是由Spring AOP的本质决定的，对于其它非public的方法,如果标记了@Transactional也不会报错,但方法没有事务功能。

- 错误使用
  - 接口中A、B两个方法，A无@Transactional标签，B有，上层通过A间接调用B，此时事务不生效
  - 默认配置下，spring只有在抛出的异常为`unchecked`异常时才会回滚该事物，也就是抛出的的异常为`RuntimeException`以及`Error`时会回滚。而抛出`checked`异常不会回滚。可通过rollbackFor属性进行设置。
  - 多线程下事物管理
    - 多线程下事务管理因为线程不属于 spring 托管，故线程不能够默认使用 spring 的事务,  也不能获取spring 注入的 bean 。
    -   在被 spring 声明式事务管理的方法内开启多线程，多线程内的方法不被事务控制。一个使用了@Transactional 的方法，如果方法内包含多线程的使用，方法内部出现异常，不会回滚线程中调用方法的事务

### 原理

@Transactional 实质是使用了JDBC的事物来进行事务控制的，基于Spring的动态代理的机制。

@Transactional实现原理
	1)事务开始时，通过AOP机制，生成一个代理connection对象，并将其放入DataSource实例的某个与DataSourceTransactionManager相关的某处容器中。在接下来的整个事务中，客户代码都应该使用该connection连接数据库，执行所有数据库命令[不使用该connection连接数据库执行的数据库命令，在本事务回滚的时候得不到回滚]（物理连接connection逻辑上新建一个会话session；DataSource与TransactionManager配置相同的数据源）
	2)事务结束时，回滚在第1步骤中得到的代理connection对象上执行的数据库命令，然后关闭该代理connection对象（事务结束后，回滚操作不会对已执行完毕的SQL操作命令起作用）

### 常用参数

| 参数名称                   | 功能描述                                                     |
| -------------------------- | ------------------------------------------------------------ |
| **readOnly**               | 该属性用于设置当前事务是否为只读事务                         |
| **rollbackfor**            | 该属性用于设置需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，则进行事务回滚。如：<br/>指定单一异常类：@Transactional(rollbackFor=RuntimeException.class)<br/>指定多个异常类：@Transactional(rollbackFor={RuntimeException.class, Exception.class}) |
| **rollbackForClassName**   | 该属性用于设置需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，则进行事务回滚。如：<br/>指定单一异常类名称：@Transactional(rollbackForClassName="RuntimeException") <br/>指定多个异常类名称：@Transactional(rollbackForClassName={"RuntimeException","Exception"}) |
| **noRollbackFor**          | 该属性用于设置不需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，不进行事务回滚。 |
| **noRollbackForClassName** | 该属性用于设置不需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，不进行事务回滚。 |
| **propagation**            | 该属性用于设置事务的传播行为。                               |
| **isolation**              | 该属性用于设置底层数据库的事务隔离级别，事务隔离级别用于处理多事务并发的情况，通常使用数据库的默认隔离级别即可。 |
| **timeout**                | 该属性用于设置事务的超时秒数，默认值为-1永不超时             |

### 事务隔离级别

- 事务隔离级别：指若干个并发的事务之间的隔离程度

- @Transactional(isolation = Isolation.READ_UNCOMMITTED)：读取未提交数据(会出现脏读, 不可重复读) 基本不使用

- @Transactional(isolation = Isolation.READ_COMMITTED)：读取已提交数据(会出现不可重复读和幻读)

- @Transactional(isolation = Isolation.REPEATABLE_READ)：可重复读(会出现幻读)

- @Transactional(isolation = Isolation.SERIALIZABLE)：串行化

### 事务传播行为

- 事务传播行为：如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为

  | 参数名称          | 功能描述                                                     |
  | ----------------- | ------------------------------------------------------------ |
  | **REQUIRED**      | 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。（**默认值**） |
  | **REQUIRES_NEW**  | 创建一个新事务，如果当前事务存在，则把当前事务挂起           |
  | **SUPPORTS**      | 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行 |
  | **NOT_SUPPORTED** | 以非事务的方式运行，如果当前存在事务，则把当前事务挂起       |
  | **NEVER**         | 以非事务的方式运行，如果当前存在事务，则抛出异常             |
  | **MANDATORY**     | 如果当前存在事务，则加入该事务;如果当前没有事务，则抛出异常  |
  | **NESTED**        | 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于REQUIRED |

### 嵌套事务

​	带有事务的方法调用其他事务的方法，此时执行的情况取决配置的事务的传播属性。

-  **PROPAGATION_REQUIRES_NEW** ：**启动一个新的，不依赖外部环境的”内部“事务**。

​	这个事务将被完全commit或者rollback而不依赖于外部事务。它拥有自己的隔离范围，自己的锁等等，当内部事务开始执行时，外部事务将被挂起，内部事务结束时，外部事务将继续执行。

- **PROPAGATION_NESTED** ： **如果外部事务commit，嵌套事务也会被commit；如果外部事务rollback，嵌套事务也会被rollback。**

  开始的一个“嵌套”的事务，它是已经存在事务的一个真正的子事务。嵌套事务开始执行时，它将取得一个savepoint，如果这个嵌套事务失败，将回滚到此savepoint，嵌套事务是外部事务的一部分，只有外部事务结束后它才会被提交。

### Spring 事务回滚规则

- spring 事务管理器回滚一个事务的推荐方法是在当前事务的上下文内抛出异常。spring事务管理器会捕获任务任何未处理的异常，然后依据规则决定是否回滚抛出异常的事务。
- 默认配置下，spring只有在抛出的异常为运行时unchecked异常时才会回滚该事务，也就是抛出的异常为RuntimeException的子类（Error 也会导致事务回滚），而抛出checked异常则不会导致事务回滚。
- 可通过属性rollbackFor属性改变这种行为

### 多线程事务管理

- 描述
  - 因为线程不属于spring托管，故线程不能够默认使用spring的事务,也不能获取spring注入的bean
  - 在被spring声明式事务管理的方法内开启多线程，多线程内的方法不被事务控制‘
- 解决
  - 如果方法中调用多线程
    - 方法主体的事务不会传递到线程中
    - 线程中可以单独调用Service接口，接口的实现方法使用@Transactional，保证线程内部的事务
  - 多线程实现的方法
    - 使用异步注解@Async的方法上再加上注解@Transactional，保证新线程调用的方法是有事务管理的
- 原理
  - Spring中事务信息存储在ThreadLocal变量中，变量是某个线程上进行的事务所特有的(这些变量对于其他线程中发生的事务来讲是不可见的，无关的)
  - 单线程的情况下，一个事务会在层级式调用的Spring组件之间传播
  - 在@Transactional注解的服务方法会产生一个新的线程的情况下，事务是不会从调用者线程传播到新建线程的

### 注意

[@Transactional注解失效的6种场景](https://blog.csdn.net/hon_vin/article/details/105134342)

