---
title: "RocketMQ的两种消费模式"
date: 2025-06-11 19:23:40
categories: RocketMQ
tags: RocketMQ
---

**目录**

一、概述

二、集群消费

三、广播消费

------------------------------------------------------------------------

## **一、概述**

RocketMQ主要提供了两种消费模式：**集群消费以及广播消费**。我们只需要在定义消费者的时候通过\*\*setMessageModel(MessageModel.XXX)\*\*方法就可以指定是集群还是广播式消费，**默认是集群消费模式**，即每个Consumer Group中的Consumer均摊所有的消息。下面我们通过简单的示例演示一下。

## **二、集群消费**

一个Consumer Group 中的 Consumer 实例平均分摊消费消息。

**使用方法：setMessageModel(MessageModel.*CLUSTERING*)**

- 定义生产者

``` java
public class MQProducer {
    public static void main(String[] args) throws MQClientException, UnsupportedEncodingException, RemotingException, InterruptedException, MQBrokerException {
        // 创建DefaultMQProducer类并设定生产者名称
        DefaultMQProducer mqProducer = new DefaultMQProducer("producer-group-test");
        // 设置NameServer地址，如果是集群的话，使用分号;分隔开
        mqProducer.setNamesrvAddr("10.0.91.71:9876");
        // 消息最大长度 默认4M
        mqProducer.setMaxMessageSize(4096);
        // 发送消息超时时间，默认3000
        mqProducer.setSendMsgTimeout(3000);
        // 发送消息失败重试次数，默认2
        mqProducer.setRetryTimesWhenSendAsyncFailed(2);
        // 启动消息生产者
        mqProducer.start();

        // 循环十次，发送十条消息
        for (int i = 1; i <= 10; i++) {
            String msg = "hello, 这是第" + i + "条同步消息";
            // 创建消息，并指定Topic(主题)，Tag(标签)和消息内容
            Message message = new Message("CLUSTERING_TOPIC", "", msg.getBytes(RemotingHelper.DEFAULT_CHARSET));
            // 发送同步消息到一个Broker，可以通过sendResult返回消息是否成功送达
            SendResult sendResult = mqProducer.send(message);
            System.out.println(sendResult);
        }

        // 如果不再发送消息，关闭Producer实例
        mqProducer.shutdown();
    }
}
```

    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC57E00000, offsetMsgId=0A005B4700002A9F00000000000010A9, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=3], queueOffset=5]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC57EF0001, offsetMsgId=0A005B4700002A9F0000000000001174, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=0], queueOffset=5]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC57F60002, offsetMsgId=0A005B4700002A9F000000000000123F, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=1], queueOffset=6]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC57FB0003, offsetMsgId=0A005B4700002A9F000000000000130A, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=2], queueOffset=5]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC58050004, offsetMsgId=0A005B4700002A9F00000000000013D5, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=3], queueOffset=6]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC58110005, offsetMsgId=0A005B4700002A9F00000000000014A0, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=0], queueOffset=6]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC581D0006, offsetMsgId=0A005B4700002A9F000000000000156B, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=1], queueOffset=7]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC58290007, offsetMsgId=0A005B4700002A9F0000000000001636, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=2], queueOffset=6]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC582E0008, offsetMsgId=0A005B4700002A9F0000000000001701, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=3], queueOffset=7]
    SendResult [sendStatus=SEND_OK, msgId=AC6E005625A818B4AAC211CC58340009, offsetMsgId=0A005B4700002A9F00000000000017CC, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=0], queueOffset=7]

- 定义消费者A

``` java
public class MQConsumerA {
    public static void main(String[] args) throws MQClientException {

        // 创建DefaultMQPushConsumer类并设定消费者名称
        DefaultMQPushConsumer mqPushConsumer = new DefaultMQPushConsumer("consumer-group-test");

        // 设置NameServer地址，如果是集群的话，使用分号;分隔开
        mqPushConsumer.setNamesrvAddr("10.0.91.71:9876");

        // 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费
        // 如果不是第一次启动，那么按照上次消费的位置继续消费
        mqPushConsumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        // 设置消费模型，集群还是广播，默认为集群
        mqPushConsumer.setMessageModel(MessageModel.CLUSTERING);

        // 消费者最小线程量
        mqPushConsumer.setConsumeThreadMin(5);

        // 消费者最大线程量
        mqPushConsumer.setConsumeThreadMax(10);

        // 设置一次消费消息的条数，默认是1
        mqPushConsumer.setConsumeMessageBatchMaxSize(1);

        // 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息，如果订阅该主题下的所有tag，则使用*
        mqPushConsumer.subscribe("CLUSTERING_TOPIC", "*");

        // 注册回调实现类来处理从broker拉取回来的消息
        mqPushConsumer.registerMessageListener(new MessageListenerConcurrently() {
            // 监听类实现MessageListenerConcurrently接口即可，重写consumeMessage方法接收数据
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgList, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                MessageExt messageExt = msgList.get(0);
                String body = new String(messageExt.getBody(), StandardCharsets.UTF_8);
                System.out.println("消费者接收到消息: " + messageExt.toString() + "---消息内容为：" + body);
                // 标记该消息已经被成功消费
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        // 启动消费者实例
        mqPushConsumer.start();
    }
}
```

    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=203, queueOffset=6, sysFlag=0, bornTimestamp=1646362604534, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602259, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F000000000000123F, commitLogOffset=4671, bodyCRC=540691780, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=7, CONSUME_START_TIME=1646362604548, UNIQ_KEY=AC6E005625A818B4AAC211CC57F60002, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 51, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第3条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=5, sysFlag=0, bornTimestamp=1646362604527, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602253, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001174, commitLogOffset=4468, bodyCRC=240311509, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=6, CONSUME_START_TIME=1646362604549, UNIQ_KEY=AC6E005625A818B4AAC211CC57EF0001, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 50, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第2条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=6, sysFlag=0, bornTimestamp=1646362604561, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602293, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F00000000000014A0, commitLogOffset=5280, bodyCRC=1516474450, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=7, CONSUME_START_TIME=1646362604579, UNIQ_KEY=AC6E005625A818B4AAC211CC58110005, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 54, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第6条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=203, queueOffset=7, sysFlag=0, bornTimestamp=1646362604573, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602303, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F000000000000156B, commitLogOffset=5483, bodyCRC=1946878403, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=8, CONSUME_START_TIME=1646362604586, UNIQ_KEY=AC6E005625A818B4AAC211CC581D0006, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 55, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第7条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=204, queueOffset=7, sysFlag=0, bornTimestamp=1646362604596, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602322, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F00000000000017CC, commitLogOffset=6092, bodyCRC=2041898758, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=8, CONSUME_START_TIME=1646362604616, UNIQ_KEY=AC6E005625A818B4AAC211CC58340009, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 49, 48, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第10条同步消息

- 定义消费者B

``` java
public class MQConsumerB {
    public static void main(String[] args) throws MQClientException {

        // 创建DefaultMQPushConsumer类并设定消费者名称
        DefaultMQPushConsumer mqPushConsumer = new DefaultMQPushConsumer("consumer-group-test");

        // 设置NameServer地址，如果是集群的话，使用分号;分隔开
        mqPushConsumer.setNamesrvAddr("10.0.91.71:9876");

        // 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费
        // 如果不是第一次启动，那么按照上次消费的位置继续消费
        mqPushConsumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        // 设置消费模型，集群还是广播，默认为集群
        mqPushConsumer.setMessageModel(MessageModel.CLUSTERING);

        // 消费者最小线程量
        mqPushConsumer.setConsumeThreadMin(5);

        // 消费者最大线程量
        mqPushConsumer.setConsumeThreadMax(10);

        // 设置一次消费消息的条数，默认是1
        mqPushConsumer.setConsumeMessageBatchMaxSize(1);

        // 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息，如果订阅该主题下的所有tag，则使用*
        mqPushConsumer.subscribe("CLUSTERING_TOPIC", "*");

        // 注册回调实现类来处理从broker拉取回来的消息
        mqPushConsumer.registerMessageListener(new MessageListenerConcurrently() {
            // 监听类实现MessageListenerConcurrently接口即可，重写consumeMessage方法接收数据
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgList, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                MessageExt messageExt = msgList.get(0);
                String body = new String(messageExt.getBody(), StandardCharsets.UTF_8);
                System.out.println("消费者接收到消息: " + messageExt.toString() + "---消息内容为：" + body);
                // 标记该消息已经被成功消费
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        // 启动消费者实例
        mqPushConsumer.start();
    }
}
```

    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=3, storeSize=203, queueOffset=5, sysFlag=0, bornTimestamp=1646362604513, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602244, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F00000000000010A9, commitLogOffset=4265, bodyCRC=664430631, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=6, CONSUME_START_TIME=1646362604525, UNIQ_KEY=AC6E005625A818B4AAC211CC57E00000, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 49, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第1条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=2, storeSize=203, queueOffset=5, sysFlag=0, bornTimestamp=1646362604539, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602266, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F000000000000130A, commitLogOffset=4874, bodyCRC=1573106993, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=6, CONSUME_START_TIME=1646362604547, UNIQ_KEY=AC6E005625A818B4AAC211CC57FB0003, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 52, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第4条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=3, storeSize=203, queueOffset=6, sysFlag=0, bornTimestamp=1646362604549, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602276, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F00000000000013D5, commitLogOffset=5077, bodyCRC=1940595872, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=7, CONSUME_START_TIME=1646362604558, UNIQ_KEY=AC6E005625A818B4AAC211CC58050004, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 53, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第5条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=2, storeSize=203, queueOffset=6, sysFlag=0, bornTimestamp=1646362604585, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602311, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001636, commitLogOffset=5686, bodyCRC=2061592313, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=7, CONSUME_START_TIME=1646362604592, UNIQ_KEY=AC6E005625A818B4AAC211CC58290007, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 56, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第8条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=3, storeSize=203, queueOffset=7, sysFlag=0, bornTimestamp=1646362604590, bornHost=/10.0.90.139:51996, storeTimestamp=1646362602317, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001701, commitLogOffset=5889, bodyCRC=1418327912, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=8, CONSUME_START_TIME=1646362604598, UNIQ_KEY=AC6E005625A818B4AAC211CC582E0008, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 57, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第9条同步消息

可以看到， 生产者发送了10条消息，ConsumerA与ConsumerB属于同一个消费者组，集群消费模式下每个消费者摊分消费所有消息。注意，两个消费者的ConsumerGroup组名需要一致，才算是同一个消费者组。

**简单总结一下：**

- **1、在Rocket集群消费模式下，（订阅）同一个主题（Topic）下的消息，对于不同的消费者组是一种“广播形式”，即每个消费者组的都会消费消息。**

- **2、在Rocket集群消费模式下，（订阅）同一个主题（Topic）下的消息，对于相同的消费者组的消费者而言是一种集群模式，即同一个消费者组内的所有消费者均分消息并消费。**

## **三、广播消费**

一条消息被多个 Consumer 消费，即使这些 Consumer 属于同一个 Consumer Group，消息也会被 Consumer Group 中的每个 Consumer 都消费一次，广播消费中的 Consumer Group 概念可以认为在消息划分方面无意 义。

**使用方法：setMessageModel(MessageModel.*BROADCASTING*)**

我们将前面的消费者定义中的消息模式设置为BROADCASTING即可，重新启动后观察控制台输出。

- 生产者

<!-- -->

    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D48F970000, offsetMsgId=0A005B4700002A9F0000000000001898, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=0], queueOffset=8]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D48FAE0001, offsetMsgId=0A005B4700002A9F0000000000001963, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=1], queueOffset=8]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D48FBB0002, offsetMsgId=0A005B4700002A9F0000000000001A2E, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=2], queueOffset=7]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D48FCB0003, offsetMsgId=0A005B4700002A9F0000000000001AF9, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=3], queueOffset=8]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D48FDB0004, offsetMsgId=0A005B4700002A9F0000000000001BC4, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=0], queueOffset=9]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D48FF10005, offsetMsgId=0A005B4700002A9F0000000000001C8F, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=1], queueOffset=9]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D4900C0006, offsetMsgId=0A005B4700002A9F0000000000001D5A, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=2], queueOffset=8]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D4901A0007, offsetMsgId=0A005B4700002A9F0000000000001E25, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=3], queueOffset=9]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D4902A0008, offsetMsgId=0A005B4700002A9F0000000000001EF0, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=0], queueOffset=10]
    SendResult [sendStatus=SEND_OK, msgId=AC6E0056231818B4AAC211D490360009, offsetMsgId=0A005B4700002A9F0000000000001FBB, messageQueue=MessageQueue [topic=CLUSTERING_TOPIC, brokerName=broker-a, queueId=1], queueOffset=10]

- 消费者A

<!-- -->

    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143064, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139802, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001898, commitLogOffset=6296, bodyCRC=664430631, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143093, UNIQ_KEY=AC6E0056231818B4AAC211D48F970000, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 49, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第1条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143086, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139825, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001963, commitLogOffset=6499, bodyCRC=240311509, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143125, UNIQ_KEY=AC6E0056231818B4AAC211D48FAE0001, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 50, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第2条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=2, storeSize=203, queueOffset=7, sysFlag=0, bornTimestamp=1646363143099, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139841, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001A2E, commitLogOffset=6702, bodyCRC=540691780, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=8, CONSUME_START_TIME=1646363143144, UNIQ_KEY=AC6E0056231818B4AAC211D48FBB0002, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 51, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第3条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=3, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143115, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139857, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001AF9, commitLogOffset=6905, bodyCRC=1573106993, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143158, UNIQ_KEY=AC6E0056231818B4AAC211D48FCB0003, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 52, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第4条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=9, sysFlag=0, bornTimestamp=1646363143131, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139870, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001BC4, commitLogOffset=7108, bodyCRC=1940595872, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1646363143165, UNIQ_KEY=AC6E0056231818B4AAC211D48FDB0004, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 53, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第5条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=203, queueOffset=9, sysFlag=0, bornTimestamp=1646363143153, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139897, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001C8F, commitLogOffset=7311, bodyCRC=1516474450, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1646363143188, UNIQ_KEY=AC6E0056231818B4AAC211D48FF10005, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 54, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第6条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=2, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143180, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139922, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001D5A, commitLogOffset=7514, bodyCRC=1946878403, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143205, UNIQ_KEY=AC6E0056231818B4AAC211D4900C0006, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 55, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第7条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=3, storeSize=203, queueOffset=9, sysFlag=0, bornTimestamp=1646363143194, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139933, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001E25, commitLogOffset=7717, bodyCRC=2061592313, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1646363143238, UNIQ_KEY=AC6E0056231818B4AAC211D4901A0007, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 56, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第8条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=204, queueOffset=10, sysFlag=0, bornTimestamp=1646363143222, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139956, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001FBB, commitLogOffset=8123, bodyCRC=2041898758, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=11, CONSUME_START_TIME=1646363143240, UNIQ_KEY=AC6E0056231818B4AAC211D490360009, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 49, 48, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第10条同步消息
    消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=10, sysFlag=0, bornTimestamp=1646363143210, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139947, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001EF0, commitLogOffset=7920, bodyCRC=1418327912, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=11, CONSUME_START_TIME=1646363143240, UNIQ_KEY=AC6E0056231818B4AAC211D4902A0008, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 57, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第9条同步消息

- 消费者B

``` java
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143064, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139802, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001898, commitLogOffset=6296, bodyCRC=664430631, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143110, UNIQ_KEY=AC6E0056231818B4AAC211D48F970000, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 49, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第1条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=2, storeSize=203, queueOffset=7, sysFlag=0, bornTimestamp=1646363143099, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139841, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001A2E, commitLogOffset=6702, bodyCRC=540691780, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=8, CONSUME_START_TIME=1646363143140, UNIQ_KEY=AC6E0056231818B4AAC211D48FBB0002, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 51, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第3条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143086, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139825, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001963, commitLogOffset=6499, bodyCRC=240311509, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143141, UNIQ_KEY=AC6E0056231818B4AAC211D48FAE0001, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 50, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第2条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=3, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143115, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139857, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001AF9, commitLogOffset=6905, bodyCRC=1573106993, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143163, UNIQ_KEY=AC6E0056231818B4AAC211D48FCB0003, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 52, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第4条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=9, sysFlag=0, bornTimestamp=1646363143131, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139870, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001BC4, commitLogOffset=7108, bodyCRC=1940595872, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1646363143169, UNIQ_KEY=AC6E0056231818B4AAC211D48FDB0004, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 53, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第5条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=203, queueOffset=9, sysFlag=0, bornTimestamp=1646363143153, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139897, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001C8F, commitLogOffset=7311, bodyCRC=1516474450, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1646363143187, UNIQ_KEY=AC6E0056231818B4AAC211D48FF10005, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 54, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第6条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=2, storeSize=203, queueOffset=8, sysFlag=0, bornTimestamp=1646363143180, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139922, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001D5A, commitLogOffset=7514, bodyCRC=1946878403, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=9, CONSUME_START_TIME=1646363143208, UNIQ_KEY=AC6E0056231818B4AAC211D4900C0006, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 55, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第7条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=3, storeSize=203, queueOffset=9, sysFlag=0, bornTimestamp=1646363143194, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139933, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001E25, commitLogOffset=7717, bodyCRC=2061592313, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=10, CONSUME_START_TIME=1646363143233, UNIQ_KEY=AC6E0056231818B4AAC211D4901A0007, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 56, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第8条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=0, storeSize=203, queueOffset=10, sysFlag=0, bornTimestamp=1646363143210, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139947, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001EF0, commitLogOffset=7920, bodyCRC=1418327912, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=11, CONSUME_START_TIME=1646363143234, UNIQ_KEY=AC6E0056231818B4AAC211D4902A0008, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 57, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第9条同步消息
消费者接收到消息: MessageExt [brokerName=broker-a, queueId=1, storeSize=204, queueOffset=10, sysFlag=0, bornTimestamp=1646363143222, bornHost=/10.0.90.139:52985, storeTimestamp=1646363139956, storeHost=/10.0.91.71:10911, msgId=0A005B4700002A9F0000000000001FBB, commitLogOffset=8123, bodyCRC=2041898758, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='CLUSTERING_TOPIC', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=11, CONSUME_START_TIME=1646363143236, UNIQ_KEY=AC6E0056231818B4AAC211D490360009, CLUSTER=DefaultCluster}, body=[104, 101, 108, 108, 111, 44, 32, -24, -65, -103, -26, -104, -81, -25, -84, -84, 49, 48, -26, -99, -95, -27, -112, -116, -26, -83, -91, -26, -74, -120, -26, -127, -81], transactionId='null'}]---消息内容为：hello, 这是第10条同步消息
```

可以看到， 生产者发送了10条消息，ConsumerA与ConsumerB属于同一个消费者组，广播模式下每个消费者都会全量消费所有消息。
