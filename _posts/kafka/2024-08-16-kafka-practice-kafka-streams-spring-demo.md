--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Spring Boot"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 의 특징과 장점 그리고 Spring Boot 기반 구현 예시에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Topology
    - Kafka Streams Topology
    - Consumer
    - Producer
    - Processor
    - Processor API
    - Streams DSL
toc: true
use_math: true
---  

## Kafka Streams
기존 스트림 링크 건뒤 Spring 기반 데모를 위해 한번 더 정리한다고 하면서 또 설명하기
[여기]() 
에서 `Kafka Streams` 에 대한 기본적인 개념에 대해서 알아보았다. 
`Kafka Streams Spring Boot Demo` 를 구성하기 앞서 한번 더 추가적인 내용을 집고 넘어가고자 한다.  

`Kafka Streams` 는 메시지 스트리밍 구현을 위한 다양한 `API` 와 메시지 처리, 변환, 집계와 같은 기능을 포함한다. 
그리고 필요에따라 유연하게 확장 가능한 구성과 메시지 처리의 신뢰성 그리고 유지 관리에 이점을 가져다 줄 수 있다. 
또한 실시간으로 무한히 들어오는 메시지를 낮은 지연시간을 바탕으로 빠른 처리를 가능하게 한다.

`Kafka Streams API` 는 `Consumer` 와 `Producer` 를 사용해 `Kafka Broker` 에 존재하는 `Topic` 의 메시지를
실시간으로 스트리밍하고, 처리/변환/집계 후, 다른 토픽으로 쓰는 역할을 수행한다.  
추가적으로 처리가 필요한 데이터가 `DB` 나 다른 외부 저장소에 있다면 `Kafka Connect API` 를 사용해서 외부 데이터를 `Kafka` 와 연결하는 파이프라인을 구성 할 수 있다.   

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-spring-boot-1.drawio.png)



### 특징

#### Processor Topology
`Kafka Streams` 는 `Processor Topology` 라는
하나의 스트림을 구성하는 프로세서들의 집합 개념을 통해 메시지를 처리한다.
`Processor` 에는 크게 `Source`, `Stream`, `Sink` 라는 3가지 타입의 종류가 있다.

- `Source Processor` : 토픽에서 메시지를 소비하는 역할을 하고, 소비한 데이터는 하나 이상의 `Stream` 혹은 `Sink` 프로세서에게 전달된다.
- `Stream Prcessor` : 실제 메시지를 처리하는 역할을 한다. 메시지 변환, 집계, 필터링을 수행하는데 이는 여러 단계로 구성된 `Stream Processor Chain` 을 메시지가 통과하며 수행된다. 최종 메시지는 `Sink` 프로세서에게 전달 된다.
- `Sink Processor` : `Processor Topology` 의 최종 결과를 다른 토픽에 쓰는 역할을 한다.

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-spring-boot-2.drawio.png)


`Stateful Processor` 라고 불리는 상태를 가진 프로세서는 다른 `Streams` 혹은 `Tables` 데이터와 결합하거나,
`Windowing` 을 바탕으로 데이터 그룹화를 통한 집계를 수행하는 프로세서를 의미한다.
이러한 프로세서는 추가적인 저장소를 사용해서 상태를 유지하는 특성을 가지고 있다.

그리고 하나의 `Processor Topology` 를 기준으로 각 메시지는 순차적으로 처리되는 특성을 갖는다.
이는 앞선 메시지가 처리 완료되기 전까지 다음 메시지 처리는 수행되지 않음을 의미한다.
`Topology` 는 `Sub-Topology` 를 포함 할 수 있는데,
`Sub-Topology` 와 다른 `Sub-Topology` 는 병렬로 메시지를 처리한다.


#### Tasks 와 Threads
`Kafka Streams` 에서 `Task` 와 `Threads` 는 메시지 처리의 `Throughput` 과 관련있다.
각 `Source Topic Partition` 은 각 `Task` 에 1:1 관계로 매핑된다.
그리고 각 `Task` 는 자신만의 `Topology` 을 복사본을 통해 메시지를 처리한다. (10개의 파티션이라면 10개 Task)
그리고 `Thread` 는 이러한 `Task` 를 실제로 실행하는 역할을 담당한다.
하나의 `Thread` 는 1개 이상의 `Task` 수행을 담당할 수 있고,
이러한 `Thread` 의 수는 `num.streams.threads` 설정을 통해 구성된다.
다만 `Thread` 의 수는 `Task` 의 수를 초과 할 수 없음을 기억해야 한다.

만약 `num.streams.threads=5` 이고, 10개의 파티션을 가지는 경우를 가정해보자.
그럼 `Task` 도 10개가 생성되고, 1개의 `Thread` 는 2개의 `Task` 실행을 담당하게 되는 형식이다.

#### Streams 와 Tables
`Kafka Streams Topology` 에서 메시지의 모델링은 `Stateless`(상태 비저장) 와 `Statful`(상태 저장) 로 나눠 질 수 있다.
여기서 `Stateless` 는 `Streams` 가 갖는 특성으로 각 메시지는 독립적으로 처리되는 것을 의미한다.
연속적인 데이터 플로우를 처리하지만, 특정 시점에서의 상태를 유지하지는 않는다.
대신 모든 이벤트는 개별적으로 관찰되고 처리된다.

그리고 `Statful` 는 `Tables` 가 갖는 특성으로 메시지의 최신 상태를 저징하는 것을 의마한다.
시간이 지남에 따라 발생하는 메시지를 기반으로 현재 상태를 유지한다.
`Kafka Streams` 에서 `Tables` 는 기본적으로 로컬 상태 저장소인 `RocksDB` 에 상태가 저장되고 추적된다.


#### Streams DSL 과 Processor API
`Kafka Streams` 는 2가지 `API` 인 `Streams DSL` 과 `Processor API` 를 사용해 애플리케이션을 개발 할 수 있도록 제공한다.
먼저 `Processor API` 는 저수준 `API` 로 스트림 처리를 위한 더 세밀한 제어가 필요할 떄 사용한다.
이를 사용하면 레코드 단위로 직접 작업하고 스트림의 각 메시지를 개별로 처리 할 수 있다.
좀 더 정교한 제어를 할 수 있지만, `Streams DSL` 보다 복잡성을 요구한다.

다음으로 `Streams DSL` 은 고수준의 추상화를 제공하며, 함수형 프로그래밍 스타일(map, join. filter)를 사용한다.
`Streams DSL` 은 앞서 살펴본 `Streams` 와 `Tables` 개념을 사용해서 데이터를 모델링한다.
`KStreams` 는 레코드 스트림을 나타내고, `KTable` 은 변경 가능한 데이터 집합의 최신 상태를 나타내고,
`GlobalKTable` 은 전체 데이의 집합 뷰를 나타낸다.

#### Scalability
`Kafka Streams` 는 `Kafka` 가 제공하는 확장성의 이점을 그대로 누릴 수 있다. 
작업 단위(`Task`)는 소스 토픽 파티션으로 각 작업은 분산된 애플리케이션 인스턴스 별로 분산된다. 
더 높은 확장성이 필요하다면 소스 토픽 파티션 수를 늘리는 방식으로 쉽게 적용 할 수 있다. 
`Kafka Streams` 에서 토픽의 데이터를 가져오는 것은 `Kafka Consumer` 를 사용하기 때문에, 
소스 토픽 파티션이 늘어나면 그 만큼 `Consumer Group` 을 구성하는 `Kafka Consumer` 즉 `Task` 가 늘어나게 된다.  

아래 그림을 보면 소스 토픽 파티션이 4개이고, 2개의 애플리케이션 인스턴스로 구성된다면 
각 애플리케이션 인스턴스당 2개의 `Kafka Consumer`(`Task`)로 확장되는 것이다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-spring-boot-3.drawio.png)



#### Reliability
`Kafka Streams` 는 신뢰성 측면에서도 `Kafka` 가 제공하는 신뢰성을 그대로 누릴 수 있다. 
처리되는 메시지들은 모두 `Kafka Cluster` 에 의해 복제되기 때문에 장애에 대한 강한 내성을 가지고 있다. 
만약 `Consumer` 가 실패한다면 `Consumer Group` 내 다른 `Consumer` 에게 할당되어 지속적인 메시지 소비가 가능하다. 
또한 `Kafka Cluster` 중 특정 노드의 장애상황에서도 `Kafka` 의 `failover` 를 통해 메시지 손실 위험을 최소화 할 수 있다.  


#### Maintainability
`Kafka Streams` 를 사용해서 메시지를 처리하는 것은 직관적인 `Java Library` 를 통해 구현된다. 
그러므로 `Kafka` 와 `Java` 에 대한 기반지식이 있다면 큰 러닝커브 없이 메시지 처리 스트림을 구현 할 수 있다.  



### Spring Boot Demo
앞서 우리는 `Kafka Streams API` 와 구조와 장점 등에 대해서 알아보았다. 
이번에는 `Kafka Streams API` 를 사용해서 `Stateless`, `Stateful` 메시지 처리를 구현하는 
`Spring Boot Application` 을 통해 그 방법에 대해 알아보고자 한다.  

`Spring Boot Application` 에서는 `Kafka` 에서 매출 이벤트를 수신하고, 
이를 처리하는 과정을 담고 있는데 크게 3가지로 분류 할 수 있다. 

- `Statless` 처리 : 카드 매출의 경우 카드 수수료를 계산을 수행한다. 
- `Stateful` 처리 : `RocksDB` 를 사용해서 매출 금액을 집계한다. 
- `emit event` : 최종적으로 매출 이벤트는 지정된 저장소로 보내기 위해 `Outbound` 토픽으로 전송된다.  

관련 전체 코드는 [여기]()
에서 확인 할 수 있다.  

애플리케이션에 구현한 `Kafka Streams Topology` 를 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-spring-boot-4.drawio.png)


#### Kafka Streams Configuration
아래 코드는`Kafka Streams` 의 구성요소를 설정하는 코드이다. 
아래 설정을 바탕으로 `Spring` 은 `Kafka Streams` 의 구성요소와 관련 설정을 연결하는 역할을 수행한다.  


```java
@Slf4j
@Configuration
@ComponentScan(basePackages = "com.windowforsun.kafka.streams")
@EnableKafkaStreams
public class KafkaConfig {
    @Value("${kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration kafkaStreamsConfig() {
        Map<String, Object> props = new HashMap<>();

        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "demo-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.Long().getClass().getName());

        return new KafkaStreamsConfiguration(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(final ConsumerFactory<String, String> consumerFactory) {
        final ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        return factory;
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate(final ProducerFactory<String, String> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        final Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "demo-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        final Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, this.bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        return new DefaultKafkaProducerFactory<>(props);
    }
}
```  

#### Processor
앞서 도식화한 `Topology` 를 구현하는 코드이다. 
토폴로지를 구성하고 `Inbound Topic` 으로 부터 메시지를 받아 처리 후 지정된 `Outbound Topic` 으로 전송한다.  

```java
@Autowired
public void buildPipeline(StreamsBuilder streamsBuilder) {
    KStream<String, SalesEvent> messageStreams = streamsBuilder
            // subscribe inbound topic
            .stream(this.properties.getSalesInboundTopic(), Consumed.with(STRING_SERDE, SalesSerdes.serdes()))
            .peek((k, sales) -> log.info("salesInbound {} : {}", k, sales))
            // filtering unsupported data store
            .filter((k, sales) -> SUPPORTED_DATA_STORE.contains(sales.getDataStore()))
            .peek((k, sales) -> log.info("salesInbound filtered {} : {}", k, sales));

    // branching by sales type cash or card
    Map<String, KStream<String, SalesEvent>> salesTypeBranches = messageStreams.split(Named.as("sales-type-"))
            .branch((k, sales) -> sales.getSalesType().toUpperCase().equals(SalesType.CASH.name()), Branched.as("cash"))
            .branch((k, sales) -> sales.getSalesType().toUpperCase().equals(SalesType.CARD.name()), Branched.as("card"))
            .noDefaultBranch();

    // card type charging card fee
    KStream<String, SalesEvent> feeStreams = salesTypeBranches.get("sales-type-card")
            .mapValues(sales -> SalesEvent.builder()
                    .eventId(sales.getEventId())
                    // calculate charging card fee
                    .salesAmount(sales.transformCardFee())
                    .salesType(sales.getSalesType())
                    .dataStore(sales.getDataStore())
                    .build());

    // merging card and cash branched streams
    KStream<String, SalesEvent> mergedStreams = salesTypeBranches.get("sales-type-cash")
            .merge(feeStreams)
            .peek((k, sales) -> log.info("merged sales {} : {}", k, sales));

    // aggregating total sales amounts by sales type in state store(rocksdb)
    mergedStreams
            .map((k, sales) -> new KeyValue<>(sales.getSalesType(), sales.getSalesAmount()))
            .groupByKey(Grouped.with(STRING_SERDE, LONG_SERDE))
            .aggregate(() -> 0L,
                    (k, v, agg) -> agg + v,
                    Materialized.<String, Long, KeyValueStore< Bytes, byte[]>>as("totalAmount")
                            .withKeySerde(STRING_SERDE)
                            .withValueSerde(LONG_SERDE)
            );
    
    // branching by data store db or es
    Map<String, KStream<String, SalesEvent>> dataStoreBranches = mergedStreams
            .split(Named.as("data-store-"))
            .branch((k, sales) -> sales.getDataStore().equals(DataStore.DB.name()), Branched.as("db"))
            .branch((k, sales) -> sales.getDataStore().equals(DataStore.ES.name()), Branched.as("es"))
            .noDefaultBranch();

    // send to db store outbound topic
    dataStoreBranches.get("data-store-db").to(this.properties.getDbStoreOutboundTopic(), Produced.with(STRING_SERDE, SalesSerdes.serdes()));
    // send to es store outbound topic
    dataStoreBranches.get("data-store-es").to(this.properties.getEsStoreOutboundTopic(), Produced.with(STRING_SERDE, SalesSerdes.serdes()));
    }
```  

#### Describe Topology
[Kafka Streams Topology Visualizer](https://zz85.github.io/kafka-streams-viz/)
를 사용하면 구성한 `Topology` 를 시각화해서 확인해 볼 수 있다.  

먼저 아래 코드로 구성된 `Topology` 정보를 출력한다. 

```java
@Autowired
StreamsBuilderFactoryBean factoryBean;

String topologyInfo = this.factoryBean.getTopology().describe().toString();

System.out.println(topologyInfo);
```   

```
Topologies:
   Sub-topology: 0
    Source: KSTREAM-SOURCE-0000000000 (topics: [sales-topic])
      --> KSTREAM-PEEK-0000000001
    Processor: KSTREAM-PEEK-0000000001 (stores: [])
      --> KSTREAM-FILTER-0000000002
      <-- KSTREAM-SOURCE-0000000000
    Processor: KSTREAM-FILTER-0000000002 (stores: [])
      --> KSTREAM-PEEK-0000000003
      <-- KSTREAM-PEEK-0000000001
    Processor: KSTREAM-PEEK-0000000003 (stores: [])
      --> sales-type-
      <-- KSTREAM-FILTER-0000000002
    Processor: sales-type- (stores: [])
      --> sales-type-card, sales-type-cash
      <-- KSTREAM-PEEK-0000000003
    Processor: sales-type-card (stores: [])
      --> KSTREAM-MAPVALUES-0000000007
      <-- sales-type-
    Processor: KSTREAM-MAPVALUES-0000000007 (stores: [])
      --> KSTREAM-MERGE-0000000008
      <-- sales-type-card
    Processor: sales-type-cash (stores: [])
      --> KSTREAM-MERGE-0000000008
      <-- sales-type-
    Processor: KSTREAM-MERGE-0000000008 (stores: [])
      --> KSTREAM-PEEK-0000000009
      <-- sales-type-cash, KSTREAM-MAPVALUES-0000000007
    Processor: KSTREAM-PEEK-0000000009 (stores: [])
      --> KSTREAM-MAP-0000000010, data-store-
      <-- KSTREAM-MERGE-0000000008
    Processor: data-store- (stores: [])
      --> data-store-db, data-store-es
      <-- KSTREAM-PEEK-0000000009
    Processor: KSTREAM-MAP-0000000010 (stores: [])
      --> totalAmount-repartition-filter
      <-- KSTREAM-PEEK-0000000009
    Processor: data-store-db (stores: [])
      --> KSTREAM-SINK-0000000018
      <-- data-store-
    Processor: data-store-es (stores: [])
      --> KSTREAM-SINK-0000000019
      <-- data-store-
    Processor: totalAmount-repartition-filter (stores: [])
      --> totalAmount-repartition-sink
      <-- KSTREAM-MAP-0000000010
    Sink: KSTREAM-SINK-0000000018 (topic: db-store-topic)
      <-- data-store-db
    Sink: KSTREAM-SINK-0000000019 (topic: es-store-topic)
      <-- data-store-es
    Sink: totalAmount-repartition-sink (topic: totalAmount-repartition)
      <-- totalAmount-repartition-filter

  Sub-topology: 1
    Source: totalAmount-repartition-source (topics: [totalAmount-repartition])
      --> KSTREAM-AGGREGATE-0000000011
    Processor: KSTREAM-AGGREGATE-0000000011 (stores: [totalAmount])
      --> none
      <-- totalAmount-repartition-source
```  


그리고 출력된 결과를 위 링크에 입력하면 아래와 같은 시각화된 결과를 확인할 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-spring-boot-5.png)


---  
## Reference
[Kafka Streams: Introduction](https://www.lydtechconsulting.com/blog-kafka-streams-intro.html)  
[Kafka Streams: Spring Boot Demo](https://www.lydtechconsulting.com/blog-kafka-streams-demo.html)  



