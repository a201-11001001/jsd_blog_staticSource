---
title: RabbitMQ避免消息不丢失
date: 2023-10-27 21:17:51
tags: 
    - RabbitMQ
description: RabbitMQ避免消息不丢失 ## 页面描述
keywords: 加密  ## 关键字
comments: true  ## 评论模块
cover: /top_img/RabbitMQ避免消息丢失top_img.jpg ## 文章缩略图
# top_img:  ## 顶部图片
aside: false  ## 显示侧边栏（默认 true）
---

#### RabbitMQ避免消息丢失
---

##### RabbitMQ消息丢失的场景
![RabbitMQ消息丢失的场景](/WX20231205-204120@2x.png)
1. 生产者弄丢了数据,消息发送给 RabbitMq 的时候出现了网络波动,导致服务没有收到消息
2. RabbitMQ 没有持久化,自己把消息弄丢了
3. 消费端弄丢了数据,消费端接收到数据还没来及消费出现了宕机或故障导致消息丢失

##### 1、保证消息到RabbitMQ服务 
* 开启事物
    ```
    // 结合声明式事物
    @Transactional // 事务注解
    public void sendMessage() {
        // 开启事务
        rabbitTemplate.setChannelTransacted(true);
        // 发送消息
        rabbitTemplate.convertAndSend(RabbitMQConfig.Direct_Exchange, routingKey, message);
    }
    ```
* 使用 confirm 机制
    > 事务机制和 confirm 机制最大的不同在于，事务机制是同步的，你提交一个事务之后会阻塞在那儿，但是 confirm 机制是异步的

    ```
    # 配置文件中开启 confirm
    spring:
        rabbitmq:
            publisher-confirm-type: correlated  # 开启发送方确认机制
    ```

    在生产者开启了confirm模式之后，每次写的消息都会分配一个唯一的id，然后如果写入了rabbitmq之中，rabbitmq会给你回传一个ack消息，告诉你这个消息发送OK了；如果rabbitmq没能处理这个消息，通过调用 setConfirmCallback 可以进行重试。而且可以结合这个机制知道自己在内存里维护每个消息的id，如果超过一定时间还没接收到这个消息的回调，那么可以进行重发。
    ```
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Override
    public void sendMessage() {
        // 发送消息
        rabbitTemplate.convertAndSend(RabbitMQConfig.Direct_Exchange, routingKey, message);
        // 设置消息确认回调方法
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            /**
             * MQ确认回调方法
             * @param correlationData 消息的唯一标识
             * @param ack 消息是否成功收到
             * @param cause 失败原因
             */
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                // 记录日志
                log.info("ConfirmCallback...correlationData["+correlationData+"]==>ack:["+ack+"]==>cause:["+cause+"]");
                if (!ack) {
                    // 出错处理
                    ...
                }
            }
        });
    }
    ```

##### 2、避免 RabbiqMQ 自己把消息丢了
* 开启持久化 
    1. Exchange 设置持久化
    2. Queue 设置持久化
    3. Message持久化发送：发送消息设置发送模式deliveryMode=2，代表持久化消息

* 设置集群镜像模式
    1. 单节点模式：最简单的情况，非集群模式，节点挂了，消息就不能用了。业务可能瘫痪，只能等待。
    
    2. 普通模式：消息只会存在与当前节点中，并不会同步到其他节点，当前节点宕机，有影响的业务会瘫痪，只能等待节点恢复重启可用（必须持久化消息情况下）。
    
    3. 镜像模式：消息会同步到其他节点上，可以设置同步的节点个数，但吞吐量会下降。属于RabbitMQ的HA方案

[集群讲的很有深度的文章]https://blog.csdn.net/weixin_43498985/article/details/122185972

![RabbitMQ镜像模式](RabbitMQ镜像模式.png)
> 其中最常用的就是cluster模式（集群），它可以动态增加节点或减少，但只支持同一网段的局域网内的节点。而federation允许单台服务器上或多台服务器组成的集群之间进行消息转发和路由。federation队列类似于单向点对点连接，消息会在整个联合队列之间转发任意次，直到被消费者接收。但是此方式也有弊端，就是无法实现高可用，当其中的某一个节点宕机时就导致服务不可用，这时候就需要引入中间件Hadoop。通常使用federation来连接internet上的中间服务器，用作订阅分发消息或工作队列。shovel与federation类似，不过实现更偏向于底层，它们两者均支持部署广域网节点。下面我们介绍cluster集群模式和部署过程。


————————————————
版权声明：本文为CSDN博主「一条特立独行的狗、」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_43498985/article/details/122185972

![RabbitMQ镜像模式工作原理](RabbitMQ镜像模式工作原理.png)
上图就是镜像集群模式的实现流程，其中有三个节点（主节点、备节点1、备节点2）和三个镜像队列queue（其中备节点上的queue是由主节点镜像生成的）。要注意的是，这里的主节点和备节点是针对某个队列而言的，并不能认为一个节点作为了所有队列的主节点，因为在整个镜像集群模式下，会存在多个节点和多个队列，这时候任何一个节点都能作为某一个队列的镜像主节点，其它节点则成了镜像备节点（例如：有A、B、C三个节点和Q1、Q2、Q3三个队列，如果A作为Q1的镜像主节点，那么B和C就作为了Q1的镜像备节点，在此基础上，如果B作为了Q2的镜像主节点，那么A和C就是Q2的镜像备节点）。

每一个队列都是由两部分组成的，一个是queue，用来接收消息和发布消息，另外还有一个BackingQueue，它是用来做本地消息持久化处理。客户端发送给主节点队列的消息和ack应答都将会同步到其它备节点上。
> 所有关于镜像主队列（mirror_queue_master）的操作，都会通过组播GM的方式同步到其它备用节点上，这里的GM负责消息的广播，mirror_queue_slave则负责回调处理（更新本次同步内容），因此当消息发送给备用节点时，则由mirror_queue_slave来做实际处理，将消息存储在queue中，如果是持久化消息则同时存储在BackingQueue中。master上的回调则由coordinator来处理（发布本次同步内容）。在主节点中，BackingQueue的存储则是由Queue进行调用。对于生产者而言，消息发送给queue之后，接着调用mirror_queue_master进行持久化处理，之后再通过GM广播发送本次同步消息给备用节点，备用节点通过回调mirror_queue_slave同步本次消息到queue和BackingQueue；对于消费者而言，从queue中获取消息之后，消息队列会等待消费者的ack应答，ack应答收到之后删除queue和BackingQueue中的该条消息，并将本次ack内容通过GM广播发送给备用节点同步本次操作。如果slave宕机了，那对于客户端的服务提供将不会有任何影响。如果master宕机了，则其它备用节点就提升为master继续服务消息不会丢失。那这其中多个备用节点是如何选择其中一个来作为master的呢？这里通过选取出“最年长的”节点作为master，因为这个备用节点相对于其它节点而言是同步时间最长、同步状态最好的一个节点，但如果存在没有任何一个slave与master完全同步的情况，那么master中未同步的消息将会丢失。
————————————————
版权声明：本文为CSDN博主「一条特立独行的狗、」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_43498985/article/details/122185972

##### GM
GM模块实现的一种可靠的组播通讯协议，该协议能够保证组播消息的原子性，即保证组中活着的节点要么都收到消息要么都收不到。
它的实现大致为：将所有的节点形成一个循环链表，每个节点都会监控位于自己左右两边的节点，当有节点新增时，相邻的节点保证当前广播的消息会复制到新的节点上；当有节点失效时，相邻的节点会接管保证本次广播的消息会复制到下一个节点。在master节点和slave节点上的这些gm形成一个group，group（gm_group）的信息会记录在mnesia中。不同的镜像队列形成不同的group。消息从master节点对应的gm发出后，顺着链表依次传送到所有的节点，由于所有节点组成一个循环链表，master节点对应的gm最终会收到自己发送的消息，这个时候master节点就知道消息已经复制到所有的slave节点了。另外需要注意的是，每一个新节点的加入都会先清空这个节点原有数据，下图是新节点加入集群的一个简单模型：
————————————————
版权声明：本文为CSDN博主「一条特立独行的狗、」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_43498985/article/details/122185972

![GM](RabbitMQ_GM.png)
消息的同步：
将新节点加入已存在的镜像队列，在默认情况下ha-sync-mode=manual，镜像队列中的消息不会主动同步到新节点，除非显式调用同步命令。当调用同步命令后，队列开始阻塞，无法对其进行操作，直到同步完毕。

总结
镜像集群模式通过从主节点拷贝消息的方式使所有节点都能保留一份数据，一旦主节点崩溃，备节点就能完成替换从而继续对外提供服务。这解决了节点宕机带来的困扰，提高了服务稳定性，但是它并不能实现负载均衡，因为每个操作都要在所有节点做一遍，这无疑降低了系统性能。再者当消息大量入队时，集群内部的网络带宽会因此时的同步通讯被大大消耗掉，因此对于可靠性要求高、性能要求不高且消息量并不多的场景比较适用。如果对高可用和负载均衡都有要求的场景则需要结合HAProxy（实现节点间负载均衡）和keepalived（实现HAproxy的主备模式）中间件搭配使用，下面我们将对这种场景的部署进行全流程概述。

————————————————
版权声明：本文为CSDN博主「一条特立独行的狗、」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_43498985/article/details/122185972

##### 3、避免消费者丢失消息
开启事物ACK机制
![](ACK机制.png)
```
# ack 模式设置为手动的
spring:
  rabbitmq:
    listener:
      simple:
		acknowledge-mode: manual # 手动ack模式
        concurrency: 1 # 最少消费者数量
        max-concurrency: 10 # 最大消费者数量
```

```
@Component
public class MQConsumer {
 
    
    @Autowired
    private DispatcherService dispatcherService;
    
    @RabbitListener(queues = "order.queue")
    public void messageConsumer(String orderMsg, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
 
        try {
 
            System.out.println("消息：" + orderMsg);
            JSONObject order = JSONObject.parseObject(orderMsg);
            String orderId = order.getString("orderId");
            // 派单处理
            dispatcherService.dispatch(orderId);

            System.out.println(1 / 0); // 出现异常
            // 手动确认
            channel.basicAck(tag, false);
        } catch (Exception e) {
 
            // 如果出现异常的情况下 根据实际情况重发
            // 重发一次后，丢失
            // 参数1：消息的tag
            // 参数2：多条处理
            // 参数3：重发
                // false 不会重发，会把消息打入到死信队列
                // true 重发，建议不使用try/catch 否则会死循环
            
            // 手动拒绝消息
            channel.basicNack(tag, false, false);
        }
    }   
}
```
##### 手动 Ack 存在的弊端
如果消费者存在Bug的话，就会导致所有的消息都抛出异常，然后队列的Unacked消息数暴涨，导致MQ响应越来越慢，然后down掉 。

原因:因为上面消费者抛出异常，所以MQ没有得到ack响应，注意：这些消息会堆积在Unacked消息里，不会抛弃，即使另外打开一个消费者也不会被消费，直到原来的消费者客户端断开重连时，才会变成ready，这时如果通过qos设置了prefetch，没有ack响应的话，Broker不会再分配新的消息下来，就导致了阻塞

解决方案: 重试机制+死信队列 

##### 如何保证RabbitMQ消息不重复消费
消费者的业务从 MQ 队列中接收数据 => 接着处理业务 => 业务处理成功后，消费者给 MQ 返回 ack 进行手动确认 => 返回回调执行结果的过程中，因为网络抖动等原因，回调数据时，MQ没有返回成功。所以MQ队列中的数据会再次发给业务项目，造成重复消费。

> 解决方案:

* 消息确认机制：消费者在消费消息后向RabbitMQ服务器发送确认消息（ACK），RabbitMQ收到确认消息后才会将该消息从队列中删除，如果由于某种原因消费者未能发送确认消息，RabbitMQ会认为该消息未被正常处理，然后重新将该消息发送给其他消费者或者当前消费者进行重试。
* 幂等性处理：通过在系统设计中引入唯一标识符，使得同一个消息可以被多次接收并处理，但只有第一次处理会对系统状态产生影响。例如，消费端可以在处理完一个消息后，在数据库中记录下该消息的ID，以便之后查询是否已经处理过该消息，如果已经处理过则直接忽略该消息。

---
最后
[RabbitMQ集群部署] https://blog.csdn.net/weixin_43498985/article/details/122185972