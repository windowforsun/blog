--- 
layout: single
classes: wide
title: "[Kafka] Spring Kafka Consumer & Producer Json Overview"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Spring Kafka 에서 Consumer 와 Producer 를 Json 메시지 포멧을 사용해 구현하는 예제에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Consumer
    - Producer
    - Spring Kafka
    - KafkaTemplate
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


![그림 1]({{site.baseurl}}/img/kafka/kafka-spring-consumer-producer-json.drawio.png)


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

#### Producer Factory
`KafakTemplate` 빈은 `Producer Factory` 빈을 통해 생성된다. 
아래는 `StringSerializer` 를 사용해서 메시지를 문자열로 직렬화하는 예시이다.  

```java
@Bean
public ProducerFactory producerFactory(@Value("${kafka.bootstrap-servers}") final String bootstrapServers) {
   final Map config = new HashMap<>();
   config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
   config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
   config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
   return new DefaultKafkaProducerFactory<>(config);
}
```  


`Producer Factory` 빈에는 [Kafka Producer](https://docs.confluent.io/platform/current/installation/configuration/producer-configs.html)
설정에 필요한 내용들을 지정할 수 있다. 
`Consumer Factory` 와 동일하게 직렬화, 재시도, 배치 크기, 트랜잭션 등 구현에 필요한 설정을 지정할 수 있다.  


### Kafka Message Serialization
앞서 서술한 `Kafka Consumer` 와 `Kafka Producer` 를 `Spring Kafka` 를 사용해 구성하는 내용은 
문자열 그대로 메시지를 소비/생산하는 예시였다. 
`Kafka` 를 사용해서 `MessagingStream` 을 구현할 때는 데이터를 구성하는 포맷으로 소비/생산하는게 일반적이다. 
이번에는 `Spring Kafka` 를 사용해서 문자열이 아닌 특정 포맷으로 메시지를 소비(역직렬화)/생산(직렬화)하는 법을 알아 볼 것이다. 
그 중 일반적인 메시지 포맷인 `JSON` 형식의 메시지를 사용하는 방법과 흐름에 대해 알아본다.  

#### Spring Kafka JSON Serialization
`Kafka Broker` 와 메시지를 소비/생산 할때 실제 메시지는 바이트 배열로 전송된다. 
하지만 이를 정해진 데이터 유형으로 직렬화/역직렬화해 좀 더 친화적으로 사용 할 수 있다. 
그 중 가장 보편적인 유형이 바로 `String` 타입이다. 
하지만 `String` 타입을 사용할 경우 큰 확률로 다시 다른 타입으로 직렬화/역직렬화가 필요한 경우가 많으므로, 
`JSON` 타입을 사용하면 이를 다시 `POJO` 로 변화해 비지니스 처리에 사용할 수 있다.  

```json
{
  "id" : 1,
  "name" : "jack",
  "age" : 27
}
```

위 예시와 같이 `JSON` 형태는 `String` 과 비교해서 데이터 관점에서 좀 더 읽기 쉽고, 
필드를 구분으로 다양한 데이터와 포맷을 담아 둘 수 있다는 장점이 있다.  

`Spring Kafka` 는 `Kafka Consumer` 가 토픽으로 부터 메시지 소비단계에서 
바이트 배열을 역직렬화하는 것과 `Kafka Producer` 가 토픽으로 메시지 전송 단계에서 
바이트 배열로 직렬화하는 직렬화/역직렬화 추상화를 제공한다. 
이를 통해 사용자는 필요한 직렬화 포맷만 `Kafka Consumer` 와 `Kafka Producer` 빈 생성 단계에서 
설정해주면 `Java POJO` 객체를 사용한 메시지 처리를 손 쉽게 구현 할 수 있다.  

`Spring Kafka` 를 사용해서 직렬화/역직렬화 설정은 `Properties` 파일에 지정하는 방법과 
`Java Config` 파일에서 빈 생성시 직접 지정하는 방법이 있다. 
`Properties` 로 지정하는 방법의 경우 `Spring Context` 가 로드되는 시점에 `Properties` 에 
지정된 값들을 바탕으로 `Spring Kafka` 가 필요한 기본 빈들을 생성할 때 사용된다. 
`Java Config` 로 직접 설정하는 방법의 경우 직접 모든 빈을 선언하고 설정해줘야 한다는 점이 있지만, 
컴파일 시점에 설정에 대한 유효성 검사를 할 수 있다는 장점이 있다. 
그리고 하나의 애플리케이션에서 하나 이상의 `Consumer` 와 `Producer` 를 사용할 때 더욱 유용하다.  


### Spring Kafka JSON Serialization Demo
데모 애플리케이션은 단일 `Kafka Consumer` 와 `Kafka Producer` 로 구성돼 있는데, 
이를 도식화 하면아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-spring-consumer-producer-json-2.drawio.png)


1. `Kafka Consumer` 는 `demo-inbound-topic` 으로 부터 메시지를 소비한다. 
2. `Kafka Consumer` 는 소비한 바이트 배열의 메시지의 `key/value` 를 `JsonDeserializer` 를 사용해서 프로젝트에 정의된 `POJO` 인 `InboundKey`, `InboundPayload` 클래스로 변환한다. 
3. `Kafka Producer` 는 메시지 전처리가 완료된 메시지 구성 `OutboundKey`, `OutboundPayload` 의 `POJO` 를 `JsonSerializer` 를 사용해서 바이트 배열로 직렬화 한다. 
4. `Kafka Producer` 는 최종적으로 메시지를 `demo-outbound-topic` 으로 전송한다.  

데모의 전체적인 코드는 [여기](https://github.com/windowforsun/spraing-kafka-json-message-exam)
에서 확인할 수 있다.  

#### Consumer
데모 애플리케이션에서 사용하는 `Consumer` 구현부는 아래와 같다. 
토픽과 `ConsumerGroup` 아이디, `containerFactory` 를 지정한다. 

```java
@KafkaListener(topics = "${demo.inboundTopic}",
    groupId = "demo-consumer-group",
    containerFactory = "kafkaListenerContainerFactory")
public void listen(@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) InboundKey key, @Payload InboundPayload payload) {
	// ...
}
```  

메시지의 키와 페이로드 타입은 `InboundKey`, `InboundPayload` 로 별도로 정의한 `POJO` 이다. 
이는 후술할 `Java Config` 파일에서 타입에 대한 지정을 되어있는 상태로, 
사용자는 `@KafkaListener` 어노테이션이 선언된 메서드에 타입에 맞게 사용만 하면 된다. 
이는 바이트 배열의 키와 페이로드를 `JSON` 으로 역직렬화 하고 나서, 
선언된 `POJO` 객체로 매핑하게 된다. 
이러한 내용들은 모두 `KafkaListenerContainerFactory` 에 설정된 내용을 기반으로 수행된다.  

#### KafkaListenerContainerFactory
`KafkaListenerContainerFactory` 는 별도로 `Java Config` 파일에 정의한 빈으로
`@KafkaListener` 어노테이션의 `containerFactory` 에 참조되어 사용된다. 
이를 정의하기 위해서는 `ConsumerFactory` 빈이 필요한데 `ConsumerFactory` 는 아래와 같이 
`Kafka Consumer` 의 직렬화 키/값 속성 등의 설정이 정의 돼있다.  

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<Object, Object> kafkaListenerContainerFactory(final ConsumerFactory<Object, Object> consumerFactory) {
	final ConcurrentKafkaListenerContainerFactory<Object, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
	factory.setConsumerFactory(consumerFactory);

	return factory;
}

@Bean
public ConsumerFactory<Object, Object> consumerFactory() {
    final Map<String, Object> props = new HashMap<>();

    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, JsonDeserializer.class);
    props.put(JsonDeserializer.KEY_DEFAULT_TYPE, InboundKey.class.getCanonicalName());
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);
    props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, InboundPayload.class.getCanonicalName());

    return new DefaultKafkaConsumerFactory<>(props);
}
```  

`ConsumerFactory` 정의를 보면 키/값에 대한 `DeserializerClass` 로 `ErrorHandlingDeserializer` 를 사용했다. 
그리고 해당 역직렬화 클래스는 `DefaultType` 으로 지정된 `JsonDeserializer` 에게 실제 역직렬화 동작을 위임한다. 
여기서 `ErrorHandlingDeserializer` 클래스를 사용하는 이유는 역직렬화 과정에서 발생할 수 있는 에러를 위해서다. 
만약 해당 클래스를 사용하지 않는 상황에서 `JSON` 으로 역직렬화 할 수 없는 메시지를 소비하게 되면 `Consumer` 는 예외가 발생하게 된다. 
그리고 해당 메시지가 정상적으로 소비되지 않았기 때문에 `Offset` 는 증가하지 않고 다음 폴링에도 동일한 메시지가 소비된다. 
이러한 예외 동작의 반복으로 해당 `Partition` 의 메시지 소비는 다음 메시지로 넘어가지 않고 소비가 중단 된 것 같은 `Poison Pill` 상태에 빠지게 되는 것이다.  

`ErrorHandlingDeserializer` 를 사용하면 역직렬화 과정에서 에러가 발생했을 때, 
에러 처리에 대한 구현부를 작성하고 다음 메시지를 이어서 계속 소비할 수 있어 `Partition` 의 메시지 소비가 막히는 것을 방지할 수 있다. 
여기서 에러 처리는 로깅을 남긴다던가, `dead-letter` 토픽으로 전송한다던가 하는 동작이 될 수 있다.  


아래는 `ErrorHandlingDeserializer` 를 사용 할때 역직렬화 에러 발생시 처리하는 `errorHandler` 를 사용하는 예시이다. 
`errorHandling` 구현 빈을 선언하고 `@KafkaListener` 어노테이션에 해당 빈을 설정해주면 된다. 
아래 예제는 역직렬화 에러 발생시 로깅만 남기고 다음 메시지로 넘어가게 된다.  

```java
@KafkaListener(topics = "#{'${demo.inboundTopic}'}",
    groupId = "demo-consumer-group",
    containerFactory = "kafkaListenerContainerFactory",
    errorHandler = "errorHandler")
public void listen(@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) InboundKey key, @Payload InboundPayload payload) {
	// ...
}

@Bean
public KafkaListenerErrorHandler errorHandler() {
    return (message, e) -> {
            log.error("message deserializer error message: {}, error : {}", message, e.getMessage());

            return null;
    };
}
```  

메시지의 키/값에 대한 타입 설정은 명시적으로 선언해두는 방식을 사용했다. 
명시적으로 설정하지 않고 이러한 타입 정보를 메시지 헤더에 탐아 사용 할 수도 있다. 
`Spring Kafka` 는 기본적으로 메시지 헤더에 메시지 타입에 대한 정보를 추가한다. 
만약 이러한 처리가 불필요할 경우 아래와 같이 `ProducerFactory` 에 설정 할 수 있다.  

```java
config.put(JsonSerializer.ADD_TYPE_HEADERS, false);
```  

만약 메시지 헤더에 타입정보가 존재한다면 `Consumer` 는 기본적으로 헤더의 타입정보를 사용한다. 
`ConsumerFactory` 에서 헤더에 타입정보가 존재하더라도 명시적으로 설정한 타입을 사용하도록 설정 할 수 있다.  

```java
config.put(JsonDeserializer.USE_TYPE_INFO_HEADERS, false);
```  

이런 메시지 직렬화/역직렬화에 대한 설정은 `ConsumerFactory` 에서 하는 방법도 있지만, 
아래와 같이 별도의 빈을 선언하지 않고 기본 `ConsumerFactory` 빈을 바탕으로 `Properties` 기반으로 설정 할 수 있다.  

```properties
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
spring.kafka.consumer.properties.spring.deserializer.value.delegate.class=org.springframework.kafka.support.serializer.JsonDeserializer
```  

#### Producer
애플리케이션에서 처리가 완료된 메시지를 `Outbound Topic` 으로 전송하는 `DemoProducer` 는 
미리 선언한 `KafkaTemplate` 빈을 내부적으로 사용한다. 
앞서 알아본 `Consumer` 의 `@KafkaListener` 와 유사하게 `KafkaTemplate` 는 선언시 기입한 설정(직렬화,..) 값들을 바탕으로 구성되고, 
직렬화를 위해 별도의 처리코드를 작성할 필요 없다.  

메시지를 토픽으로 전송은 아래와 같이 `KafkaTemplate` 의 `send()` 메소드에 토픽 이름, 메시지 키, 메시지 값을 전달하면 된다.  

```java
this.kafkaTemplate.send(this.properties.getOutboundTopic(), key, payload).get();
```  

`send()` 메소드가 호출되면 파라미터로 전달된 값들을 기반으로 메시지를 `Kafka` 로 전송한다. 
`send()` 메소드를 호출할 때 `get()` 를 호출해주는 것을 볼 수 있다. 
만약 `send()` 만 호출하게 된다면 `Kafka` 로 메시지 전송은 비동기로 이뤄진다. 
즉 전송의 성공여부를 기다리지 않는 `fire and forget` 으로 수행되는데, 
전송 결과 확인 및 전송 후 처리가 필요하다면 `send()` 로 번환되는 `ListenableFuture(CompletableFuture)` 를 사용해야 한다. 
`get()` 을 추가로 호출하는 경우 전송 과정은 동기적으로 수행하기 때문에 전송이 완료될 때까지 대기하게 된다. 
`get()` 이 반환하는 `SendResult` 를 사용해서 예외처리를 한다던가 재시도를 하는 등의 식의 코드를 이후에 작성 할 수 있다.  

`send()` 처리가 완료되면 `Spring Kafka` 는 `enable.auto.commit=true` 인 경우 소비한 메시지에 대한 오프셋도 커밋하여 
메시지가 정상적으로 소비되었음을 표시해 중복 처리가 되지 않도록 한다.  

#### KafkaTemplate Config
`KafkaTemplate` 또한 `Java Config` 를 통해 미리 정의된 빈으로, 
`ProducerFactory` 빈을 기반으로 생성된다. 
앞서 알아본 `ConsumerFactory` 와 동일하게 `ProducerFactory` 에는 
`Kafka Producer` 관련 설정인 직렬화, 키/값 속성등이 포함돼있다.  

```java
@Bean
public KafkaTemplate<Object, Object> kafkaTemplate(final ProducerFactory<Object, Object> producerFactory) {
    return new KafkaTemplate<>(producerFactory);
}

@Bean
public ProducerFactory<Object, Object> producerFactory() {
    final Map<String, Object> props = new HashMap<>();

    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

    return new DefaultKafkaProducerFactory<>(props);
}
```  

`Producer` 의 경우는 `Consumer` 와 다르게 전송과정에서 메시지 지결ㄹ화에 대한 에러 핸들링이 가능하기 때문에 
`ErrorHandling` 은 사용하지 않고, 
직접적으로 `JsonSerializer` 를 명시적으로 설정해서 사용한다. 
`Producer` 의 설정 또한 기본 `KafkaTemplate` 빈에 대한 설정을 `Properties` 를 기반으로 구성할 수 있다.  

#### Spring Kafka Generic Types
예제 코드에서 `Spring Kafka` 관련 모든 빈은 `DemoConfig` 라는 `Java Config` 파일에 선언돼있다. 
이때 이러한 빈들은 `Generic` 사용해서 아래와 같이 타입을 강하게 지정할 수 있다.  

```java
public ConcurrentKafkaListenerContainerFactory<DemoInboundKey, DemoInboundPayload> kafkaListenerContainerFactory(final ConsumerFactory<DemoInboundKey, DemoInboundPayload> consumerFactory) {
	// ...
}

public KafkaTemplate<DemoOutboundKey, DemoOutboundEvent> kafkaTemplate(final ProducerFactory<DemoOutboundKey, DemoOutboundEvent> producerFactory) {
    // ..
}
```  

그리고 `Object` 와 같이 지정해서 명시적으로 타입을 지정하지 않을 수도 있다. 

```java

public ConcurrentKafkaListenerContainerFactory<Object, Object> kafkaListenerContainerFactory(final ConsumerFactory<Object, Object> consumerFactory) {
	// ...
}

public KafkaTemplate<Object, Object> kafkaTemplate(final ProducerFactory<Object, Object> producerFactory) {
    // ..
}
```  

`Object` 와 같이 명시적인 타입을 지정하지 않을 경우 여러 타입에 대한 `Producer`, `Consumer` 를 생성하지 않아도 된다는 장점이 있다. 
즉 여러 메시지 타입을 소비하고 생산하는 애플리케이션에서 하나의 빈을 공유해서 재사용이 가능한 것이다.

---  
## Reference
[Kafka JSON Serialization](https://www.lydtechconsulting.com/blog-kafka-json-serialization.html)   
[Kafka Consume & Produce: Spring Boot Demo](https://www.lydtechconsulting.com/blog-kafka-consume-produce-demo.html)   

