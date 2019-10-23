+++
author = "陈 sir"
categories = ["mq"]
tags = ["kafka"]
date = "2019-10-23"
description = "kafka 深入理解"
featured = "main/title.png"
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "kafka 深入理解"
type = "post"

+++


# kafka 服务端
## 协议

在2.0版本中支持43中协议，每种协议包含相同的请求头和不同的请求体。

### 请求头

请求头包含四个字段

| 字段          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| api_key       | api标识 比如 produce fetch等。说明需要执行的动作             |
| api_version   | api 版本号                                                   |
| corelation_id | 由客户端生成的请求id（全局唯一），服务端（broker）在处理完请求后也会把相同的id写回 |
| client_id     | 客户端id                                                     |



### produce request 结构

{{< fancybox path="/img/main" file="pr.png" caption="produce request 结构" gallery="produce request 结构" >}}

{{< fancybox path="/img/main" file="prinfo.png" caption="produce request 字段说明" gallery="produce request 字段说明" >}}

消息处理过程

sender线程从消息累加器中获取消息<Partition,<ProduceBatch>>，将消息转换为**<node,List<ProduceRequest>>**的格式。

如果acks 的值为非0，客户端将异步等待服务端的相应（ProduceResponse）

### produce response 结构

{{< fancybox path="/img/main" file="produce-response.png" caption="响应结构" gallery="响应结构" >}}

{{< fancybox path="/img/main" file="producerep.png" caption="响应结构说明" gallery="响应结构说明" >}}

### Fetch 请求/响应的结构

{{< fancybox path="/img/main" file="fetchreq.png" caption="" gallery="" >}}

{{< fancybox path="/img/main" file="fetchreqinfo1.png" caption="" gallery="" >}}

{{< fancybox path="/img/main" file="fetchreqinfo2.png" caption="" gallery="" >}}

{{< fancybox path="/img/main" file="fetchrep.png" caption="" gallery="" >}}

## 时间轮（timingwheel）

时间轮是一个存储定时任务的环形队列，底层采用数组实现。数组中的每个元素存放一个定时任务列表（TimerTaskList）。TimerTaskList是一个双向链表，链表中的每个元素表示一个任务项（TimerTaskEntry）封装了真正的任务（TimerTask）

### 相关概念

- tickms 基本时间跨度
- wheelSize 时间格数
- interval 总体时间跨度 = tickms x wheelSize
- 层级，时间轮是分层级的，每一层的时间格数不变，但是上一层的基本时间跨度=下一层的总时间跨度

{{< fancybox path="/img/main" file="twheel.png" caption="" gallery="" >}}

{{< fancybox path="/img/main" file="multiwheel.png" caption="" gallery="" >}}

### 延迟操作

当acks设置为-1时，会生成一个延时操作，只有消息同步到所有分区后才发送返回。

## 控制器

### 选举

控制器的选举，通过向zookeeper 中创建/controller 临时节点来进行竞争。成功创建的节点即为控制器。

/controller 中包含三个变量

- version 固定为1
- broker_id 成功竞选为控制器的broker的id
- timestamp 竞选成功的时间

/controller_epoch 记录控制器变化的次数。

**控制器的职责**

1. 监听分区变化，通过在zookeeper中注册相关的handler来实现
2. 监听主题变化，通过在zookeeper中注册相关的handler来实现
3. 监听broker变化，在zookeeper中/brokers/ids节点添加BrokerChangehandler 实现
4. 读取集群相关信息进行管理
5. 启动并管理分区状态机与分区状态机
6. 更新集群元数据



### 关闭流程

{{< fancybox path="/img/main" file="shutdown.png" caption="" gallery="" >}}

### 分区leader 选举

分区的选举由控制器实施，策略为OfflinePartitionLeaderElectionStrategy，选取AR 中，第一个存活的副本。