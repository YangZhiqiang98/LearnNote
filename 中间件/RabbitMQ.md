[TOC]
# 书籍《RabbitMQ实战指南》

## 基础概念

### 消息中间件的作用

1. 解耦
2. 冗余（存储）
3. 扩展性
4. 削峰
5. 可恢复性
6. 顺序保证
7. 缓冲
8. 异步通信

### RabbitMQ

默认情况下，rabbitmq 服务的用户名和密码都是 `guest`，但这个默认账户只能通过本地网络访问，远程网络访问受限。解决办法，

1、修改配置文件或者命令，去除 guest 账户限制。（不推荐）相关配置详解：https://www.rabbitmq.com/access-control.html,  https://www.rabbitmq.com/configure.html,

![image-20200930062741810](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211028062534.png)

![image-20200930062334757](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211028062532.png)

2、添加一个新的账号。相关配置详解：[https://www.rabbitmq.com/configure.html](https://www.rabbitmq.com/configure.html),  	

```
rabbitmqctl add_user root 123
rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
rabbitmqctl set_user_tags root administrator
```

设置的权限语句，默认 default_vhost 为 `/`，三个权限分别为  `configure`，`read`，`write`。

![image-20200930063341114](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211028062608.png)



### 交换器类型

常用的类型有四种，`fanout`、`direct`、`topic`、`headers`。

* fanout：把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中。

* direct：把消息路由到 BindingKey 和 RoutingKey **完全匹配**的队列中。

* topic：在 direct 交换器的匹配规则上进行扩展，将消息路由到 BindingKey 和 RoutingKey **相匹配**的队列中。

  匹配规则:

  - RoutingKey 为 `.` 分隔的字符串，（被 `.` 分隔开的每一段独立的字符串成为一个单词）。如：“com.rabbitmq.client”
  - BindingKey 和 RoutingKey 一样也是 `.` 分隔的字符串。
  - BindingKey 中也可以存在两种特殊字符串 `*` 和 `#`，用于做模糊匹配。其中，`*`用于匹配一个单词，`#` 用于匹配多个单词（也可以是 0 个）。

* headers：headers 类型的交换器不依赖路由键的匹配规则来路由消息，而是根据发送的消息的消息内容中的 headers 属性进行匹配。（性能很差，不实用）

### 生产者-消费者示例

生产者

```java
public class Provider {
  public static final String EXCHANGE_NAME = "exchange_demo";
  public static final String ROUTING_KEY = "routingkey_demo";
  public static final String QUEUE_NAME = "queue_demo";
  public static final String IP_ADDRESS = "49.235.238.92";
  /**端口默认为5672 在创建时可自行修改*/
  public static final int PORT = 5672;
  public static void main(String[] args) throws IOException, TimeoutException {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost(IP_ADDRESS);
    factory.setPort(PORT);
    factory.setUsername("root");
    factory.setPassword("123");
    // 创建连接
    Connection connection = factory.newConnection();
    // 创建通道
    Channel channel = connection.createChannel();
    // 创建一个type为direct 、 持久化的、非自动删除的交换器
    channel.exchangeDeclare(EXCHANGE_NAME, "direct", true, false, null);
    // 创建一个持久化的、非独占的、非自动删除的队列
    channel.queueDeclare(QUEUE_NAME, true, false, false, null);
    // 将交换器与队列通过路由键绑定
    channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, ROUTING_KEY);
    // 发送一条持久化的消息
    String message = "hello world";
    channel.basicPublish(EXCHANGE_NAME,ROUTING_KEY, MessageProperties.PERSISTENT_TEXT_PLAIN,
        message.getBytes());

    // 关闭资源  channel 非必要 在connection关闭时，channel自动关闭
    channel.close();
    connection.close();
  }
}

```

消费者

```java
public class Consumer {
  public static final String QUEUE_NAME = "queue_demo";
  public static final String IP_ADDRESS = "49.235.238.92";
  /**端口默认为5672 在创建时可自行修改*/
  public static final int PORT = 5672;

  public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
    Address[] addresses = new Address[]{
        new  Address(IP_ADDRESS, PORT)
    };

    ConnectionFactory factory = new ConnectionFactory();
    factory.setUsername("root");
    factory.setPassword("123");
    factory.setConnectionTimeout(10000);
    Connection connection = factory.newConnection(addresses);
    final Channel channel = connection.createChannel();
    // 设置客户端最多接受未被ack的消息的个数
    channel.basicQos(64);
    DefaultConsumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        System.out.println("recv message: " + new String(body));
        try{
          TimeUnit.SECONDS.sleep(1);
        }catch (InterruptedException e){
          e.printStackTrace();
        }
        channel.basicAck(envelope.getDeliveryTag(), false);
      }
    };
    channel.basicConsume(QUEUE_NAME, consumer);
    TimeUnit.SECONDS.sleep(5);
    channel.close();
    connection.close();
  }
}

```



### 部分方法详解

#### 交换器相关

###### 1、exchangeDeclare ：声明一个交换器（有许多重载方法，缺省方法中的某些参数构成）

```java
Queue.DeclareOk exchangeDeclare(String exchange,String type, boolean durable, boolean autoDelete, boolean internal, Map<String, Object> arguments) throws IOException;
```

参数说明：

* exchange：交换器的名称。
* type：交换器的类型。常见的有 fanout、direct、topic
* durable：设置是否持久化。true，代表进行持久化，false， 代表非持久化。持久化可以将交换器存盘，在服务器重启的时候不会丢失相关信息。
* autoDelete：设置是否自动删除。true，代表自动删除。**自动删除的前提是至少有一个队列或者交换器与这个交换器绑定，之后所有与这个交换器绑定的队列或者交换器都与此解绑**。不能错误的理解为：“当与此交换器连接的客户端都断开时，RabbitMQ会自动删除本交换器”。
* internal：设置为是否是内置的。如果设置为true，则表示是内置的交换器，客户端程序无法直接发送消息到这个交换器中，只能通过交换器路由到交换器这种方式。
* argument： 其他一些结构化参数。

###### 2、exchangeDeclarePassive

```java
Exchange.DeclareOk exchangeDeclarePassive(String name) throws IOException;
```

主要用来检测相应的交换器是否存在。如果存在则正常返回，如果不存在则抛出404 channel exception，同时channel也会被关闭。

###### 3、exchangeDelete：删除交换器

```java
Exchange.DeleteOk exchangeDelete(String exchange, boolean ifUnused) throws IOException;
```

参数说明：

- exchange：交换器的名称。
- ifUnused：设置是否在交换器没有被使用的情况下删除。

###### 4、exchangeBind 将交换器与交换器绑定

```java
Exchange.BindOk exchangeBind(String destination, String source, String routingKey, Map<String, Object> arguments) throws IOException;
```



#### 队列相关

###### 1、queueDeclare：声明一个队列

重载方法有两个

（1）Queue.DeclareOk queueDeclare() throws IOException：默认创建一个由RabbitMQ命名的（匿名队列）、排他的、自动删除的、非持久化的队列。

（2）

```java
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
Map<String, Object> arguments) throws IOException 
```

参数说明：

- queue： 队列的名称。
- durable：设置是否持久化。持久化可以将交换器存盘，在服务器重启的时候不会丢失相关信息。
- exclusive：设置是否排他。如果一个队列被声明为排他队列，**该队列仅对首次声明它的连接可见**，并在连接断开时自动删除。
  - 排他队列是**基于连接（Connection）可见**的，同一个连接的不同信道（Channel）是可以同时访问同一连接创建的排他队列。
  - “首次”是指如果一个连接已经声明了一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同。
  - 即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除，这种队列适用于一个客户端同时发送和读取消息的应用场景。
- autoDelete：设置是否自动删除。true，代表自动删除。自动删除的前提是：至少有一个消费者连接到这个队列，之后所有与这个队列连接的消费者都断开时，才会自动删除。在生产者创建这个队列或者没有消费者与这个队列连接时，都不会自动删除这个队列。
- arguments： 其他一些结构化参数，如x-message-ttl、x-expires、x-max-length、x-dead-letter-exchange，x-dead-letter-routing-key等。

###### 2、Queue.DeclareOk queueDeclarePassive(String queue) throws IOException;

```java
Queue.DeclareOk queueDeclarePassive(String queue) throws IOException;
```

主要用来检测相应的队列是否存在。如果存在则正常返回，如果不存在则抛出404 channel exception，同时channel也会被关闭。

##### 3、queueDelete：删除队列

```java
Queue.DeleteOk queueDelete(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;
```

参数说明：

- queue： 队列的名称。
- ifUnused：设置是否在队列没有被使用的情况下删除。
- ifEmpty：设置是否在队列为空（队列里面没有任何消息堆积）的情况下删除。

4、queuePurge：清空队列中的内容，而不删除队列本身。

```java
Queue.PurgeOk queuePurge(String queue) throws IOException;
```

###### 5、queueBind ：将队列和交换器绑定

```java
Queue.BindOk queueBind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```

参数说明：

- queue：队列名称。
- exchange：交换器的名称。
- routingKey：用来绑定队列和交换器的路由键。
- argument：定义绑定的一些参数。

6、queueUnbind：将队列和交换器进行解绑。

```java
Queue.UnbindOk queueUnbind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```

#### 发送消息

###### basicPublish

```java
void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body) throws IOException;
```



```java
void basicPublish(String exchange, String routingKey, boolean mandatory, BasicProperties props, byte[] body) throws IOException;
```



```java
 void basicPublish(String exchange, String routingKey, boolean mandatory, boolean immediate, BasicProperties props, byte[] body) throws IOException;
```

参数说明：

- exchange：交换的名称，指明消息需要发送到哪个交换器中。如果设置为空字符串，则消息会被发送到RabbitMQ默认的交互器中。
- routingKey：路由键，交换器根据路由键将消息存储到相应的队列之中。
- props：消息的基本属性集，其中包含14个属性成员，分别有contentType、contentEncoding、headers（Map<String,Object>）、deliveryMode、priority、correlationId、replyTo、expiration、messageId、timestamp、type、userId、appId、clusterId。
- byte[] body：消息体（payload），真正需要发送的消息。
- mandatory：**当参数mandatory参数设为true时，交换器无法根据自身的类型和路由键找到一个符合条件的队列时，那么RabbitMQ会调用Basic.Return命令将消息返回给生产者。当mandatory参数设置为false时，出现上述情形，则消息直接被丢弃**。
- immediate：**当immediate参数设为true时，如果交换器在消息路由到队列时发现队列上并不存在任何消费者，那么这条消息不会存入队列中。当与路由键匹配的所有队列都没有消费者时，该消息会通过Basic.Return返回至生产者。**（已废弃）

概括来说，mandatory和immediate都有在消息传递过程中不可达目的地时将消息返回给生产者的功能。mandatory参数告诉服务器至少将该消息路由到一个队列中，否则将消息返回给生产者。immediate参数告诉服务器，如果该消息关联的队列上有消费者，则立刻投递；如果所有匹配的队列上都没有消费者，则直接将消息返还给生产者，不用将消息存入队列而等待消费者了。

#### 消费消息

##### 1、推模式 ：采用Basic.Consume消费

```java
 String basicConsume(String queue, boolean autoAck, String consumerTag, Consumer callback) throws IOExcept
```

在推模式中，可以通过持续订阅的方式来消费消息，实用的相关类：

```java
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
```

接受消息一般通过实现Consumer接口或者继承DefaultConsumer类来实现。不同的订阅采用不同的消费者标签（consumerTag）来区分彼此，在同一个Channel中的消费者也需要通过唯一的消费者标签进行区分。

```java
    boolean autoAck = false;
    channel.basicQos(64);
    channel.basicConsume(QUEUE_NAME, autoAck, "myconsumer",new DefaultConsumer(channel){
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String routingKey = envelope.getRoutingKey();
        String contentType = properties.getContentType();
        long deliveryTag = envelope.getDeliveryTag();
        System.out.println(new String(body));
        channel.basicAck(deliveryTag, false);
      }
    });
```

###### basicConsume

```java
 String basicConsume(String queue, boolean autoAck, String consumerTag, boolean noLocal, boolean exclusive, Map<String, Object> arguments, DeliverCallback deliverCallback, CancelCallback cancelCallback, ConsumerShutdownSignalCallback shutdownSignalCallback) throws IOException;
```

参数说明：

- consumerTag ： 消费者标签，用来区分多个消费者。
- noLocal：设置为true则表示不能将同一个Connection中生产者发送的消息传递给这个Connection中的消费者。
- callback：设置消费者的回调函数。用来处理RabbitMQ推送过来的消息。

重写的方法：

```java
 void handleConsumeOk(String consumerTag);
 void handleCancelOk(String consumerTag);
 void handleCancel(String consumerTag);
 void handleShutdownSignal(String consumerTag, ShutdownSignalException sig);
 void handleRecoverOk(String consumerTag);
 void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
```

##### 2、拉模式：调用 Basic.get 消费

通过 channel.basicGet 方法可以单条地获取消息。

```java
GetResponse basicGet(String queue, boolean autoAck) throws IOException;
```

```java
    GetResponse response = channel.basicGet(QUEUE_NAME, false);
    System.out.println(new String(response.getBody()));
    channel.basicAck(response.getEnvelope().getDeliveryTag(),false);

```


#### 消费端的确认与拒绝

##### 消息确认机制

为了保证消息从队列可靠地到达消费者并被消费，rabbit 提供了消息确认机制（message acknowledgement）。消费者在订阅队列时，可以指定 autoAck 参数，当 autoAck 参数为 false 时，rabbit 会等待消费者显示地回复确认信号后才从内存（或者磁盘）中移去消息（实质上是先打上删除标记，之后再删除）。当 autoAck等于 true 时，RabbitMQ 会自动把发送出去的消息置为确认，然后从内存（或者磁盘）中删除，而不管消费者是否真正地消费到了这些消息。

那么当 autoAck 为 false 时，如果 RabbitMQ 一直没有收到消费者的确认信号，并且消费此消息的消费者已经断开连接，则 RabbitMQ 会安排该消息重新进入队列，等待投递给下一个消费者，当然也可能是原来那个消费者。（30 分钟默认）

##### 消息拒绝

###### basicReject：一次拒绝一条消息

```java
  void basicReject(long deliveryTag, boolean requeue) throws IOException;
```

参数说明：

- deliveryTag：可以看作消息的编号，是一个 64 位的长整数值，最大值是 9223372036854775807。
- requeue：如果设置为 true，则 RabbitMQ 会重新将这条消息存入队列，以便可以发送给下一个订阅的消费者；如果设置为 false，则 RabbitMQ 立即会把消息从队列中移除，而不会把它发送给新的消费者。

###### basicNack：批量拒绝消息

```java
void basicNack(long deliveryTag, boolean multiple, boolean requeue) throws IOException;
```

参数说明：

- multiple：设置为 false，表示拒绝编号为 deliveryTag 的这一条消息，此时和 basicReject 一样；设置为 true，则表示拒绝 deliveryTag 编号之前的所有未被当前消费者确认的消息。

> basicReject basicNack 的 requeue 参数设置为 false，可以启用“死信队列”的功能。死信队列可以通过检测被拒绝或者未送达的消息来追踪问题。

##### 可重入队列

###### basicRecover：请求 RabbitMQ 重新发送还未被确认的消息

```java
Basic.RecoverOk basicRecover() throws IOException;
Basic.RecoverOk basicRecover(boolean requeue) throws IOException;
```
参数说明：

- requeue：设置为 true，则未被确认的消息会被重新加入到队列中，这样对于同一条消息，可能会被分配给与之前不同的消费者。如果 requeue 参数设置为   false，那么同一个消息会被分配给与之前相同的消费者。默认 requeue 为 true。



#### 关闭连接
```java
channel.close();
conn.close();	
```

Connection 和 Channel 所具备的生命周期如下：

1. open: 开启状态，代表当前对象可用。
2. Closing：正在关闭状态。当前对象被显式地通知调用关闭方法（shutdown），这样就产生了一个关闭请求让其内部对象进行相应的操作，并等待这些关闭操作的完成。
3. Closed：已经关闭状态。当前对象已经接受到所有内部对象已完成关闭动作的通知，并且其也关闭了自身。

与关闭相关的其他方法：

```java
addShutdownListener(ShutdownListener listener)
removeShutdownListener(ShutdownListener listener)
```

在 Connection 和 Channel 的状态转变为 Closed 的时候会调用 ShutdownListener。而且如果将一个 ShutdownListener 注册到一个已经处于关闭状态的对象（Connection 和 Channel 对象），会立刻调用 ShutdownListener。

```java
getCloseReason:得到对象关闭的原因。
isOpen：检测对象当前是否处于开启状态。
close(int closeCode,String closeMessage):显式地通知当前对象执行关闭操作。
```

## RabbitMQ进阶
### 消息何去何从
#### mandatory参数

**当参数 mandatory 参数设为 true 时，交换器无法根据自身的类型和路由键找到一个符合条件的队列时，那么 RabbitMQ 会调用 Basic.Return 命令将消息返回给生产者。当 mandatory 参数设置为 false 时，出现上述情形，则消息直接被丢弃**。

生产者通过调用 channel.addReturnListene r来添加 ReturnListener 监听器来获取到没有被正确路由到合适队列的消息。

示例如下：

```java
channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, MessageProperties.PERSISTENT_TEXT_PLAIN, 
                     "mandatory".getBytes());

channel.addReturnListener(new ReturnListener() {
    @Override
    public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, 									AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body);
        System.out.println("Basic.Return 返回的结果是：" + message);
    }
});
```

#### 备份交换器AE（Alternate Exchange）
##### 引入原因
生产者在发送消息时候如果不设置mandatory参数，那么消息在未被路由的情况下将会丢失；如果添加了 mandatory 参数，那么需要添加 ReturnListener 的编程逻辑，这样会让生产者的编程变得复杂。如果既不想复杂化生产者的编程逻辑，又不想消息丢失，可以使用备份交换器，可以将未被路由的消息存放在 RabbitMQ 中，再在需要的时候去处理这些消息。
##### 实现方式
1、通过在声明交换器（channel.exchangeDeclare 方法）添加 alternate-exchange 参数来实现。（优先级别更高）。

```java
Map<String,Object> args = new HashMap<>();
args.put("alternate-exchange", "myAe");
channel.exchangeDeclare("normalExchange", "direct", true, false, args);//将myAe设置为normalExchange的备份交换器
channel.exchangeDeclare("myAe","fanout",true,false,null);
```

为了方便使用，备份交换器建议设置为fanout类型。原因：**消息被重新发送到备份交互器时的路由键和从生产者发出的路由键是一样的**。

AE的几种特殊情况：

- 如果设置的备份交换器不存在，客户端和 RabbitMQ 服务端都不会出现异常，此时消息会丢失。
-  如果备份交换器没有绑定任何队列，客户端和 RabbitMQ 服务端都不会出现异常，此时消息会丢失。
- 如果备份交换器没有任何匹配的队列，客户端和 RabbitMQ 服务端都不会出现异常，此时消息会丢失。
- 如果备份交换器和 mandatory 参数一起使用，那么 mandatory 参数无效。

2、通过策略（Policy）实现。

#### 过期时间（TTL）

TTL（Time to Live）,RabbitMQ 可以对消息和队列设置 TTL。

##### 设置消息的TTL

设置消息的TTL的两种方法：
1、通过队列属性设置，队列中所有消息都具有相同的过期时间。

```java
Map<String,Object> args = new HashMap<>();
args.put("x-message-ttl", 6000); //单位毫秒
channel.queueDeclare(QUEUE_NAME, true, false, false, args);
```

如果不设置 TTL，表示消息不会过期，如果设置为 0，表示除非此时可以直接将消息投递到消费者，否则该消息会被立即丢弃。



2、对消息本身进行单独设置。

```java
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.deliveryMode(2);//持久化消息
builder.expiration("6000");//设置TTL=6000ms
AMQP.BasicProperties properties = builder.build();
channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, properties, "hello".getBytes());
```

如果两种方法同时使用，则以两者之间 TTL 较小的数值为准。

区别：

第一种设置队列属性的方法，一旦消息过期，就会从队列中抹去，而第二种方式中，即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费者之前判定的。

消息在队列中的生存时间一旦超过设置的 TTL 值时，就会变成死信（Dead Message）;

##### 设置队列的TTL
通过 `channel.queueDeclare` 方法中的 x-expires 参数可以控制队列被自动删除前处于未使用状态的时间。（未使用的意思是队列上没有任何消费者，队列也没有被重新声明，并且在过期时间段内也未调用过 `Basic.Get` 命令）。

RabbitMQ 会确保在过期时间到达后将队列删除，但不保障删除的动作有多及时。在 RabbitMQ 重启后，持久化的队列的过期时间会被重新计算。

```java
Map<String,Object> args = new HashMap<>();
args.put("x-expires", 180000);
channel.queueDeclare(QUEUE_NAME, true, false, false, args);
```

#### 死信队列

DLX（Dead-Letter-Exchange）,死信交换器。当消息在一个队列中变成死信（Dead Message）之后，它能被重新被发送到另一个交换器中，这个交换器就是DLX，绑定DLX的队列就称之为死信队列。

消息变成死信一般是由于以下几种情况：

- 消息被拒绝（Basic.Reject/Basic.Nack）,并且设置 requeue 参数为 false；
- 消息过期。
- 队列达到最大长度。

 当队列中存在死信时，RabbitMQ 就会自动地将这个消息重新发布到设置的 DLX 上去，进而被路由到另一个队列，即死信队列。通过在 `channel.queueDeclare` 方法中设置 x-dead-letter-exchange 参数来为这个队列添加 DLX。

```java
// 创建DLX：dlx_exchange
channel.exchangeDeclare("dlx_exchange", "direct");
Map<String,Object> args = new HashMap<>();
args.put("x-dead-letter-exchange", "dlx_exchange");
//也可为这个DLX指定路由键，如果没有特殊指定，则使用原队列的路由键
args.put("x-dead-letter-routing-key", "dlx_routing_key");
// 为队列添加DLX
channel.queueDeclare("myqueue", false,false,false,args);
```

#### 延迟队列
延迟队列存储的对象是对应的延迟消息。所谓“延迟消息”是指当消息被发送以后，并不让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。
RabbitMQ 本身没有直接支持延迟队列的功能，但是可以通过 DLX（死信交换器）和 TTL（过期时间）模拟出延迟队列的功能。

![延迟队列](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211028063219.png)

[【RabbitMQ】一文带你搞定RabbitMQ延迟队列 - 弗兰克的猫 - 博客园 (cnblogs.com)](https://www.cnblogs.com/mfrank/p/11260355.html)

#### 优先级队列

优先级队列：具有高优先级的队列具有高的优先权，优先级高的消息具备优先被消费的特权。
可以通过设置队列的 x-max-priority 参数来实现。

```java
// 配置 队列的最大优先级
Map<String,Object> args = new HashMap<>();
args.put("x-max-priority", 10);
channel.queueDeclare("myqueue", true,false,false,args);
```
配置好队列的最大优先级之后，需要在发送时在消息中设置消息当前的优先级。
```java
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.priority(5);//设置消息的优先级为5
AMQP.BasicProperties properties = builder.build();
channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, properties, "hello".getBytes());
```
上面代码中，设置消息的优先级为5。**默认最低为 0，最高为队列设置的最大优先级**。优先级高的消息可以被优先消费，这个也是有前提的：如果消费者的消费速度大于生产者的速度，此时对消息设置优先级就没什么实际意义。