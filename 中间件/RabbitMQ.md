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

默认情况下，rabbitmq服务的用户名和密码都是“guest”，但这个默认账户只能通过本地网络访问，远程网络访问受限。解决办法，

1、修改配置文件或者命令，去除guest账户限制。（不推荐）相关配置详解：https://www.rabbitmq.com/access-control.html,  https://www.rabbitmq.com/configure.html,

![image-20200930062741810](http://image.yzqfrist.com/20200930062744.png)

![image-20200930062334757](http://image.yzqfrist.com/20200930062343.png)

2、添加一个新的账号。相关配置详解：[https://www.rabbitmq.com/configure.html](https://www.rabbitmq.com/configure.html),  	

```
rabbitmqctl add_user root 123
rabbitmqctl set_permissions -p / root ".*" ".*" ".*"
rabbitmqctl set_user_tags root administrator
```

设置的权限语句，默认default_vhost 为“/”,三个权限分别为configure，read，write。

![image-20200930063341114](http://image.yzqfrist.com/20200930063342.png)



### 交换器类型

常用的类型有四种，fanout、direct、topic、headers。

* fanout 广播: 把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中。

* direct :把消息路由到BindingKey和RoutingKey完全匹配的队列中。

* topic 在direct交换器的匹配规则上进行扩展，将消息路由到BindingKey和RoutingKey相匹配的队列中。

  匹配规则:

  - RoutingKey 为一个点号“.”分隔的字符串，（被点号分隔开的每一段独立的字符串成为一个单词）。如：“com.rabbitmq.client”
  - BindingKey和RoutingKe一样也是点号“.”分隔的字符串。
  - BindingKey中也可以存在两种特殊字符串“*”和“#”，用于做模糊匹配。其中，“\*”用于匹配一个单词，“#”用于匹配多个单词（也可以是0个）；

* headers：headers类型的交换器不依赖路由键的匹配规则来路由消息，而是根据发送的消息的消息内容中的headers属性进行匹配。（性能很差，不实用）

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
* type：交换器的类型。常见的有fanout、direct、topic
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

##### 2、拉模式：调用Basic.get消费

通过channel.basicGet方法可以单条地获取消息。

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

为了保证消息从队列可靠地到达消费者，rabbit提供了消息确认机制（message acknowledgement）。消费者在订阅队列时，可以指定autoAck参数，当autoAck参数为false时，rabbitz会等待消费者显示地回复确认信号后才从内存（或者磁盘）中移去消息（实质上是先打上删除标记，之后再删除）。当autoAck等于true时，RabbitMQ会自动把发送出去的消息置为确认，然后从内存（或者磁盘）中删除，而不管消费者是否真正地消费到了这些消息。

那么当autoAck为false时，如果RabbitMQ一直没有收到消费者的确认信号，并且消费此消息的消费者已经断开连接，则RabbitMQ会安排该消息重新进入队列，等待投递给下一个消费者，当然也可能是原来那个消费者。

##### 消息拒绝
###### basicReject：一次拒绝一条消息

```java
  void basicReject(long deliveryTag, boolean requeue) throws IOException;
```

参数说明：

- deliveryTag：可以看作消息的编号，是一个64位的长整数值，最大值是9223372036854775807。
- requeue：如果设置为true，则RabbitMQ会重新将这条消息存入队列，以便可以发送给下一个订阅的消费者；如果设置为false，则RabbitMQ立即会把消息从队列中移除，而不会把它发送给新的消费者。

###### basicNack：批量拒绝消息

```java
void basicNack(long deliveryTag, boolean multiple, boolean requeue) throws IOException;
```

参数说明：

- multiple：设置为false，表示拒绝编号为deliveryTag的这一条消息，此时和basicReject一样；设置为true，则表示拒绝deliveryTag编号之前的所有未被当前消费者确认的消息。

> basicReject basicNack 的requeue参数设置为false，可以启用“死信队列”的功能。死信队列可以通过检测被拒绝或者未送达的消息来追踪问题。

##### 可重入队列
###### basicRecover：请求RabbitMQ重新发送还未被确认的消息

```java
Basic.RecoverOk basicRecover() throws IOException;
Basic.RecoverOk basicRecover(boolean requeue) throws IOException;
```
参数说明：

- requeue：设置为true，则未被确认的消息会被重新加入到队列中，这样对于同一条消息，可能会被分配给与之前不同的消费者。如果requeue参数设置为false，那么同一个消息会被分配给与之前相同的消费者。默认requeue为true。



#### 关闭连接
```java
channel.close();
conn.close();	
```

Connection和Channel 所具备的生命周期如下：

1. open: 开启状态，代表当前对象可用。
2. Closing：正在关闭状态。当前对象被显式地通知调用关闭方法（shutdown），这样就产生了一个关闭请求让其内部对象进行相应的操作，并等待这些关闭操作的完成。
3. Closed：已经关闭状态。当前对象已经接受到所有内部对象已完成关闭动作的通知，并且其也关闭了自身。

与关闭相关的其他方法：

```java
addShutdownListener(ShutdownListener listener)
removeShutdownListener(ShutdownListener listener)
```

在Connection和Channel的状态转变为Closed的时候会调用ShutdownListener。而且如果将一个ShutdownListener注册到一个已经处于关闭状态的对象（Connection和Channel对象），会立刻调用ShutdownListener。

```java
getCloseReason:得到对象关闭的原因。
isOpen：检测对象当前是否处于开启状态。
close(int closeCode,String closeMessage):显式地通知当前对象执行关闭操作。
```

## RabbitMQ进阶
### 消息何去何从
#### mandatory参数

**当参数mandatory参数设为true时，交换器无法根据自身的类型和路由键找到一个符合条件的队列时，那么RabbitMQ会调用Basic.Return命令将消息返回给生产者。当mandatory参数设置为false时，出现上述情形，则消息直接被丢弃**。

生产者通过调用channel.addReturnListener来添加ReturnListener监听器来获取到没有被正确路由到合适队列的消息。

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
生产者在发送消息时候如果不设置mandatory参数，那么消息在未被路由的情况下将会丢失；如果添加了mandatory参数，那么需要添加ReturnListener的编程逻辑，这样会让生产者的编程变得复杂。如果既不想复杂化生产者的编程逻辑，又不想消息丢失，可以使用备份交换器，可以将未被路由的消息存放在RabbitMQ中，再在需要的时候去处理这些消息。
##### 实现方式
1、通过在声明交换器（channel.exchangeDeclare方法）添加alternate-exchange参数来实现。（优先级别更高）。
```java
Map<String,Object> args = new HashMap<>();
args.put("alternate-exchange", "myAe");
channel.exchangeDeclare("normalExchange", "direct", true, false, args);//将myAe设置为normalExchange的备份交换器
channel.exchangeDeclare("myAe","fanout",true,false,null);
```

为了方便使用，备份交换器建议设置为fanout类型。原因：**消息被重新发送到备份交互器时的路由键和从生产者发出的路由键是一样的**。

AE的几种特殊情况：

- 如果设置的备份交换器不存在，客户端和RabbitMQ服务端都不会出现异常，此时消息会丢失。
-  如果备份交换器没有绑定任何队列，客户端和RabbitMQ服务端都不会出现异常，此时消息会丢失。
- 如果备份交换器没有任何匹配的队列，客户端和RabbitMQ服务端都不会出现异常，此时消息会丢失。
- 如果备份交换器和mandatory参数一起使用，那么mandatory参数无效。

2、通过策略（Policy）实现。

#### 过期时间（TTL）
TTL（Time to Live）,RabbitMQ可以对消息和队列设置TTL。
##### 设置消息的TTL
设置消息的TTL的两种方法：
1、通过队列属性设置，队列中所有消息都具有相同的过期时间。

```java
Map<String,Object> args = new HashMap<>();
args.put("x-message-ttl", 6000); //单位毫秒
channel.queueDeclare(QUEUE_NAME, true, false, false, args);
```

如果不设置TTL，表示消息不会过期，如果设置为0，表示除非此时可以直接将消息投递到消费者，否则该消息会被立即丢弃。



2、对消息本身进行单独设置。

```java
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.deliveryMode(2);//持久化消息
builder.expiration("6000");//设置TTL=6000ms
AMQP.BasicProperties properties = builder.build();
channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, properties, "hello".getBytes());
```

如果两种方法同时使用，则以两者之间TTL较小的数值为准。

区别：

第一种设置队列属性的方法，一旦消息过期，就会从队列中抹去，而第二种方式中，即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费者之前判定的。

消息在队列中的生存时间一旦超过设置的TTL值时，就会变成死信（Dead Message）;

##### 设置队列的TTL
通过channel.queueDeclare方法中的x-expires参数可以控制队列被自动删除前处于未使用状态的时间。（未使用的意思是队列上没有任何消费者，队列也没有被重新声明，并且在过期时间段内也未调用过Basic.Get命令）。

RabbitMQ会确保在过期时间到达后将队列删除，但不保障删除的动作有多及时。在RabbitMQ重启后，持久化的队列的过期时间会被重新计算。

```java
Map<String,Object> args = new HashMap<>();
args.put("x-expires", 180000);
channel.queueDeclare(QUEUE_NAME, true, false, false, args);
```

#### 死信队列

DLX（Dead-Letter-Exchange）,死信交换器。当消息在一个队列中变成死信（Dead Message）之后，它能被重新被发送到另一个交换器中，这个交换器就是DLX，绑定DLX的队列就称之为死信队列。

消息变成死信一般是由于以下几种情况：

- 消息被拒绝（Basic.Reject/Basic.Nack）,并且设置requeue参数为false；
- 消息过期。
- 队列达到最大长度。

 当队列中存在死信时，RabbitMQ就会自动地将这个消息重新发布到设置的DLX上去，进而被路由到另一个队列，即死信队列。通过在channel.queueDeclare方法中设置x-dead-letter-exchange参数来为这个队列添加DLX。

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
RabbitMQ本身没有直接支持延迟队列的功能，但是可以通过DLX（死信交换器）和TTL（过期时间）模拟出延迟队列的功能。
[![image.png](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA/oAAAErCAYAAABuG/gCAAAgAElEQVR4AeydB5xcV33vf9PLzszO9l60aqteLau4N4wLBmxqKAFSCCR5eZCQlwReCnkhgRACgcSQQIAYDLYTG3dbyJZtybLVu7Ta1Wp7351t08t9n/+ZHWl3tbK2zWp253fs0czce8+553zvnbvnX49O0zQNLCRAAiRAAiRAAiRAAiRAAiRAAiQwRwSi0Sjq6+uxePFi6PX6OTpr+pyGRNPnWnOkJEACJEACJEACJEACJEACJJAyBIxGI3Q6Xcr0ZyF1hIL+QrqaHAsJkAAJkAAJkAAJkAAJkAAJzAMCIuA7nc550NP52UUK+vPzurHXJEACJEACJEACJEACJEACJDBvCUgEucfjASPJk3MJKegnhytbJQESIAESIAESIAESIAESIAESIIFrQoCC/jXBzpOSAAmQAAmQAAmQAAmQAAmQQPoSENd9t9vNGP0k3QIU9JMEls2SAAmQAAmQAAmQAAmQAAmQAAlMTEBc9oeGhibeya0zJkBBf8YI2QAJkAAJkAAJkAAJkAAJkAAJkMBUCcgSe4zRnyq1yR1PQX9ynHgUCZAACZAACZAACZAACZAACZDALBEQ1/2MjAy67s8Sz/HNUNAfT4TfSYAESIAESIAESIAESIAESIAEkkpALPli0WdJDgEK+snhylZJgARIgARIgARIgARIgARIgATegUAgEKDr/jvwmckuCvozoce6JEACJEACJEACJEACJEACJEAC0yJgtVrpuj8tclevREH/6ox4BAmQAAmQAAmQAAmQAAmQAAmQwCwSkBh9k8k0iy2yqdEEKOiPpsHPJEACJEACJEACJEACJEACJEACSSeQWF6PWfeTg5qCfnK4slUSIAESIAESIAESIAESIAESIIF3IGA0Gum6/w58ZrKLgv5M6LEuCZAACZAACZAACZAACZAACZDAlAmI677T6ZxyPVaYHAEK+pPjxKNIgARIgARIgARIgARIgARIgARmiYC47Hs8HmbdnyWe45uhoD+eCL+TAAmQAAmQAAmQAAmQAAmQAAkknYBY9VmSQ4CCfnK4slUSIAESIAESIAESIAESIAESIIErEBAh3+12M0b/CnxmupmC/kwJsj4JkAAJkAAJkAAJkAAJkAAJkMCUCIjr/uDg4JTq8ODJE6CgP3lWPJIESIAESIAESIAESIAESIAESGCWCESjUcbozxLL8c1Q0B9PhN9JgARIgARIgARIgARIgARIgASSSkBc9zMyMui6nyTKFPSTBJbNkgAJkAAJkAAJkAAJkAAJkAAJTExAXPcjkcjEO7l1xgQo6M8YIRsgARIgARIgARIgARIgARIgARKYKoFgMEjX/alCm+TxFPQnCYqHkQAJkAAJkAAJkAAJkAAJkAAJzB4Bq9VK1/3ZwzmmJQr6Y3DwCwmQAAmQAAmQAAmQAAmQAAmQQLIJSIy+yWRK9mnStn0K+ml76TlwEiABEiABEiABEiABEiABErg2BCRGf2hoiK77ScJPQT9JYNksCZAACZAACZAACZAACZAACZDAlQkYjUa67l8Zz4z2UNCfET5WJgESIAESIAESIAESIAESIAESmCoBcd13Op1TrcbjJ0mAgv4kQfEwEiABEiABEiABEiABEiABEiCB2SEgrvsej4eu+7OD87JWKOhfhoQbSIAESIAESIAESIAESIAESIAEkk1ArPosySFAQT85XNkqCZAACZAACZAACZAACZAACZDAFQiIkO92uxmjfwU+M91MQX+mBFmfBEiABEiABEiABEiABEiABEhgSgTEdX9wcHBKdXjw5AkYJ3/o7B0Z1YDH67w43BNCKDZ77bKlmRHIs+pxV6kV1xVYZtYQa5MACZAACZAACZAACZAACZDAVQhEo1EVo08X/quAmsbuayLoxzQNLzQFsLMtgGBMm0a3WSUZBErsBuRbDRT0kwGXbZIACZAACZAACZAACZAACVwkIMK9w+Gg6/5FIrP74ZoI+jKE/lAMfaEYgmLeZ0kJAnaDDt4Ir0dKXAx2ggRIgARIgARIgARIgAQWMAFx3Q+Hwwt4hNd2aIzRv7b8eXYSIAESIAESIAESIAESIAESSEsCwWCQy+sl6cpT0E8SWDZLAiRAAiRAAiRAAiRAAiRAAiRwZQJWq5Wu+1fGM6M9FPRnhI+VSYAESIAESIAESIAESIAESIAEpkpAYvRNJtNUq/H4SRKgoD9JUDyMBEiABEiABEiABEiABEiABEhgdghIjP7Q0BBd92cH52WtUNC/DAk3kAAJkAAJkAAJkAAJkAAJkAAJJJuAWPS5tF5yKFPQTw5XtkoCJEACJEACJEACJEACJEACJHAFAonl9a6wm5tnSOCaLa83w36zOgmQAAnMLwLRGLRQBIjF5le/2duZEzAaoDMbAZ1u5m2xBRJIdwJaFIh6gRiX5Eq7W0FvAgx2QJe+4os3rCEU07BQFsOORqNo7O6HLsMNnZ725yv9po06wGrUwayf2jwifX8pVyLJ7SRAAiSQBALRtl4EXzuJaOdAElpnkylLQK+DaXEhLO/aCJ2FCYdS9jqxY/OHQLAdOPf/AM/h+dNn9nR2CGSuAZb+GZCxeHbam2etRGIavndiEK+1BxFcIDYDidEPBoyw1vZSGf4O92OV04DPrHDg+gLLOxx1+S4K+pcz4RYSIAESmHUCsUE/wuc7KOjPOtkUb3BE+26JLpBZWYrjZvfSgEDEC/TuAzwn0mCwHOIYApFhoDJ9leUxDdjXFcLuziCC0YVi0wc06KEbCI651PwylkCr14S7y6NjN07iGwX9SUDiISRAAiQwYwKaBshfaXlnSR8Ccrl5zdPnenOkc0BAnqOx+GsOzsZTpBABue4Lxml9elxlGiEy/gKS86cHIs1qRTVtWlMJBkOk2Y3C4ZIACZAACZAACZAACZAACZAACSxsAhT0F/b15ehIgARIgARIgARIgARIgARIgATSjAAF/TS74BwuCZAACZAACZAACZAACZAACZDAwiZAQX9hX1+OjgRIgARIgARIgARIgARIgARIIM0IUNBPswvO4ZIACZAACZAACZAACZAACZAACSxsAhT0F/b15ehIgARIgARIgARIgARIgARIgATSjAAF/TS74BwuCZAACZAACZAACZAACZAACZDAwiZAQX9hX1+OjgRIgARIgARIgARIgARIgARIIM0IUNBPswvO4ZIACZAACZAACZAACZAACZAACSxsAhT0F/b15ehIgARIgARIgARIgARIgARIgATSjAAF/TS74BwuCZAACZAACZAACZAACZAACZDAwiZAQX9hX1+OjgRIgARIgARIgARIgARIgARIIM0IpLWgn23WI8+ih92ow3KXEdWZpqte/uvzzbi3zIZc69XR3VFixYpMI0y6qzbLA0iABEggNQnodTAUZcFQ6IbOZHjHPursFpjXVl46xmSAzmJUL0hdnQ76TDsMJTnQZ1gvHSfPSKNBtS/nSLygH3nOmo3q/Ia8zEt1+IkESIAE5jsBkxPIWgtkrY6PxOgAyj4MGMZNHOVRKI/fxEt2S133KsBaMJbC+GOlTmLKqjcC9gIg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)](https://www.cnblogs.com/mfrank/p/11260355.html) 
#### 优先级队列
优先级队列：具有高优先级的队列具有高的优先权，优先级高的消息具备优先被消费的特权。
可以通过设置队列的x-max-priority参数来实现。
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
上面代码中，设置消息的优先级为5。**默认最低为0，最高为队列设置的最大优先级**。优先级高的消息可以被优先消费，这个也是有前提的：如果消费者的消费速度大于生产者的速度，此时对消息设置优先级就没什么实际意义。