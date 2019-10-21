+++
author = "陈 sir"
categories = ["mq"]
tags = ["kafka"]
date = "2019-10-121"
description = ""
featured = "main/title.png"
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "kafka 概念介绍"
type = "post"

+++


# kafka 基础概念
## 基础概念

- **Producer**: 消息发送者
- **broker**： kafka节点
- consumer: 消费者
- topic： 主题，一组消息的集合，分布在多个分区里
- partition：分区， 一个分区只能属于一个主题。在存储曾铭，分区可以看做一个可追加log日志文件，每个消息在分区中表现为一条日志，而消息的位置，通过offset 来记录，offset 是消息在分区分区中的唯一标示，offset保证消息的顺序性，注意，offset 不跨分区，在不同的分区中，offset 可能相同。
- replica：副本，对分区的冗余，会尽量分布在不同的节点。一个分区会有leader 和 follower 两种角色的副本。
- AR（assignded-replica）：分区中的所有副本统称为AR
- ISR (in-sync replica)：与 leader 副本保持一定程度同步的副本成为 ISR， ISR 是AR 的子集。
- OSR (out-of-sync replica)：与leader 同步滞后达到一定程度的副本，这个一定程度可以通过参数进行设置。AR=ISR+OSR
- HW(high watermark 高水位)：指一个特殊的OFFSET，消费者，只能获取HW 这个offset 之前的消息，即便HW 之后有其他消息，对消费者而言也是不可见的。（与ISR 强相关）
- LSO(logstartoffset)：消息开始的初始offset，值为0
- LEO(logendoffset)：当前分区中，下一条待写入消息的offset。



{{< fancybox path="/img/main" file="offset-introduce.png" caption="offset" gallery="offset" >}}



## 消息的发送过程

producer-> 拦截器（非必须） -> 序列化器 -> 分区器- broker

{{< fancybox path="/img/main" file="producer-flow.png" caption="producer-flow" gallery="producer-flow" >}}



## 消息投递方式 P2P , PUB/SUB

kafka 通过消费者组和消费者来适配两种模式

如果所有的消费者头隶属于同一个消费者组，那么消息只会投递给其中一个消费者，相当于P2P模式

如果消费者隶属于不同的消费者组，那么消息会发送给所有的消费者组。适配PUB/SUB模式

## 消费位移（offset）

指消费者在kafka中的消费位置，注意这个角度跟消息的offset 不是一个意思。消息的offset 是指消息在kafka中的偏移量，而消费位移，是指消费者在分区中将消息消费到了哪一条。消费者在消费完消息之后需要提交消费位移。注意提交的消费位移的值，当前消费过的最后一条消息的offset+1,而不是当前消费了的消息的offset

> 旧的消费这的消费位移存储在zk中，而新添加的消费这的消费位移存储在kafka的内部主题 `_consumer_offsets` 中

## 消费位移的提交

kafka 默认是自动提交（客户端参数,enable.auto.commit），然后自动提交容易造成消息丢失。消费位移的提交并不是每消费一条消息就提交，而是定期提交，通过参数(`auto.commit.interval.ms`) 指定提交间隔。默认是5秒

## 再均衡

只一个一个分区的的所有权，从一个消费者分配給另一个消费者的过程。再均衡存在以下影响

1. 再均衡期间，消费组不可用。
2. 可能存在重复消费。例如一个消费者A消费完后，还没来得及提交消费位移，就发生了再均衡。而消费者A被分配到了另一个分区。
3. 再均衡时，可以通过subscribe()方法中的，均衡监听器，进行处理。
   1. onPartionRevoked 方法，这个方法在再均衡发生之前，和消费者停止读取消息之后被调用。可以用来处理提交消费位移。避免不必要的重复消费。





