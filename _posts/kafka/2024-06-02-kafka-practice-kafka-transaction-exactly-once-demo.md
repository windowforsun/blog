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


### Wiremock
앞서 소개한 것과 같이 애플리케이션에 소비된 메시지는 `Outbound Topic 1` 에 메시지를 발생하고, `REST` 호출 수행, `Outbound Topic 2` 에 메시지를 발생한다. 
여기서 `REST` 호출은 `Kafka Transaction` 에 원자적으로 포함될 수 없다. 
하지만 이런 특징을 활용해서 강제적인 `RetryableException` 을 발생시켜 재시도를 수행할 수 있다는 점과 [Wiremock](https://wiremock.org/)
을 사용해서 `REST` 의 결과를 제어하는 방식을 통해 시나리오에 맞는 테스트를 진행하고자 한다. 
`Wiremock` 을 사용하면 첫 번째 `REST` 요청은 500 에러로 실패 했지만, 그 다음 요청은 200 으로 성공하도록 제어가 가능하다.  

```java
stubWiremock("/api/kafkatransactionsdemo/" + key, 500, "Unavailable", "failOnce", STARTED, "succeedNextTime");
stubWiremock("/api/kafkatransactionsdemo/" + key, 200, "success", "failOnce", "succeedNextTime", "succeedNextTime");
```  

메시지 폴링 후 `Outbound Topic 1` 으로 메시지를 전송하고 `REST` 요청이 500 으로 실패하면 해당 메시지 폴링은 실패한다. 
그리고 다음 폴링에서 동일한 메시지가 다시 소비되는데, 
다시 `Outbound Topic 1` 으로 메시지를 전송하고 이번 `REST` 요청은 200 으로 성공하게 된다. 
그러면 `Outbound Topic 2` 에도 메시지가 전송된다. 
그리고 `Producer` 는 `Consumer Coordinator` 로 `Consumer` 의 `offsets` 를 전송해 해당 메시지가 성공적으로 소비 됐음을 알린다. 
최종적으로 `Transaction` 이 완료되면, `Spring` 에서는 `Transaction Coordinator` 를 통해 두 `Outbound Topic` 과 `Consumer Offset` 에 
`Commit Marker` 를 기록함으로써 트랜잭션 커밋이 완료 된다.  

### Embedded Kafka
`Spring Boot` 에서는 `Embedded Kafka` 를 사용해서 `Kafka` 구현 코드에 대한 통합 테스트를 지원한다. 
`Embedded Kafka` 는 `in-memory` 방식으로 동작하는 `Kafka Broker` 로 아래 의존성 추가와 테스트 코드상 어노케이션을 선언하는 방식으로 
손 쉽게 사용할 수 있다.  

```groovy
testImplementation 'org.springframework.kafka:spring-kafka-test'
```

```java
@EmbeddedKafka(controlledShutdown = true, 
        count = 3, 
        topics = {"demo-transactional-inbound-topic", "demo-non-transactional-inbound-topic"})
```  

위와 같은 설정으로 테스트를 실행하면 3개의 `Kafka Broker` 와 2개의 토픽을 사용해서 테스트를 수행 할 수 있는 카프카 환경이 구성된다.  

### Test Consumer
데모 구성을 통해 최종적으로 보고자하는 것은 구현한 애플리케이션이 발생하는 `Outbound Topic` 을 구독하는 `Consumer` 에게 
메시지가 어떤 식으로 전달되는지 이다. 
그러므로 테스트 코드에서는 총 4개의 별도로 구성한 `Consumer` 를 사용한다. 
애플리케이션에서 발행하는 토픽이 `Outbound Topic 1, 2` 로 2개이기 때문에 토픽 한개당 2개의 `Consumer` 씩 둔다. 
그리고 한 토픽에서 각 `Consumer` 서로다른 `Consumer Group` 을 사용하고 아래와 같이 서로다른 `ISOLATION_LEVEL` 로 
`READ_COMMITTED` 와 `READ_UNCOMMITTED` 로 설정한다. 

```java
config.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, IsolationLevel.READ_COMMITTED.toString().toLowerCase(Locale.ROOT));
// or
config.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, IsolationLevel.READ_UNCOMMITTED.toString().toLowerCase(Locale.ROOT));
```  

애플리케이션에서 생산한 메시지는 각 `Consumer` 에게 전달되고, 
해당 `Test Consumer` 들은 메시지가 몇번 자신에게 전달 됐는지 카운트 하여 `Exactly-Once` 에 대한 결과를 확인 한다.  

### Test Result
최종적으로 애플리케이션에서 발행하는 메시지를 소비하는 `Consumer` 중 `ISOLATION_LEVEL` 이 `READ_COMMITTED` 인 `Consumer` 만 
애플리케이션 처리가 중간에 실패해 재시도 과는 과정에서도 `Exactly-Once` 특성에 맞게 메시지를 단 한번만 소비한다.  

`Outbound Topic 1` 을 구독하는 `Consumer` 중 `READ_COMMITTED` 는 한번 만 메시지를 소비하지만, 
`READ_UNCOMMITTED` 는 2번 메시지를 소비하게 된다. 
이는 재시도 과정에서 `Kafka Transaction` 이 정상적으로 커밋하지 않은 메시지도 소비하여 중복 메시지가 소비된 것이다.  

그리고 `Outbound Topic 2` 구독하는 `Consumer` 는 `ISOLATION` 레벨에 관계없이 모두 한번의 메시지 소비만 발생한다.  

이로써 `Kafka` 를 사용하는 애플리케이션에서 메시지를 주고 받을 때 `Exactly-Once` 를 보장 받기 위해서는 메시지를 생산하는 애플리케이션에서 
`Kafka Transaction` 스펙에 맞는 `Consumer` 와 `Producer` 설정이 필요하고, 
해당 애플리케이션에서 생산하는 토픽을 구독하는 `Consumer` 또한 `READ_COMMITTED` 로 구성 돼야 중복 메시지에 대한 대응이 가능하다는 것을 알 수 있다.  


---  
## Reference
[Kafka Transactions: Part 2 - Spring Boot Demo](https://www.lydtechconsulting.com/blog-kafka-transactions-part2.html)     





