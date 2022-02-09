以下内容来自：[长文捋明白 Spring 事务！隔离性？传播性？一网打尽！](https://mp.weixin.qq.com/s/6tRPXwXnWUW4mVfCdBlkog)

### 1、什么是事务

数据库事务是指作为单个逻辑工作单元执行的一系列操作，这些操作要么一起成功，要么一起失败，是一个不可分割的工作单元。

- **原子性（Atomicity）：** 一个事务（transaction）中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。即，事务不可分割。
- **一致性（Consistency）：** 在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设约束、触发器、级联回滚等。
- **隔离性（Isolation）：** 数据库允许多个并发事务同时对其数据进行读写和修改，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括`未提交读（Read Uncommitted）`、`提交读（Read Committed）`、`可重复读（Repeatable Read）`和`串行化（Serializable）`。
- **持久性（Durability）:** 事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

这是事务的四大特性。

### 2、Spring 中的事务

####  三大基础设施

Spring 中对事务的支持提供了三大基础设施，

1. PlatformTransactionManager
2. TransactionDefinition
3. TransactionStatus

这三个核心类是 Spring 处理事务的核心类。

##### PlatformTransactionManager

`PlatformTransactionManager` 是事务处理的核心，有多个实现类：

![image-20211031153044894](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211031153052.png)

其定义如下：

```java
public interface PlatformTransactionManager {
 TransactionStatus getTransaction(@Nullable TransactionDefinition definition);
 void commit(TransactionStatus status) throws TransactionException;
 void rollback(TransactionStatus status) throws TransactionException;
}
```

1.getTransaction()

getTransaction() 是根据传入的 TransactionDefinition 获取一个事务对象，TransactionDefinition 中定义了一些事务的基本规则，如传播性、隔离级别等。

2.commit()

commit() 用来提交事务。

3.rollback()

rollback() 用来回滚事务。

##### TransactionDefinition

`TransactionDefinition ` 用来描述事务的具体规则，也称为事务的属性。主要有五种属性:

1.隔离性

2.传播性

3.回滚规则（实现类中）

4.超时时间

5.事务是否只读

`TransactionDefinition` 类中方法：

![image-20211031153957994](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211031153959.png)

其实现类：

![image-20211031154207574](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211031154209.png)



如果开发者使用了编程式事务的话，直接使用 `DefaultTransactionDefinition` 即可。

##### TransactionStatus

`TransactionStatus` 可直接理解为事务本身，

```java
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {
	boolean hasSavepoint(); //判断是否存在 savePoint()。
	void flush();//将底层会话中的修改刷新到数据库，一般用于 Hibernate/JPA 的会话，对如 JDBC 类型的事务无任何影响。
}
```

在其父类中，存在若干方法，如：

1. isNewTransaction() 方法获取当前事务是否是一个新事务。
2. setRollbackOnly() 方法设置事务必须回滚。
3. isRollbackOnly() 方法获取事务是否只能回滚。
4. isCompleted() 方法用来获取是一个事务是否结束。

###  3、编程式事务

```xml
<!--  1.配置数据源  -->
<bean class="org.springframework.jdbc.datasource.DriverManagerDataSource" id="dataSource">
    <property name="password" value="123" />
    <property name="username" value="root"/>
    <property name="url" value="jdbc:mysql://test?serverTimezone=Asia/Shanghai" />
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
</bean>
<!--  2.提供一个事务管理器 -->
<bean class="org.springframework.jdbc.datasource.DataSourceTransactionManager" id="transactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>

<!--  3.配置 TransactionTemplate  -->
<bean class="org.springframework.transaction.support.TransactionTemplate" id="transactionTemplate">
    <property name="transactionManager" ref="transactionManager" />
</bean>

<!--  4.配置 JdbcTemplate  -->
<bean class="org.springframework.jdbc.core.JdbcTemplate" id="jdbcTemplate">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

#### 3.1 PlatformTransactionManager 实现编程事务

```java
@Service
public class TransferService {
    @Autowired
    JdbcTemplate jdbcTemplate;
    @Autowired
    PlatformTransactionManager txManager;

    public void transfer() {
        //1.定义默认的事务属性
        DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
        //2.获取 TransactionStatus
        TransactionStatus status = txManager.getTransaction(definition);
        try {
            jdbcTemplate.update("update user set account=account+100 where username='zhangsan'");
            int i = 1 / 0;
            jdbcTemplate.update("update user set account=account-100 where username='lisi'");
            //提交事务
            txManager.commit(status);
        } catch (DataAccessException e) {
            e.printStackTrace();
           	//回滚事务
            txManager.rollback(status);
        }
    }
}
```

#### 3.2 TransactionTemplate 实现编程事务

```java
@Service
public class TransferService {
    @Autowired
    JdbcTemplate jdbcTemplate;
    @Autowired
    TransactionTemplate tranTemplate;
    public void transfer() {
        tranTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                try {
                    jdbcTemplate.update("update user set account=account+100 where username='zhangsan'");
                    int i = 1 / 0;
                    jdbcTemplate.update("update user set account=account-100 where username='lisi'");
                } catch (DataAccessException e) {
                    //设置当前事务回滚
                    status.setRollbackOnly();
                    e.printStackTrace();
                }
            }
        });
    }
}
```

### 4、声明式事务

声明式事务如果使用 `XML` 配置，可以做到无侵入；如果使用 `Java` 配置，也只有一个 `@Transactional` 注解侵入而已，相对来说非常容易。

#### 4.1 XML 配置

```xml
    <!--
       XML 配置事务分为三个步骤：
       1.配置事务管理器
       2.配置事务通知
       3.配置 AOP 
    -->
    <!--  配置事务管理器  -->
    <!--  1.配置数据源  -->
    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource" id="dataSource">
        <property name="password" value="123" />
        <property name="username" value="root"/>
        <property name="url" value="jdbc:mysql://test?serverTimezone=Asia/Shanghai" />
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
    </bean>
    <!--  2.提供一个事务管理器 -->
    <bean class="org.springframework.jdbc.datasource.DataSourceTransactionManager" id="transactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>
    
    <!--  配置事务通知  -->
    <tx:advice transaction-manager="transactionManager" id="txAdvice">
        <tx:attributes>
            <tx:method name="add*"/>
            <tx:method name="update*"/>
            <tx:method name="delete*"/>
        </tx:attributes>
    </tx:advice>
    <!--  配置 AOP   -->
    <aop:config>
        <aop:pointcut id="pc1" expression="execution(* com.tan.demo.*.*(..))"/>
        <aop:advisor advice-ref="txAdvice" pointcut-ref="pc1"/>
     </aop:config>
```

#### 4.2 Java 配置

```java
@Configuration
@ComponentScan
//开启事务注解支持
@EnableTransactionManagement
public class JavaConfig {
    @Bean
    DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setPassword("123");
        ds.setUsername("root");
        ds.setUrl("jdbc:mysql:///test01?serverTimezone=Asia/Shanghai");
        ds.setDriverClassName("com.mysql.cj.jdbc.Driver");
        return ds;
    }

    @Bean
    JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    @Bean
    PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

配置完成后，接下来，哪个方法需要事务就在哪个方法上添加 `@Transactional` 注解即可。当`@Transactional` 注解加在类上面的时候，表示该类的所有方法都有事务，该注解加在方法上面的时候，表示该方法有事务。

#### 4.3 混合配置

也可以 Java 代码和 XML 混合配置来实现声明式事务，就是一部分配置用 XML 来实现，一部分配置用 Java 代码来实现：

假如 XML 配置如下：

```xml
   <!--
    开启事务的注解配置，添加了这个配置，就可以直接在代码中通过 @Transactional 注解来开启事务了
    -->
    <tx:annotation-driven />
```

那么 Java 代码中的配置如下：

```java
@Configuration
@ComponentScan
@ImportResource(locations = "classpath:applicationContext3.xml")
public class JavaConfig {
    @Bean
    DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setPassword("123");
        ds.setUsername("root");
        ds.setUrl("jdbc:mysql:///test01?serverTimezone=Asia/Shanghai");
        ds.setDriverClassName("com.mysql.cj.jdbc.Driver");
        return ds;
    }

    @Bean
    JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    @Bean
    PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

Java 配置中通过 `@ImportResource` 注解导入了 XML 配置，XML 配置中的内容就是开启 `@Transactional` 注解的支持，所以 Java 配置中省略了 `@EnableTransactionManagement` 注解。

### 5、事务属性

#### 5.1 隔离性

隔离性文章：[四个案例看懂 MySQL 事务隔离级别](https://mp.weixin.qq.com/s?__biz=MzI1NDY0MTkzNQ==&mid=2247494689&idx=1&sn=59e5a30d342f9f1330830b9221aa45a8&scene=21#wechat_redirect)。

MySQL 中有四种不同的隔离级别，这四种不同的隔离级别在 Spring 中都得到了很好的支持。Spring 中默认的事务隔离级别是 default，即数据库本身的隔离级别是啥就是啥，default 就能满足我们日常开发中的大部分场景。

若要调整，调整方式如下：

##### 5.1.1 编程式事务隔离级别

**TransactionTemplate**

```java
transactionTemplate.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
```

`TransactionDefinition` 中定义了各种隔离级别。

**PlatformTransactionManager**

```java
public void update2() {
    //创建事务的默认配置
    DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
    def0inition.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
    TransactionStatus status = platformTransactionManager.getTransaction(definition);
    try {
        jdbcTemplate.update("update account set money = ? where username=?;", 999, "zhangsan");
        int i = 1 / 0;
        //提交事务
        platformTransactionManager.commit(status);
    } catch (DataAccessException e) {
          e.printStackTrace();
        //回滚
        platformTransactionManager.rollback(status);
    }
}
```

这里是在 `DefaultTransactionDefinition` 对象中设置事务的隔离级别。

##### 5.1.2 声明式事务隔离级别

XML：

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <!--以 add 开始的方法，添加事务-->
        <tx:method name="add*"/>
        <tx:method name="insert*" isolation="SERIALIZABLE"/>
    </tx:attributes>
</tx:advice>
```

Java：

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void update4() {
    jdbcTemplate.update("update account set money = ? where username=?;", 998, "lisi");
    int i = 1 / 0;
}
```

#### 5.2 传播性

何谓事务的传播性：

> 事务传播行为是为了解决业务层方法之间互相调用的事务问题，当一个事务方法被另一个事务方法调用时，事务该以何种状态存在？例如新方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行，等等，这些规则就涉及到事务的传播性。

在 Spring 中，`Propagation` 定义了事务的传播性：

```java
public enum Propagation {
    REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),
    SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),
    MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),
    REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),
    NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),
    NEVER(TransactionDefinition.PROPAGATION_NEVER),
    NESTED(TransactionDefinition.PROPAGATION_NESTED);
    private final int value;
    Propagation(int value) { this.value = value; }
    public int value() { return this.value; }
}
```

具体含义如下：

| 传播性        | 描述                                                         |
| :------------ | :----------------------------------------------------------- |
| REQUIRED      | 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务（默认） |
| SUPPORTS      | 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行 |
| MANDATORY     | 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常 |
| REQUIRES_NEW  | 创建一个新的事务，如果当前存在事务，则把当前事务挂起         |
| NOT_SUPPORTED | 以非事务方式运行，如果当前存在事务，则把当前事务挂起         |
| NEVER         | 以非事务方式运行，如果当前存在事务，则抛出异常               |
| NESTED        | 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于 TransactionDefinition.PROPAGATION_REQUIRED |

一共是七种传播性，具体配置也简单：

**TransactionTemplate中的配置**

```java
transactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
```

**PlatformTransactionManager中的配置**

```java
public void update2() {
    //创建事务的默认配置
    DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
    definition.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);
    definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    TransactionStatus status = platformTransactionManager.getTransaction(definition);
    try {
        jdbcTemplate.update("update account set money = ? where username=?;", 999, "zhangsan");
        int i = 1 / 0;
        //提交事务
        platformTransactionManager.commit(status);
    } catch (DataAccessException e) {
          e.printStackTrace();
        //回滚
        platformTransactionManager.rollback(status);
    }
}
```

**声明式事务的配置（XML）**

```xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <!--以 add 开始的方法，添加事务-->
        <tx:method name="add*"/>
        <tx:method name="insert*" isolation="SERIALIZABLE" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>
```

**声明式事务的配置（Java）**

```java
@Transactional(noRollbackFor = ArithmeticException.class,propagation = Propagation.REQUIRED)
public void update4() {
    jdbcTemplate.update("update account set money = ? where username=?;", 998, "lisi");
    int i = 1 / 0;
}
```

##### 5.2.1 REQUIRED

**REQUIRED 表示如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。**

例如有如下代码：

```java
@Service
public class AccountService {
    @Autowired
    JdbcTemplate jdbcTemplate;
    @Transactional
    public void handle1() {
        jdbcTemplate.update("update user set money = ? where id=?;", 1, 2);
    }
}
@Service
public class AccountService2 {
    @Autowired
    JdbcTemplate jdbcTemplate;
    @Autowired
    AccountService accountService;
    @Transactional
    public void handle2() {
        jdbcTemplate.update("update user set money = ? where username=?;", 1, "zhangsan");
        accountService.handle1();
    }
}
```

在 handle2 方法中调用 handle1 方法，那么：

1. 如果 handle2 方法本身是有事务的，则 handle1 方法就会加入到 handle2 方法所在的事务中，这样两个方法将处于同一个事务中，一起成功或者一起失败（不管是 handle2 还是 handle1 谁抛异常，都会导致整体回滚）。
2. 如果 handle2 方法本身是没有事务的，则 handle1 方法就会自己开启一个新的事务，自己玩。

##### 5.2.2 REQUIRES_NEW

**REQUIRES_NEW 表示创建一个新的事务，如果当前存在事务，则把==当前事务挂起==。换言之，不管外部方法是否有事务，REQUIRES_NEW 都会开启自己的事务。**

还是上面的例子，假设 handle1 和 handle2 方法都有事务，handle2 方法的事务传播性是 REQUIRED，而 handle1 方法的事务传播性是 REQUIRES_NEW，然后再 handle2 方法中调用 handle1。

此时：

1. 如果 handle2 方法没有事务，handle1 方法自己开启一个事务自己玩。
2. 如果 handle2 方法有事务，handle1 方法还是会开启一个事务。此时，如果 handle2 发生了异常进行回滚，并不会导致 handle1 方法回滚，因为 handle1 方法是独立的事务；如果 handle1 方法发生了异常导致回滚，并且 handle1 方法的异常没有被捕获处理传到了 handle2 方法中，那么也会导致 handle2 方法回滚。

##### 5.2.3 NESTED

NESTED 表示如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于 `TransactionDefinition.PROPAGATION_REQUIRED`。

假设 handle2 方法有事务，handle1 方法也有事务且传播性为 NESTED，此时，NESTED 修饰的内部方法（handle1）属于外部事务的子事务，外部主事务回滚的话，子事务也会回滚，而内部子事务可以单独回滚而不影响外部主事务和其他子事务（需要处理掉内部子事务的异常）。

##### 5.2.4 MANDATORY

**MANDATORY 表示如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。**

##### 5.2.5 SUPPORTS

**SUPPORTS 表示如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。**

假设 handle2 方法有事务，handle1 方法也有事务且传播性为 SUPPORTS，那么 handle1 会加入到 handle2 事务中，不会创建新的事务。

假设 handle2 方法无事务，handle1 方法有事务且传播性为 SUPPORTS，这个最终就不会开启事务。

##### 5.2.6 NOT_SUPPORTED

**NOT_SUPPORTED 表示以非事务方式运行，如果当前存在事务，则把当前事务挂起。**

##### 5.2.7 NEVER

**NEVER 表示以非事务方式运行，如果当前存在事务，则抛出异常。**

假设 handle2 方法有事务，handle1 方法也有事务且传播性为 NEVER，那么最终会抛出如下异常：

```java
Existing transaction found for transaction marked with propagation 'never'
```

#### 5.3 回滚规则

默认情况下，事务只有遇到运行期异常（`RuntimeException` 的子类）以及 Error 时才会回滚，在遇到检查型（Checked Exception）异常时不会回滚。

> [Java 异常体系](https://www.liaoxuefeng.com/wiki/1252599548343744/1264734349295520)
>
> ![java异常体系](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211031194644.png)

像 1/0，空指针这些是 RuntimeException，而 IOException 则算是 Checked Exception，换言之，默认情况下，如果发生 IOException 并不会导致事务回滚。

如果我们希望发生 IOException 时也能触发事务回滚，那么可以按照如下方式配置：

Java 配置：

```java
@Transactional(rollbackFor = IOException.class)
public void handle2() {
    jdbcTemplate.update("update user set money = ? where username=?;", 1, "zhangsan");
    accountService.handle1();
}
```

XML 配置：

```xml
<tx:advice transaction-manager="transactionManager" id="txAdvice">
    <tx:attributes>
        <tx:method name="m3" rollback-for="java.io.IOException"/>
    </tx:attributes>
</tx:advice>
```

另外，我们也可以指定在发生某些异常时不回滚，例如当系统抛出 ArithmeticException 异常并不要触发事务回滚，配置方式如下：

Java 配置：

```java
@Transactional(noRollbackFor = ArithmeticException.class)
```

XML 配置：

```xml
<tx:advice transaction-manager="transactionManager" id="txAdvice">
    <tx:attributes>
        <tx:method name="m3" no-rollback-for="java.lang.ArithmeticException"/>
    </tx:attributes>
</tx:advice>
```

#### 5.4 是否只读

只读事务一般设置在查询方法上，但不是所有的查询方法都需要只读事务，要看具体情况。

一般来说，如果这个业务方法只有一个查询 SQL，那么就没必要添加事务，强行添加最终效果适得其反。

但是如果一个业务方法中有多个查询 SQL，情况就不一样了：多个查询 SQL，默认情况下，每个查询 SQL 都会开启一个独立的事务，这样，如果有并发操作修改了数据，那么多个查询 SQL 就会查到不一样的数据。此时，如果我们开启事务，并设置为只读事务，那么多个查询 SQL 将被置于同一个事务中，多条相同的 SQL 在该事务中执行将会获取到相同的查询结果。

设置事务只读的方式如下：

Java 配置：

```java
@Transactional(readOnly = true)
```

XML 配置：

```xml
<tx:advice transaction-manager="transactionManager" id="txAdvice">
    <tx:attributes>
        <tx:method name="m3" read-only="true"/>
    </tx:attributes>
</tx:advice>
```

#### 5.5 超时时间

超时时间是说一个事务允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。

事务超时时间配置方式如下(单位为秒)：

Java 配置：

```java
@Transactional(timeout = 10)
```

XML 配置：

```xml
<tx:advice transaction-manager="transactionManager" id="txAdvice">
    <tx:attributes>
        <tx:method name="m3" read-only="true" timeout="10"/>
    </tx:attributes>
</tx:advice>
```

在 `TransactionDefinition` 中以 `int` 的值来表示超时时间，其单位是秒，默认值为 -1。

### 6、注意事项

1. 事务只能应用到 public 方法上才会有效。
2. 事务需要从外部调用，**Spring 自调事务用会失效**。即**相同类**里边，A 方法没有事务，B 方法有事务，A 方法调用 B 方法，则 B 方法的事务会失效，这点尤其要注意，因为代理模式只拦截通过代理传入的外部方法调用，所以自调用事务是不生效的。
3. 建议事务注解 @Transactional 一般添加在实现类上，而不要定义在接口上，如果加在接口类或接口方法上时，只有配置基于接口的代理这个注解才会生效。

4. [@Transactional注解失效的6种场景](https://blog.csdn.net/hon_vin/article/details/105134342)

