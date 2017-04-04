

layout: post
categories: [Java]
tags: [Java]
code: true
title: 初识Kafka


	最近因为公司业务增加，原来的ActiveMQ集群不稳定，生产者发送MQ到队列时经常报RequestTimedOutIOException超时异常。部分业务需要切Kafka。之前没有接触过Kafka，上周末看了下文档和问了几个同事后，周一上线，运行正常。

	Kafka我们选用了0.10版本，先上代码：

`

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    
           <bean id="producerProperties" class="java.util.HashMap">
                  <constructor-arg>
                         <map>
                                <entry key="bootstrap.servers" value="${bootstrap.servers}"/>
                                <!--<entry key="group.id" value="0"/>-->
                                <entry key="retries" value="3"/>
                                <entry key="batch.size" value="16384"/>
                                <entry key="linger.ms" value="1"/>
                                <entry key="buffer.memory" value="33554432"/>
                                <entry key="key.serializer" value="org.apache.kafka.common.serialization.StringSerializer"/>
                                <entry key="value.serializer" value="org.apache.kafka.common.serialization.StringSerializer"/>
                         </map>
                  </constructor-arg>
           </bean>
    
           <bean id="imKafkaProducer" class="org.apache.kafka.clients.producer.KafkaProducer">
                  <constructor-arg>
                         <ref bean="producerProperties"/>
                  </constructor-arg>
           </bean>
    
           <bean id="pushKafkaProducer" class="org.apache.kafka.clients.producer.KafkaProducer">
                  <constructor-arg>
                         <ref bean="producerProperties"/>
                  </constructor-arg>
           </bean>
    
    
    </beans>

`

`

    package com.chen.biz.message.producer.queue;
    
    import org.apache.kafka.clients.producer.Callback;
    import org.apache.kafka.clients.producer.Producer;
    import org.apache.kafka.clients.producer.ProducerRecord;
    import org.apache.kafka.clients.producer.RecordMetadata;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.stereotype.Service;
    
    import javax.annotation.Resource;
    
    /**
     * Created by chen on 2017/3/23.
     */
    @Service
    public class KafkaQueueMessageProducer {
    
        final private Logger logger = LoggerFactory.getLogger(this.getClass());
    
    //    @Value("${feed.kafka.topic.provider.im.subject}")
        private String IM_TOPIC = "mobile-api-common-im";
    
    //    @Value("${feed.kafka.topic.provider.push.subject}")
        private String PUSH_TOPIC = "mobile-api-common-push";
    
        @Resource(name = "imKafkaProducer")
        private Producer<Object, String> imKafkaProducer;
        @Resource(name = "pushKafkaProducer")
        private Producer<Object, String> pushKafkaProducer;
    
        public static KafkaQueueMessageProducer getInstance() {
            return SpringContextUtils.getBean("kafkaQueueMessageProducer", KafkaQueueMessageProducer.class);
        }
    
        public void sendIMQueueMessage(final String message) {
            try {
                new ThreadPoolExecutor(50, 50, 0L, TimeUnit.MILLISECONDS,
    			new LinkedBlockingQueue<Runnable>(50000)).execute(new Runnable() {
                    @Override
                    public void run() {
                      long beginTime = System.currentTimeMillis();
                      imKafkaProducer.send(new ProducerRecord<>(IM_TOPIC, message),
                                           new Callback() {
                                             public void onCompletion(RecordMetadata metadata, Exception e) {
                                               if(e != null) {
                                                 logger.error("send IM exception:" + e.getMessage(), e);
                                               } else {
                                                 logger.info("The IM offset of the record we just sent is: " + metadata.toString());
                                               }
                                             }
                                           }
                                          );
                      logger.info("KafkaQueueMessageProducer--sendIMQueueMessage--String--end--message={},time={}", message, System.currentTimeMillis() - beginTime);
                    }
                });
            } catch (Exception e) {
                logger.error("sendIMQueueMessage error", e);
            }
        }
    
        public void sendPushQueueMessage(final String message) {
          try {
                ew ThreadPoolExecutor(50, 50, 0L, TimeUnit.MILLISECONDS,
    			new LinkedBlockingQueue<Runnable>(50000)).execute(new Runnable() {
                    @Override
                    public void run() {
                            pushKafkaProducer.send(new ProducerRecord<>(PUSH_TOPIC, message),
                                    new Callback() {
                                        public void onCompletion(RecordMetadata metadata, Exception e) {
                                            if(e != null) {
                                                logger.error("send PUSH exception:" + e.getMessage(), e);
                                            } else {
                                                logger.info("The PUSH offset of the record we just sent is: " + metadata.toString());
                                            }
                                        }
                                    }
                            );
                            logger.info("KafkaQueueMessageProducer--sendPushQueueMessage--String--end--message={},time={}", message, System.currentTimeMillis() - beginTime);
                    }
                });
            } catch (Exception e) {
                logger.error("sendPushQueueMessage error", e);
            }
        }
    }

`

	Kafka使用文件存储消息，Kafka对日志文件进行追加操作。和JMS实现的不同的是：即使消息被消费者消费，消息不会马上被删除，可以通过配置保留一定的时间。消费者可以根据offset记录读的位置，而offset信息保存在zookeeper,不用Producer和Consumer维护。

	JMS有queuing和publish-subscribe两种，而Kafka只支持Topic，每个consumer属于一个consumer group，每个group中可以有多个consumer。发送到Topic的消息,只会被订阅此Topic的每个group中的一个consumer消费，如果所有的consumer都具有相同的group，这种情况和queue模式很像；消息将会在consumers之间负载均衡。如果所有的consumer都具有不同的group,那这就是"发布-订阅"，消息将会广播给所有的消费者。在kafka中，一个partition中的消息只会被group中的一个consumer消费。我们可以认为一个group是一个“订阅者”，一个Topic中的每个partions，只会被一个"订阅者"中的一个consumer消费，不过一个consumer可以消费多个partitions中的消息。kafka只能保证一个partition中的消息被某个consumer消费时，消息是顺序的，但是从Topic角度上不是有序的。

	其中，可以设置partition分区的大小，默认只有三块分区，我们线上环境给了108块，生产者可以指定消息写入的partition，Producer的send(ProducerRecord<K, V> record)方法ProducerRecord参数可以指定Key，可以通过这个key hash到分区，同时可以重写hash策略；如果不指定则按照默认的算法hash到分区。有意思的是其中使用到了murmur2哈希算法，Redis也用到了。

	
