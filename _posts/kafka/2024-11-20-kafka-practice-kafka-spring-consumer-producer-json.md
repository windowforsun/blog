--- 
layout: single
classes: wide
title: "[Kafka] "
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
toc: true
use_math: true
---  

## Spring Kafka Consumer Producer
[Spring Kafka](https://spring.io/projects/spring-kafka)
는 `Kafka Consumer` 와 `Kafka Producer API` 를 추상화한 구현체이므로 이를 사용해서 `Kafka Cluster` 
토픽에 메시지를 읽고 쓸 수 있다. 
사용법은 일반적인 `Spring` 프로젝트와 동일하게 비지니스 로직에는 큰 영향 없이 `Annotation` 을 
기반으로 설정과 빈 주입이 가능하다.  

`Spring Kafka` 를 사용해서 구현한 애플리케이션을 도식화하면 아래와 같이 `Inbound Topic` 에서 
메시지를 `Consume` 하고 처리 후 `Outbound Topic` 으로 메시지를 `Producd` 하는 형상이다.  

.. 그림 ..

실 서비스에서 안정적인 메시징을 위해서는 메시지 중복 처리, 트랜잭션, 메시지 순서 등 고려할 것들이 많다. 
하지만 이번 포스팅에서는 `Spring Kafka` 를 기반으로 `Kafka Broker` 와 메시지 소비/생산에 대한 기본적인 
부분에 대해서만 초점을 맞춘 내용만 다룬다.  

### Consuming Message
`Kafka Broker` 로 부터 메시지를 소비하는 시작점은 `@Kafka Listener` 어노테이션이다. 
메시지를 전달 받아 처리할 메소드에 해당 어노테이션을 아래와 같이 선언해주면 된다.  

```java
@KafkaListener(topics = "exam-inbound-topic",
        groupId = "exam-consumer-group",
        containerFactory = "kafkaListenerContainerFactory")
public void listen(@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String key,
        @Payload final String payload) {
    // ...
}
```  

[@KafkaListener](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html)
에는 `Kafka Consumer` 에 설정할 수 있는 다양한 설정 값들이 존재한다. 
위 코드에서는 `exam-inbound-topic` 이라는 토픽에서 메시지를 소비하고, 
`Consumer Group` 의 아이디는 `exam-consumer-group` 으로 지정했다. 
그리고 메시지 소비에 사용할 `Kafka Consumer` 인스턴스는 `kafkaListenerContainerFactory` 라는 
`ContainerFactory` 를 사용하도록 했다. 
이후 설명에 나오겠지만, `kafkaListenerContainerFactory` 는 별도의 `JavaConfig` 에서 
빈을 선언해 줄 것이다.  

위와 같이 토픽으로 부터 `Kafka Consumer` 가 메시지를 소비하기 위한 추가적인 구현코드는 필요하지 않다. 
메시지를 소비하고 해당 메시지를 처리할 비지니스 로직에만 집중하면 된다.  

#### Consumer Group
다른 포스팅에서도 다룬 내용이지만, `Consumer Group` 은 `Kafka` 생태계에서 
처리량과 안전성에 밀접한 관계가 있다. 
토픽을 구성하는 `Partition` 의 수 만큼 동일한 `Consumer Group` 으로 `Consumer Instance` 를 구성해 
토픽을 기준으로 처리량을 크게 늘릴 수 있다. 
같은 `Consumer Group` 아이디를 같은 `Consumer Instance` 들은 자신이 소비하는 토픽의 하나 이상의 `Partition` 을 할당 받을 수 있기 때문이다. 
이러한 개념이기 때문에 각 `Consumer Instance` 는 다른 `Partition` 의 메시지를 소비하므로 서로 다른 메시지를 소비하게 된다. 
만약 토픽을 구성하는 `Partition` 의 수보다 많은 `Consumer Instance` 를 동일한 `Consumer Group` 으로 구성한다면, 
`Partition` 수 이상의 `Consumer Instance` 들은 메시지를 소비하지 않는 `Idle` 상태가 된다. 
그리고 이러한 개념을 통해 토픽 메시지 처리에 안정성을 높이는 `Stand-by` 모드로 `Consumer Instance` 를 추가로 구성해 둘 수 있다.  

그리고 만약 서로 다른 `Consumer Group` 아이디로 동일한 토픽을 구독한다면, 
`Consumer Group` 을 기준으로 서로 다른 `Consumer Group` 은 모두 동일한 메시지를 소비하게 된다.  

#### Listener Container Factory
`Listener Container Factory` 는 `Kafka Broker` 의 지정된 토픽으로 부터 메시지를 소비하고, 
`@KafkaListener` 어노테이션이 선언된 메소드(e.g. listen) 를 호출하는 역할을 수행한다. 
앞선 코드 예시의 `@KafkaListener` 의 `containerFactory` 설정에는 `Listener Conatiner Factory` 의 
빈 이름을 설정해주면 된다. 
그리고 실제 빈은 아래와 같이 `Java Config` 를 통해 미리 선언돼 있어야 한다.  

```java
@Bean
public ConcurrentKafkaListenerContainerFactory kafkaListenerContainerFactory(final ConsumerFactory consumerFactory) {
   final ConcurrentKafkaListenerContainerFactory factory = new ConcurrentKafkaListenerContainerFactory();
   factory.setConsumerFactory(consumerFactory);
   return factory;
}
```  

`Listener Container Facotry` 에는 추가적으로 동시성, 재시도, 메시지 필터링, 에러 핸들링 등과 같은 
설정을 추가할 수 있다. 
이중 몇가지는 `@KafkaListener` 어노테이션에서도 설정이 가능하다.  


#### Consumer Factory
`Consumer Factory` 는 `Listener Container Factory` 를 생성하는데 필요한 필수 요소로 
이또한 아래와 같이 `Java Config` 를 통해 선언해 줄 수 있다. 
아래는 `StringDeserializer` 를 통해 키/메시지를 문자열로 역직렬화하는 경우의 예시이다.  

```java
@Bean
public ConsumerFactory consumerFactory(@Value("${kafka.bootstrap-servers}") final String bootstrapServers) {
   final Map config = new HashMap<>();
   config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
   config.put(ConsumerConfig.GROUP_ID_CONFIG, "exam-consumer-group-2");
   config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
   config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
   return new DefaultKafkaConsumerFactory<>(config);
}
```  

`Consumer Factory` 는 `Kafka Consumer` 를 설정하는 것과 동일하다. 
역직렬화, 병렬화, 배치 크기 등 [Kafka Consumer Config](https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html)
에 필요한 모든 내용을 담을 수 있다. 
그리고 몇몇 설정은 `@KafkaListener` 어노테이션과 모두 중복으로 설정할 수 있다. 
앞서 제시한 코드 예시로 들면 `Consumer Group` 아이디의 경우 `Consumer Factory` 는 `exam-consumer-gorup-2` 로 했고, 
`@KafkaListener` 는 `exam-consumer-group` 으로 했다. 
이렇게 중복으로 설정된 경우 `@KafkaListener` 의 값을 사용하기 때문에 실제 설정되는 `Consumer Group` 아이디는 `exam-consumer-group` 이 된다.  


### Producing Message
#### KafkaTemplate
`Spring Kafka` 는 `Kafka Broker` 에게 메시지를 전송하기 위해 
`Producer API` 의 추상화된 구현체는 `KafkaTemplate` 을 제공한다. 
이는 낮은 수준의 추상화를 통해 사용자가 직접 메시지를 전송할 수 있도록 한다.  

```java
@Bean
public KafkaTemplate kafkaTemplate(final ProducerFactory producerFactory) {
   return new KafkaTemplate<>(producerFactory);
}
```  

[KafkaTemplate](https://docs.spring.io/spring-kafka/docs/2.6.9/api/org/springframework/kafka/core/KafkaTemplate.html)
에는 `Overload` 된 다양한 종류의 `send()` 메서드를 제공한다. 
필요에 따라 적합한 메서드를 사용해서 `Kafka Broker` 에게 메시지를 전송 할 수 있다. 
아래 예시는 `ProducerRecord` 를 사용해서 `send()` 메시지를 사용하는 예시이다.  

```java
final ProducerRecord record = new ProducerRecord<>(properties.getOutboundTopic(), key, payload);
final SendResult result = (SendResult) kafkaTemplate.send(record).get();
final RecordMetadata metadata = result.getRecordMetadata();
```  

`send()` 는 비동기 방식으로 동작하기 때문에, 
반환 값이 `ListenableFuture`([3.0](https://docs.spring.io/spring-kafka/docs/3.0.7/api/org/springframework/kafka/core/KafkaTemplate.html) 부터는 `CompletableFuture`) 이다. 
그러므로 메시지 전송을 동기적으로 하고 싶다면 아래와 같이 `send().get()` 을 호출하면 된다. 
반환 결과는 [SendResult](https://docs.spring.io/spring-kafka/docs/current/api/org/springframework/kafka/support/SendResult.html)
인데 여기에는 `Kafka Broker` 가 확인한 레코드의 메타데이터 및 메시지가 기록된 토픽, 파티션, 타임스탬프 등의 정보가 포함돼 있다.  
