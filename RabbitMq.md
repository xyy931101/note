## RabbitMQ 简介

### 介绍

　　RabbitMQ 是一个由 erlang 开发的基于 AMQP（Advanced Message Queue）协议的开源实现。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面都非常的优秀，是当前最主流的消息中间件之一。RabbitMQ 官网：[http://www.rabbitmq.com](http://www.rabbitmq.com/)

### AMQP

　　AMQP 是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，同样，消息使用者也不用知道发送者的存在。AMQP 的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。

### 系统架构

![](\image\rabbitmq\系统架构.png)

消息队列的使用过程大概如下：

客户端连接到消息队列服务器，打开一个 channel；
客户端声明一个 exchange，并设置相关属性；
客户端声明一个 queue，并设置相关属性；
客户端使用 routing key，在 exchange 和 queue 之间建立好绑定关系；
客户端投递消息到 exchange，exchange 接收到消息后，就根据消息的 key 和已经设置的 binding，进行消息路由，将消息投递到一个或多个队列里。
如上图所示：AMQP 里主要说两个组件，Exchange 和 Queue。绿色的X就是 Exchange ，红色的是 Queue ，这两者都在 Server 端，又称作 Broker，这部分是 RabbitMQ 实现的，而蓝色的则是客户端，通常有 Producer 和 Consumer 两种类型。

### 几个概念

**Producer**： 为 生产者，数据的发送方；
**Consumer**：为消费者，数据的接收方；
**Exchange**：消息交换机，它指定消息按什么规则，路由到哪个队列；
**Queue**：消息队列载体，每个消息都会被投入到一个或多个队列；
**Binding**：绑定，它的作用就是把 exchange 和 queue 按照路由规则绑定起来；
**Routing** Key：路由关键字，exchange 根据这个关键字进行消息投递；
**vhost**：虚拟主机，一个 broker 里可以开设多个 vhost，用作不同用户的权限分离；
**channel**：消息通道，在客户端的每个连接里，可建立多个 channel，每个 channel 代表一个会话任务。

## 四种 Exchange 模式

　　AMQP 协议中的核心思想就是：生产者和消费者隔离，生产者从不直接将消息发送给队列。生产者通常不知道是否一个消息会被发送到队列中，只是将消息发送到一个交换机。先由 Exchange 来接收，然后 Exchange 按照特定的策略转发到 Queue 进行存储。同理，消费者也是如此。Exchange 就类似于一个交换机，转发各个消息分发到相应的队列中。

　　RabbitMQ 提供了四种 Exchange 模式：fanout、direct、topic、header. 由于 header 模式在实际使用中较少，因此本节只对前三种模式进行比较。

### Fanout Exchange

![](\image\rabbitmq\Fanout.png)

​		所有发送到 Fanout Exchange 的消息都会被转发到与该 Exchange 绑定（Binding）的所有 Queue 上。

​		Fanout Exchange 不需要处理 RouteKey，只需要简单的将队列绑定到 exchange 上，这样发送到 exchange 的消息都会被转发到与该交换机绑定的所有队列上。类似子网广播，每台子网内的主机都获得了一份复制的消息。

所以，**Fanout Exchange 转发消息是最快的。**

### Direct Exchange

![](\image\rabbitmq\Direct.png)

所有发送到 Direct Exchange 的消息被转发到 RouteKey 中指定的 Queue.

Direct 模式可以使用 RabbitMQ 自带的 Exchange：default Exchange ，因此不需要将 Exchange 进行任何绑定（binding）操作 。消息传递时，RouteKey 必须完全匹配，才会被队列接收，否则该消息会被抛弃。

### Topic Exchange

![](\image\rabbitmq\Topic.png)

​		所有发送到 Topic Exchange 的消息被转发到所有关心 RouteKey 中指定 Topic 的 Queue 上，Exchange 将 RouteKey 和某 Topic 进行模糊匹配。此时队列需要绑定一个 Topic，可以使用通配符进行模糊匹配，符号#匹配一个或多个词，符号*匹配不多不少一个词。因此log.#能够匹配到log.info.oa，但是log.* 只会匹配到log.error.

所以，**Topic Exchange 使用非常灵活。**

## 消息的可靠性

### 持久化

- exchange要持久化
- queue要持久化
- messgae持久化

### 生产方确认机制

消息的可靠性投递(confirm确认模式,return退回模式)生产者配置

**confirm消息确认机制:**

​		消息的确认，是指生产者投递消息后，如果Broker收到消息，则会给我们生产这一个应答。

​		生产者进行接收应答，用来确定这条消息是否正常的发送到Broker，这种方式也是消息的可靠性投递的核心保障。

​	**配置**

```java
spring:
  rabbitmq:
    addresses: 8.129.216.97:5672
    username: guest
    password: guest
    virtual-host: /
    publisher-returns: true #开启消息确认
    publisher-confirms: true #开启消息确认机制
    listener:
        simple:
          #acknowledge-mode: manual #设置确认模式手工确认
          concurrency: 3 #消费者最小数量
          max-concurrency: 10 # 消费者最大数量
        retry:
        	####开启消费者重试
        	enabled: true
        	####最大重试次数（默认无数次）
        	max-attempts: 5
        	####重试间隔次数
        	initial-interval: 3000
```

**生产者代码**

```
@Autowired
RabbitTemplate rabbitTemplate;

@Test
public void testSendMessage(){
//
rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
@Override
public void confirm(CorrelationData correlationData, boolean b, String s) {
System.out.println("消息确认模式。。。。");
}
});

rabbitTemplate.convertAndSend("test", "消息确认模式的test");
}
```



#### 消息回退机制（消息由交换机发送到队列时确认）：

​		1.开启回退机制（配置文件中开启如上确认机制）

​		配置如上面一样，生产者代码

```
    @Test
    public void testReturnMessage(){
        //设置手工确认
        rabbitTemplate.setMandatory(true);

        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            @Override
            public void returnedMessage(Message message, int i, String s, String s1, String s2) {
                System.out.println("回退机制。。。。");
            }
        });
        rabbitTemplate.convertAndSend("test", "消息回退机制的test");
    }
```



### 消费方ACK

1. 自动确认机制(默认acknowledge=“none”）

2. 手动确认机制(acknowledge = "manual")

3. 根据异常情况确认(acknowledge = "auto")

   ```java
   @Component
   @EnableRabbit
   public class MqListener implements ChannelAwareMessageListener {
       //Ack手动确认时要继承该类
   
       @Override
       @RabbitListener(queues = "testMq")
       public void onMessage(Message message, Channel channel) throws Exception {
           System.out.println("接受消息" + message.getBody().toString());
           long deliveryTag = message.getMessageProperties().getDeliveryTag();
           try {
               //业务消息
               Thread.sleep(1000);
               channel.basicAck(deliveryTag, true);
           }catch (Exception e){
               channel.basicNack(deliveryTag, true, true);
           }
       }
   }
   
   ```



## TTL队列/消息

- RabbitMQ支持消息的过期时间，在消息发送时可以进行指定

- RabbitMQ支持队列的过期时间，从消息入队列开始计算，只要超过队列的超时时间配置，那么消息会自动清除

## 死信队列

1. 死信队列：DLX

2. 利用DLX，当消息在一个队列中变成死信之后，它能被重新publish到另一个Exchange，这个Exchange就是DLX

3. 消息变成死信有以下几种情况

   - 消息被拒绝(basic.reject/basic.nack) ,并且requeue = false
   - 消息TTL过期
   - 队列达到最大长度

4. DLX也是一个正常的Exchange,和一般的Exchage没有区别，他能在任何队列上被指定，实际上就是设置某个队列的属性。

5. 当这个队列中有死信时，RabbitMQ就会自动的将这个消息重新发布到设置的Exchage上去，进而被路由到另一个队列

6. 可以监听这个队列中消息做相应的处理。

7. 设置死信队列，首先需要设置死信队列的exchage和queue，然后进行绑定

   (1） Exchage ： dlx.exchage

   (2) Queue : dlx.queue

   (3)RoutingKey ： #

   (4)在队列加上一个参数 ： **arguments.put("x-dead-letter-exchage","dlx.exchage");**

## 延迟队列

### 死信队列+普通队列实现

普通队列不设置消息的消费者并且设置TTL消息过期时间，死信交换机通过路由key绑定到普通交换机上，死信交换机绑定消费者，当普通队列上的消息过期时，数据就会进入死信队列，完成消息的延迟消费



## 消息补偿机制

![](\image\rabbitmq\消息补偿.png)

## RabbitMQ两种集群

#### 普通集群

什么是普通集群呢？ 就是在多个联通的服务器上安装不同的RabbitMQ的服务，这些服务器上的RabbitMQ服务组成一个个节点，通过RabbitMQ内部提供的命令或者配置来构建集群，形成了RabbitMQ的普通集群模式

![](\image\rabbitmq\默认集群.png)

- 当用户向服务注册一个队列，该队列会随机保存到某一个服务节点上，然后将对应的元数据同步到各个不同的服务节点上
- RabbitMQ的普通集群模式中，每个RabbitMQ都保存有相同的元数据
- 用户只需要链接到任一一个服务节点中，就可以监听消费到对应队列上的消息数据
- 但是RabbitMQ的实际数据却不是保存在每个RabbitMQ的服务节点中，这就意味着用户可能联系的是RabbitMQ服务节点C，但是C上并没有对应的实际数据，也就是说RabbitMQ服务节点C，并不能提供消息供用户来消费，那么RabbitMQ的普通集群模式如何解决这个问题呢？
- RabbitMQ服务节点C发现自己本服务节点并没有对应的实际数据后，因为每个服务节点上都会保存相同的元数据，所以服务节点C会根据元数据，向服务节点B（该服务节点上有实际数据可供消费）请求实际数据，然后提供给用户进行消费
- 这样给用户的感觉就是，在RabbitMQ的普通集群模式中，用户连接任一服务节点都可以消费到消息
   -普通集群模式的优点：提高消费的吞吐量

普通集群模式的原理比较简单，但是并不能真正意义上的实现高可用，他也存在以下的以下缺点：

1. 为了请求RabbitMQ的实际数据以提供给用户，可能会在RabbitMQ内部服务节点之间进行频繁的进行数据交互，这样的交互比较耗费资源
2. 当其中一个RabbitMQ的服务节点宕机了，那么该节点上的实际数据就会丢失，用户再次请求时，就会请求不到数据，系统的功能就会出现异常

#### 镜像集群模式

镜像队列是基于普通的集群模式的，然后再添加一些策略，所以还是得先配置普通集群，然后才能设置镜像队列。镜像队列存在于多个节点。要实现镜像模式，需要先搭建一个普通集群模式，在这个模式的基础上再配置镜像模式以实现高可用。

![](\image\rabbitmq\镜像队列的结构.png)

镜像队列基本上就是一个特殊的BackingQueue，它内部包裹了一个普通的BackingQueue做本地消息持久化处理，在此基础上增加了将消息和ack复制到所有镜像的功能。所有对mirror_queue_master的操作，会通过可靠组播GM的方式同步到各slave节点。GM负责消息的广播，mirror_queue_slave负责回调处理，而master上的回调处理是由coordinator负责完成。mirror_queue_slave中包含了普通的BackingQueue进行消息的存储，master节点中BackingQueue包含在mirror_queue_master中由AMQQueue进行调用。

消息的发布（除了Basic.Publish之外）与消费都是通过master节点完成。master节点对消息进行处理的同时将消息的处理动作通过GM广播给所有的slave节点，slave节点的GM收到消息后，通过回调交由mirror_queue_slave进行实际的处理。

对于Basic.Publish，消息同时发送到master和所有slave上，如果此时master宕掉了，消息还发送slave上，这样当slave提升为master的时候消息也不会丢失。

##### GM（Guarenteed Multicast）

GM模块实现的一种可靠的组播通讯协议，该协议能够保证组播消息的原子性，即保证组中活着的节点要么都收到消息要么都收不到。

它的实现大致如下：

将所有的节点形成一个循环链表，每个节点都会监控位于自己左右两边的节点，当有节点新增时，相邻的节点保证当前广播的消息会复制到新的节点上；当有节点失效时，相邻的节点会接管保证本次广播的消息会复制到所有的节点。在master节点和slave节点上的这些gm形成一个group，group（gm_group）的信息会记录在mnesia中。不同的镜像队列形成不同的group。消息从master节点对于的gm发出后，顺着链表依次传送到所有的节点，由于所有节点组成一个循环链表，master节点对应的gm最终会收到自己发送的消息，这个时候master节点就知道消息已经复制到所有的slave节点了。

##### 新增节点

新节点的加入过程如下图所示：

![](\image\rabbitmq\新增镜像节点.png)

每当一个节点加入或者重新加入（例如从网络分区中恢复过来）镜像队列，之前保存的队列内容会被清空。

##### 节点的失效

如果某个slave失效了，系统处理做些记录外几乎啥都不做。master依旧是master，客户端不需要采取任何行动，或者被通知slave失效。 如果master失效了，那么slave中的一个必须被选中为master。被选中作为新的master的slave通常是最老的那个，因为最老的slave与前任master之间的同步状态应该是最好的。然而，需要注意的是，如果存在没有任何一个slave与master完全同步的情况，那么前任master中未被同步的消息将会丢失。

##### 消息的同步

将新节点加入已存在的镜像队列是，默认情况下ha-sync-mode=manual，镜像队列中的消息不会主动同步到新节点，除非显式调用同步命令。当调用同步命令后，队列开始阻塞，无法对其进行操作，直到同步完毕。

当ha-sync-mode=automatic时，新加入节点时会默认同步已知的镜像队列。由于同步过程的限制，所以不建议在生产的消费队列中操作。