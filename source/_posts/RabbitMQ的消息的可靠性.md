---
title: RabbitMQ的消息零丢失方案
date: 2022-09-12 12:37:48
tags: 
- MQ
- 消息可靠性
categories: RabbitMQ
---

# RabbitMQ如何保证消息的可靠性？

## 哪些环节会有丢消息的可能？

<img src="https://tva1.sinaimg.cn/large/e6c9d24ely1h63pq7alokj219l0u0gpa.jpg" style="zoom:50%;" />

### 普通消息处理流程

1. 消息生成者发送消息
2. MQ收到消息，将消息进行存储（持久化到硬盘）。
3. 返回ACK给生产者
4. MQ push 消息给对应的消费者，然后等待消费者返回ACK
5. 如果消息消费者在指定时间内成功返回ack，那么MQ认为消息消费成功，在存储中删除消息，即执行第6步；如果MQ在指定时间内没有收到ACK，则认为消息消费失败，会尝试重新push消息,重复执行4、5、6步骤
6. MQ删除消息

### 丢消息的场景

* 1，2，4三个场景，都是跨网络的，而跨网络就肯定会有丢消息的可能。
* 关于3这个环节，通常MQ存盘时都会先写入操作系统的缓存page cache中，然后再由操作系统异步的将消息写入硬盘。这个中间有个时间差，就可能会造成消息丢失。如果服务挂了，缓存中还没有来得及写入硬盘的消息就会丢失。这也是任何用户态的应用程序无法避免的。
* 对于任何MQ产品，都应该从这四个方面来考虑数据的安全性。那我们看看用RabbitMQ时要如何解决这个问题。

## RabbitMQ消息零丢失方案?

### 1. 生产者保证消息正确发送到RibbitMQ?

- **方案一：**可以使用生产者确认机制。

  通过多次确认的方式，保证生产者的消息能够正确的发送到RabbitMQ中。RabbitMQ的生产者确认机制分为同步确认和异步确认。

  - 同步确认主要是通过在生产者端使用Channel.waitForConfirmsOrDie()指定一个等待确认的完成时间。

  - 异步确认机制则是通过在生产者端注入两个回调确认函数。第一个函数是在生产者发送消息时调用，第二个函数则是生产者收到Broker的消息确认请求时调用。两个函数需要通过sequenceNumber自行完成消息的前后对应。sequenceNumber的生成方式需要通过channel的序列获取。

    ```java
    channel.addConfirmListener(ConfirmCallback var1,ConfirmCallback var2)
    ```

- **方案二：**手动事务

  - 手动事务机制主要有几个关键的方法： channel.txSelect() 开启事务；channel.txCommit() 提交事务； channel.txRollback() 回滚事务。用这几个方法来进行事务管理。比如在创建订单后开启事务，减库存成功后提交事务，失败时回滚。

  - 这种方式需要手动控制事务逻辑，并且手动事务会对channel产生阻塞，造成吞吐量下降

### 2. **RabbitMQ 主从消息同步时不丢消息**?

- 不使用普通集群模式， 使用镜像集群模式
- 给包含重要消息的队列建立一个远端备份。

> - 普通集群模式
>   - **queue的消息**，只会放在**一个rabbtimq实例**上。**每个实例都同步queue的结构和queue消息的真正位置**。
>   - 消费的时候，实际上如果连接到了另外一个实例，那么那个实例会从queue所在实例上拉取数据过来。
>   - 这种集群模式的消息可靠性不是很高。因为如果其中有个节点服务宕机了，那这个节点上的数据就无法消费了，需要等到这个节点服务恢复后才能消费。
>
> - 镜像集群模式
>
>   - 这种模式是在普通集群模式基础上的一种增强方案，这也就是RabbitMQ的官方HA高可用方案。
>
>   - 这种模式会在镜像节点中间主动进行消息同步，而不是在客户端拉取消息时临时同步。这种模式的消息可靠性更高，因为每个节点上都存着全量的消息。
>   - 他的弊端也是明显的，集群内部的网络带宽会被这种同步通讯大量的消耗，进而降低整个集群的性能。这种模式下，队列数量最好不要过多。



### 3. RabbitMQ消息存盘不丢消息?

对于Classic经典队列，直接将队列声明成为持久化队列即可。

而新增的Quorum（[ˈkwɔrəm] 仲裁）队列和Stream队列，都是明显的持久化队列，能更好的保证服务端消息不会丢失。

> Quorum队列实现了持久化，多备份的FIFO队列，主要就是针对RabbitMQ的镜像模式设计的。大部分功能都是在Classic队列基础上做减法，比如不支持是非持久化的内存队列。某些功能（例如有害消息处理）是特定于仲裁队列的。
>
> Stream队列是RabbitMQ自3.9.0版本开始引入的一种新的数据队列类型，更适合于消费者多，读消息非常频繁的场景。

> 毒消息是指消息一直不能被消费者正常消费(可能是由于消费者失败或者消费逻辑有问题等)，就会导致消息不断的重新入队，这样这些消息就成为了毒消息。这些读消息应该有保障机制进行标记并及时删除。Quorum队列会持续跟踪消息的失败投递尝试次数，并记录在"x-delivery-count"这样一个头部参数中。然后，就可以通过设置 Delivery limit参数来定制一个毒消息的删除策略。当消息的重复投递次数超过了Delivery limit参数阈值时，RabbitMQ就会删除这些毒消息。当然，如果配置了死信队列的话，就会进入对应的死信队列。

### 4. **RabbitMQ消费者不丢失消息**?

- RabbitMQ在消费消息时可以指定是自动应答，还是手动应答。将RabbitMQ的应答模式设定为手动应答可以提高消息消费的可靠性。

- 设置了手动确认，则需要在业务处理成成功后，手动调用`channer.basicAck()`,如果出现异常则调用`channer.basicNAck()`,设置消息是重新返回队列，还是直接丢掉。

**最后，任何用户态的应用程序都无法保证绝对的数据安全，所以，备份与恢复的方案也需要考虑到。**



## Consumer Ack 消费端收到消息后的确认方式

1. 有三种确认方式

   * 自动确认 acknowledge = "none"。 消息一旦到达consumer就会被确认，并将对应的message 从消息缓存中移除，实际场景中，很可能消息被收到，但是处理业务时异常，这种确认机制下，消息就会丢失。
   * 手动确认acknowledge = "manual"。设置了手动确认，则需要在业务处理成成功后，手动调用`channer.basicAck()`,如果出现异常则调用`channer.basicNAck()`,设置消息是重新返回队列，还是直接丢掉。
   * 根据异常情况确认 acknowledge = "auto"

   
