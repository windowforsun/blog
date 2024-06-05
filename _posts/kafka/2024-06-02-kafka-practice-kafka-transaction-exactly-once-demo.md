--- 
layout: single
classes: wide
title: "[Kafka] Kafka Transaction Exactly Once Demo"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Transaction
    - Kafka Transaction
    - Exactly-Once
    - Isolation Level
    - Consumer
    - Producer
toc: true
use_math: true
---

## Kafka Transaction Demo
[]()
에서 `Kafka Transaction` 과 `Exactly-Once` 에 대해 개념적으로 알아 보았다.
이번 포스팅에서는 이를 검증해볼 수 있는 애플리케이션을 통해 구현 방식을 바탕으로 좀 더 알아보고자한다. 

### Demo
데모 애플리케이션은 `Kafka Transaction` 을 통해 `Exactly-Once` 를 테스트해볼 수 있다. 
`Inbound Topic` 으로 수신된 메시지는 구현한 `Consumer` 로 소비되고, 
해당 메시지는 애플리케이션 처리를 수행한 후 결과를 `Kafka Transaction` 을 사용하는 방식과 사용하지 않는 방식으로 각 `Outbound Topic` 에 보내진다. 
그리고 최종적으로 `Outbound Topic` 을 구독하는 `Consumer` 가 이를 소비해 메시지가 어떤식으로 전달 됐는지 살펴본다.  

데모 애플리케이션의 전체 코드는 []()
에서 확인 할 수 있다.  

아래 그림은 데모의 구성 요소와 애플리케이션의 동작 과정을 보여준다.  

.. 그림 ..

1. `Inbound Topic` 으로 수신된 메시지는 처리 후 `Outbound Topic 1` 로 메시지를 전송한다. 
2. `Third party service` 의 `REST` 호출을 수행한다. 
3. `Outbound Topic 2` 로 메시지를 전송한다. 

데모는 위 과정 중 `REST` 호출이 이뤄지는 부분을 `Wiremock` 통해 성공/실패를 제어하는 방식으로 
성공과 실패 과정 그리고 `Kafka Transaction` 적용 여부 및 `READ_COMMITTED`, `READ_UNCOMMITTED` 소비 방식에 따른 결과를 살펴볼 것이다.  


### Kafka Transaction Enabled Producer Config
데모에서 `Kafka Transaction` 이 활성화된 `Producer` 의 설정은 아래와 같이, 
`Transaction Id` 설정과 `Idempotence` 설정을 활성화 시켜 `ProducerFactory` 를 설정한다. 

```java
config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "transaction-id");
config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
```  

그리고 `Kafka Transaction` 이 활성화 된 `ProducerFactory` 통해 `KafkaTransactionManager` 와 `KafkaTemplate` 를 생성한다. 

```java
@Bean
public KafkaTransactionManager kafkaTransactionManager(final ProducerFactory<String, String> producerFactoryTransactional) {
    return new KafkaTransactionManager<>(producerFactoryTransactional);
}

@Bean
public KafkaTemplate<String, String> kafkaTemplateTransactional(final ProducerFactory<String, String> producerFactoryTransactional) {
    return new KafkaTemplate<>(producerFactoryTransactional);
}
```  

최종적으로 `KafkaTransaction` 가 활성화된 `Producer` 를 사용해서 메시지를 `Outbound Topic 1, 2` 에 발생하는데, 
코드는 아래와 같이 `@Transactional` 어노테이션과 함께 사용한다.  

```java
@Transactional
public void processWithTransaction(String key, DemoInboundEvent event) {
    this.kafkaClient.sendMessageWithTransaction(key, event.getData(), this.properties.getOutboundTopic1());
    this.callThirdparty(key);
    this.kafkaClient.sendMessageWithTransaction(key, event.getData(), this.properties.getOutboundTopic2());
}
```  

### Kafka Transaction Enabled Consumer Config
`Kafka Transaction` 이 활성화된 `Consumer` 설정은 아래와 같이, 
`Auto Commit` 을 비활성화로 설정한다.  

```java
config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
```  

그리고 위 설정을 사용해서 `KafkaListenerContainerFactory` 를 생성한다. 

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(final ConsumerFactory<String, String> consumerFactory) {
    final SeekToCurrentErrorHandler errorHandler = new SeekToCurrentErrorHandler((record, e) -> {

    }, new FixedBackOff(4000L, 4L));
    final ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
    factory.setErrorHandler(errorHandler);

    return factory;
}
```  

최종적으로 `Inbound Topic` 메시지는 `@KafkaListener` 어노케이션에 토픽과 `KafkaListenerContainerFactory` 를 지정해서 가능하다.  

```java
@KafkaListener(topics = "demo-transactional-inbound-topic", 
        groupId = "kafkaConsumerGroup", 
        containerFactory = "kafkaListenerContainerFactory")
public void listen(@Header(KafkaClient.EVENT_ID_HEADER_KEY) String eventId, 
    @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String key, @Payload final String payload)
```  

애플리케이션은 `Consume - Process - Produce` 과정을 거치게 된다. 
여기서 `Kafka Transaction` 이 활성화된 흐름과 비활성화 된 상태를 비교하기 위해 
`KafkaTransactionConsumer`, `KafkaNonTransactionConsumer` 2개를 사용해 차이점을 알아 볼 것이다.  

