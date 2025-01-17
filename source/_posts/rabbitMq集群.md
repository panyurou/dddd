---
title: rabbitMq集群
date: 2022-09-21 23:38:13
tags: 集群
categories: RabbitMQ
---

# 普通集群模式

- **queue的消息**，只会放在**一个rabbtimq实例**上。**每个实例都同步queue的结构和queue消息的真正位置**。

- 消费的时候，实际上如果连接到了另外一个实例，那么那个实例会从queue所在实例上拉取数据过来。

这种集群模式的消息可靠性不是很高。因为如果其中有个节点服务宕机了，那这个节点上的数据就无法消费了，需要等到这个节点服务恢复后才能消费。

# 镜像集群模式

- 这种模式是在普通集群模式基础上的一种增强方案，这也就是RabbitMQ的官方HA高可用方案。

- 这种模式会在镜像节点中间主动进行消息同步，而不是在客户端拉取消息时临时同步。这种模式的消息可靠性更高，因为每个节点上都存着全量的消息。
- 他的弊端也是明显的，集群内部的网络带宽会被这种同步通讯大量的消耗，进而降低整个集群的性能。这种模式下，队列数量最好不要过多。

