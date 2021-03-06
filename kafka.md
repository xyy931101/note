# Kafka拓扑图

![](D:\workspace\note\image\kafka拓扑图.png)

# 相关概念

```
1.producer：
　　消息生产者，发布消息到 kafka 集群的终端或服务。
2.broker：
　　kafka 集群中包含的服务器。
3.topic：
　　每条发布到 kafka 集群的消息属于的类别，即 kafka 是面向 topic 的。
4.partition：
　　partition 是物理上的概念，每个 topic 包含一个或多个 partition。kafka 分配的单位是 partition。
5.consumer：
　　从 kafka 集群中消费消息的终端或服务。
6.Consumer group：
　　high-level consumer API 中，每个 consumer 都属于一个 consumer group，每条消息只能被 consumer group 中的一个 Consumer 消费，但可以被多个 consumer group 消费。
7.replica：
　　partition 的副本，保障 partition 的高可用。
8.leader：
　　replica 中的一个角色， producer 和 consumer 只跟 leader 交互。
9.follower：
　　replica 中的一个角色，从 leader 中复制数据。
10.controller：
　　kafka 集群中的其中一个服务器，用来进行 leader election 以及 各种 failover。
12.zookeeper：
　　kafka 通过 zookeeper 来存储集群的 meta 信息。
```