--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams DSL"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka 기반 스트림 애플리케이션을 간편하게 구현할 수 있도록 도와주는 Streams DSL 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Kafka Streams DSL
    - Streams DSL
    - KStream
    - KTable
    - GlobalKTable
    - join
toc: true
use_math: true
---  

## Streams DSL
`Kafka Streams DSL` 은 `Processor API` 를 사용해서 구현 돼있고, 
대부분의 데이터 처리를 `DSL` 을 사용하면 초보자도 간단하게 표현 할 수 있다. 

`Streams DLS` 을 `Processor API` 와 비교하면 아래와 같다. 

- `Streams DSL` 은 `KStream`, `KTable`, `GlobalKTable` 과 같은 스트림과 테이블의 추상화 구현체르 제공한다. 
- `Streams DSL` 은 `stateless transformation`(`map`, `filter`, ..) 와 `stateful transformation`(`count`, `reduce`, `join`, ..) 에 대한 동작을 함수형 스타일로 제공한다. 

`Streams DLS` 을 사용해서 `Topology` 를 구성하는 절차는 아래와 같다. 
1. `Kafka Topic` 에서 데이터를 읽은 `Input Stream` 을 하나 이상 생성한다. 
2. `Input Stream` 을 처리하는 `Transformation` 을 구성한다. 
3. `Ouput Stream` 을 생성해서 결과를 `Kafka Topic` 에 저장한다. 혹슨 `Interactive queries` 를 사용해서 결과를 `REST API` 와 같은 것으로 노출 시킨다. 

`Streams DSL` 에는 레코드의 흐름을 추상화한 3가지 개념인 `KStream`, `KTable`, `GlobalKTable` 이 있다. 
위 3가지 개념은 `Consumer`, `Producer`, `Processor API` 에는 사용되지 않는 `Streams DSL` 에만 사용되는 개념이다.  

### KStream (input topic -> KStream)
`input topic` 역할을 하는 `Kafka Topic` 에 `KStream` 을 생성해서 데이터를 읽어올 수 있다. 
`KStream` 은 분할된 레코드 스트림이다. 
여기서 분할된 레코드 스트림이란 스트림 애플리케이션이 여러 서버에 실행 된다면, 
각 애플리케이션의 `KStream` 은 `input topic` 의 특정 파티션 데이터로 채워진다. 
분할된 `KStream` 을 하나의 큰 집합으로 본다면 `input topic` 의 모든 파티션의 데이터가 처리된다.  

`KStream` 은 레코드의 흐름을 표현한 것으로 메시지 키와 메시지 값으로 구성돼 있다. 
이를 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-1.drawio.png)  


### KTable (input topic -> KTable)
`KTable` 은 `KStream` 과 다르게 메시지 키를 기준으로 묶어 사용한다. 
`KStream` 은 토픽의 모든 레코드를 조회 할 수 있었지만, `KTable` 은 유니크한 메시지 키를 기준으로 
가장 최신 레코드만 조회 가능하다. 
즉 `KTable` 로 데이터를 조회하면 메시지 키를 기준으로 가장 최신에 추가된 레코드의 데이터가 출력된다. 
새로 데이터를 적재할 때 동일한 케시지 키가 있을 경우 데이터가 업데이트되었다고 볼 수 있다.  

`KStream` 과 동일하게 `Kafka Topic` 에 `KTable` 을 생성할 수 있다. 
`KTable` 은 메시지 키의 최신 데이터만 유지하므로 `Changelog Stream` 이라고 해석 할 수 있다. 
메시지 키가 존지하지 않으면 `INSERT`, 이미 동일한 메시지 키가 존재하면 `UPDATE`, 
메시지 키가 존재하지만 메시지 값이 `null` 이라면 `DELETE` 가 수행된다. 
토픽으로 부터 데이터를 읽어올 때 `auto.offset.reset` 속성에 따라 데이터를 읽어오는 위치를 정하게 된다. 
그러므로 `auto.offset.reset` 에 따라 출력되는 결과는 달라질 수 있다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-2.drawio.png)  


### GlobalKTable
`GlobalKTable` 은 `KTable` 과 동일하게 메시지 키를 기준으로 묶어 최신 값만 사용한다. 
`KTable` 로 선언된 토픽은 1개 파티션이 1개 태스크에 할당되어 사용된다. 
하지만 `GlobalKTable` 로 선언된 토픽은 모든 파키션 데이터가 각 태스크에 할당되어 사용된다.  

`GlobalKTable` 의 용도를 가장 잘 설명할 수 있는 예는 `KStream` 과 `KTable` 데이터를 `Join` 하는 동작 수행이다. 
`KStream` 과 `KTable` 을 조인하기 위해서는 코파티셔닝(`co-partitioning`)돼야 한다. 
코파티셔닝이란 조인하는 2개 데이터의 파티션 개수가 동일하고 파티셔닝 전략(`partitioning strategy`)을 동일하게 맞추는 작업이다. 
파티션 개수, 파티셔닝 전략이 동일하면 동일한 메시지 키가 동일한 테스크에 들어가는 것을 보장되기 때문에 `KStream` 과 `KTable` 조인이 가능하다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-3.drawio.png)  

문제는 조인을 수행하려면 `KStream`, `KTable` 2개 토픽이 파티션 개수 및 전략이 다를 수 있는데(코파티셔닝 되지 않음), 
이런 상황에서 `Streams Application` 에서는 `TopologyException` 이 발생한다. 
이런 상황에서는 리파티셔닝(repartitioning`) 과정을 거쳐야 하는데, 
리파티셔닝은 새로운 토픽에 새로운 메시지 키를 가지도록 재배열 하는 과정을 의미힌다.  

코파티셔닝 되지 않은 `KStream` 과 `KTable` 을 조인해서 사용하고 싶다면 `KTable` 대신 `GlobalKTable` 을 사용하면, 
위 복잡한 과정없이 간단하게 사용할 수 있다. 
`GlobalKTable` 로 정의된 데이터는 `Streams Application` 의 모든 태스크에 동일하게 공유돼 사용되기 때문에 별도 작업없이 조인 동작이 가능하다.  

하지만 `GlobalKTable` 은 추가적인 부하를 가져다 줄 수 있기 때문에 사용에 주의가 필요하다. 
각 태스크마다 `GlobalKTable` 로 정의된 모든 데이터를 지정하고 사용하기 때문에 
`Streams Application` 의 로컬 스토리지의 사용량 증가, 네트워크, 브로커 부하가 생길수 있다. 
그러므로 용량이 작은 데이터인 경우에만 사용을 하는게 좋다. 
비교적 많은 양의 데이터를 가진 토픽을 조인해야 하는 경우라면 리파티셔닝을 통해 `KTable` 을 하는 것을 권장한다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-4.drawio.png)  


### Streams DSL 옵션
`Streams DSL` 에는 필수 옵션과 선택 옵션이 있다. 
필수 옵션은 반드시 설정해야 하는 옵션이고, 
선택 옵션은 사용자의 설정을 필수로 받지 않는다. 
하지만 선택 옵션은 기본값으로 설정되기 때문에 옵션을 설정할 때 기본값을 잘 파악해서 설정해야 한다.  

아래 설명한 옵션외 추가적인 옵션은 [여기](https://kafka.apache.org/documentation/#streamsconfigs)
에서 확인 가능하다.  



#### 필수 옵션
- `bootstrap.servers` : 프로듀서가 데이터를 전송할 대상 카프카 클러스터에 속한 브로커의 호스트를 1개 이상 작성한다. (이름:포트)
- `application.id` : `Streams Application` 을 구분하기 위한 고유한 아이디를 설정한다. 다른 로직을 가진 애플리케이션이라면 다른 `application.id` 를 가져야 한다. 

#### 선택 옵션
- `default.key.serde` : 레코드의 메시지 키를 직렬화, 역직렬화하는 클래스를 지정한다. 기본값은 바이트 동작인 `Serdes.ByteArray().getClass().getName()` 이다. 
- `default.value.serde` : 레코드의 메시지 값을 직렬화, 역직렬화 하는 클래스를 지정한다. 기본값은 바이트 동작인 `Serdes.ByteArray().getClass().getName()` 이다. 
- `num.streams.threads` : 스크림 프로세싱 실행 시 실행될 스레드 개수를 지정한다. 기본값은 1이다. 
- `state,dir` : `rocksDB` 저장소가 위치할 디렉토리를 지정한다. 기본값은 `/tmp/kafka-streams` 이다. 리얼 환경에서는 `/tmp` 디렉토리가 아닌 별도로 관리되는 디렉토리를 지정해야 안전한 데이터 관리가 가능하다.  


### Streams DSL 예제
`StreamsDSL` 을 사용해서 간단한 구현예제를 살펴본다. 
`StreamsDSL` 에서 제공하는 메소드를 사용해서 몇가지 프로세싱 직접 구현해보며 사용방법에 대해 알아보도록 한다. 
`Kafka` 는 `docker-compose` 를 사용해서 구성하는데 그 내용은 아래와 같다. 

```yaml
# docker-compose.yaml 

version: '3'

services:
  zookeeper:
    container_name: myZookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"

  kafka:
    container_name: myKafka
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
```  

실행은 아래 명령을 통해 수행할 수 있다.  

```bash
$ docker-compose up --build
[+] Running 2/0
 ⠿ Container myKafka      Created 0.0s
 ⠿ Container myZookeeper  Created 0.0s
Attaching to myKafka, myZookeeper

.. 생략 ..
```  

공통으로 사용할 수 있는 `build.gradle` 내용은 아래와 같다.  

```groovy

dependencies {
    implementation 'org.slf4j:slf4j-simple:1.7.30'
    implementation 'org.apache.kafka:kafka-streams:3.2.0'
    implementation 'org.apache.kafka:kafka-clients:3.2.0'
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.1'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.1'
}
```  

#### KStream stream(), to()
`StreamDSL` 을 사용해서 가장 간단하게 구현할 수 있는 프로세싱 동작은 
특정 토픽의 데이터를 다른 토픽으로 전달하는 것이다. 
구현 예제에서는 `my-log` 토픽 데이터를 다른 토픽인 `my-log-copy` 토픽으로 전달하는 동작을 수행한다. 
`stream()` 메서드를 통해 특정 토픽의 데이터를 가져오고, `to()` 메서드로 다른 토픽으로 데이터를 전달 한다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-5.drawio.png)  


```java
public class SimpleKafkaStreams {
    private static String APPLICATION_NAME = "streams-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String MY_LOG = "my-log";
    private static String MY_LOG_COPY = "my-log-copy";

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        KStream<String, String> streamLog = builder.stream(MY_LOG);

        streamLog.to(MY_LOG_COPY);

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```  

먼저 `my-log` 토픽을 아래 명령으로 생성해준다. 

```bash
$ docker exec -it myKafka kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 3 --topic my-logç
```  

그리고 `SimpleKafkaStreams` 애플리케이션을 먼저 실행해 준다.  

프로듀서로 `my-log` 토픽에 데이터를 넣은 다음

```bash
$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic my-log 
>log-1
>log-2
>log-3
```  

`my-log-copy` 토픽에 있는 데이터를 컨슈머로 확인하면 아래와 같이 `my-log` 토픽 데이터가 `my-log-copy` 로 전달 된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-log-copy --from-beginning
log-1
log-2
log-3
```  

#### KStream filter()
앞선 예제에서 `stream()` 과 `to()` 를 사용해서 
데이터를 토픽에서 다른 토픽으로 복사하는 방법에 대해 알아보았다. 
하지만 이는 데이터를 처리하는 동작인 스크림 프로세서가 없었다.  

이번에는 토픽으로 들어오는 데이터 중 문자열 포맷이 정해진 포맷과 맞는 경우만 필터링 하는 스트림 프로세서를 추가로 구현해 본다. 
이러한 동작을 위해서는 `filter()` 메소드를 사용하면 되는데 전체적인 흐름은 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-6.drawio.png)  

스트림 프로세서에서는 중간에 `-` 문자열이 포함된 경우만 `my-log-filter` 토픽으로 데이터를 전달 한다.  

```java
public class KafkaStreamsFilter {
    private static String APPLICATION_NAME = "streams-filter-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String MY_LOG = "my-log";
    private static String MY_LOG_FILTER = "my-log-filter";

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        KStream<String, String> streamLog = builder.stream(MY_LOG);
//        KStream<String, String> filteredStream = streamLog
//                .filter((key, value) -> value.split("-").length >= 2);
//        filteredStream.to(STREAM_LOG_FILTER);
//        or
        streamLog.filter((key, value) -> value.split("-").length >= 2).to(MY_LOG_FILTER);

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```  

애플리케이션을 실행한 다음 아래 명령으로 `my-log` 토픽에 데이터를 생성해준다.  

```bash
$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic my-log                      
>log1
>log-2
>log3
>log-4
>log5
>log-6
>log-7

```  

그리고 `my-log-filter` 토픽에 있는 데이터를 아래 컨슈머 명령으로 확인하면 `-` 포함된 데이터들만 필터링돼 전달된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-log-filter --from-beginning
log-2
log-4
log-6
log-7

```  


#### KTable, KStream join()
`KTable` 과 `KStream` 는 키를 기준으로 조인할 수 있다. 
기존 데이터베이스의 조인과 다른 점은 데이터베이스는 저장된 데이터를 조인하지만, 
`Kafka` 에서는 실시간으로 들어오는 데이터를 조인해서 사용할 수 있다. 
이러한 특징으로 조인을 위해 데이터를 한번 저장할 필요 없이 이벤트 기반 스트리밍 데이터 파이프라인상에서 바로 조인을 수행 할 수 있다.  

아래는 `order` 토픽으로 들어오는 주문 내역을 `address` 주소 토픽에 있는 데이터와 조인하는 예제이다. 
`order` 에 `jack:pc` 라는 데이터가 들어오면 `address` 에 있는 `jack:seoul` 라는 데이터와 조인해서 
최종적으로 `pc, go to the seoul` 이라는 데이터를 `order-address` 토픽에 추가한다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-7.drawio.png)  

```java
public class KStreamKTableJoin {
    private static String APPLICATION_NAME = "order-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String ADDRESS_TABLE = "address";
    private static String ORDER_STREAM = "order";
    private static String ORDER_ADDRESS_JOIN_STREAM = "order-address";

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        KTable<String, String> addressTable = builder.table(ADDRESS_TABLE);
        KStream<String, String> orderStream = builder.stream(ORDER_STREAM);

        orderStream.join(
                addressTable,
                (orderStreamValue, addressTableValue) -> orderStreamValue + ", go to the " + addressTableValue)
                .to(ORDER_ADDRESS_JOIN_STREAM);

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```  

`KTable` 과 `KStream` 을 조인할떄 가장 중요한 것은 코파티셔닝이 돼 있는지 확인하는 것이다. 
앞서 설명한 것과 같이 코파티셔닝이 돼 있지 않다면 `TopologyException` 이 발생 하기 때문에 주의해야 한다. 
코파티셔닝을 위해서는 `KTable`, `KStream` 에서 사용할 토픽을 생성할 때 파티셔닝 전략과 파티션 수를 동일하게 맞추는 것이다. 
예제에서는 파티셔닝 전략은 기본을 사용하고, 파티셔닝 수는 3으로 아래 명령을 통해 생성한다.  

```bash
$ docker exec -it myKafka kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 3 --topic address
Created topic address.

$ docker exec -it myKafka kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 3 --topic order
Created topic order.

$ docker exec -it myKafka kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 3 --topic order-address
Created topic order-address.
```  

이후 애플리케이션을 실행해주고 `KTable`, `KStream` 의 토픽에 `key:value` 구조로 데이터를 추가해 준다.  

```bash
$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic address --property "parse.key=true" --property "key.separator=:"
>jack:seoul
>peter:busan
>lisa:newyork

$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic order --property "parse.key=true" --property "key.separator=:"
>jack:pc
>peter:phone
>lisa:laptop
```  

이후 `order-address` 토픽의 데이터를 컨슈머로 조회하면 아래와 같이, `KTable`(address)과 `KSTream`(order)의 조인 결과가 들어가 있는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic order-address --property print.key=true --property key.separator=":" --from-beginning
jack:pc, go to the seoul
peter:phone, go to the busan
lisa:laptop, go to the newyork
```  

조인의 기준은 `KTable` 과 `KStream` 에서 키로 입력한 값을 기준으로 수행된 것을 확인 할 수 있다.  

`Kafka` 조인의 장점은 이후 `address` 토픽에 있는 주소가 변경된다면, 
변경된 최신 주소로 매핑해서 `order-address` 토픽에 데이터를 생성하게 된다. 
아래 명령은 `address` 에서 `peter` 의 주소를 `jeju` 로 변경하고, 
`order` 에서 `peter` 가 `monitor` 를 주문했다는 데이터를 추가하게 되면 
`order-address` 에 최신 데이터를 기준으로 조인된 결과가 바로 생성된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic address --property "parse.key=true" --property "key.separator=:"
>peter:jeju

$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic order --property "parse.key=true" --property "key.separator=:"
>peter:monitor

$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic order-address --property print.key=true --property key.separator=":" --from-beginning
peter:monitor, go to the jeju
```  

#### GlobalKTable, KStream join()
`KTable`, `KStream` 의 조인은 코파티셔닝이 돼 있다는 전재를 두고 진행했다. 
하지만 조인하고자 하는 모든 토픽이 코파티셔닝이 돼 있다는 보장은 없기 때문에 이런 경우에는 `리파티셔닝` 혹은 `GlobalKTable` 을 사용 할 수 있다. 
이번에는 `GlobalKTable` 을 통해 코피타셔닝 돼있지 않은 토픽을 조인하는 예제에 대해 살펴본다.  

예제 진행을 위해서 파티션 수 3으로 앞서 생성한 `order` 토픽은 그대로 사용하고, 
`address-2` 라는 새로운 토픽을 생성하는데 파티션 수를 2로 생성한다.  

```bash
$ docker exec -it myKafka kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 2 --topic address-2
Created topic address-2.

```  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-dsl-8.drawio.png)  

```java
public class KStreamGlobalKTableJoin {
    private static String APPLICATION_NAME = "global-table-join-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String ADDRESS_GLOBAL_TABLE = "address-2";
    private static String ORDER_STREAM = "order";
    private static String ORDER_ADDRESS_JOIN_STREAM = "order-address";

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        GlobalKTable<String, String> addressGlobalTable = builder.globalTable(ADDRESS_GLOBAL_TABLE);
        KStream<String, String> orderStream = builder.stream(ORDER_STREAM);

        orderStream.join(
                        addressGlobalTable,
                        (orderKey, orderValue) -> orderKey,
                        (orderStreamKey, addressGlobalTableValue) -> orderStreamKey + ", go to the " + addressGlobalTableValue
                )
                .to(ORDER_ADDRESS_JOIN_STREAM);

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }

}
```  

애플리케이션을 실행해주고, 
`GlobalKTable` 사용목적으로 생성한 `address-2` 토픽에 이름을 키로 하고 주소를 값으로 하는 데이터를 넣어준다. 
그리고 이어서 `order` 토픽에는 이름을 키로 하고 주문 물품을 값으로 하는 데이터를 아래 명령어로 넣는다.  

```bash
$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic address-2 --property "parse.key=true" --property "key.separator=:"
>haha:london
>hoho:la

$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic order --property "parse.key=true" --property "key.separator=:"
>haha:pc
>hoho:laptop
```  

이제 `order-address` 토픽에서 결과를 확인하면, 
코파티셔닝 되지 않은 `address-2` 와 `order` 토픽을 기반으로 조인된 결과가 잘 출력되는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic order-address --property print.key=true --property key.separator=":" --from-beginning
haha:pc send to london
hoho:laptop send to la
```  

결과만 본다면 `KTable` 과 다른 점이 없는 것 같지만, 
`GlobalKTable` 토픽은 토픽에 존재하는 모든 데이터를 태스크마다 저장하고 조인 처리를 수행할 수 있다는 점이 다르다. 
그리고 조인 수행에 있어서도 `KStream` 메시지 키 뿐만아니라 메시지 값을 기준으로도 매칭해서 조인 동작을 수행 할 수 있다.  



#### KTable count()
`KTable` 의 특징 중 하나는 메시지 키를 기준으로 묶어서 사용한다는 점이 있다. 
이러한 특징을 살려 구현할 수 있는 것이 바로 `count()` 동작이다. 
이는 스트림으로 들어오는 데이터를 키를 기준으로 한다거나 하는 방법으로 `groupBy` 하고 이를 카운트 할 수 있다.  

아래 애플리케이션은 `order` 토픽에서 키가 되는 이름을 기준으로 카운트를 수행하는 스트림 애플리케이션이다.  

```java
public class KStreamCount {
    private final static Logger log = LoggerFactory.getLogger(KStreamCount.class);
    private static String APPLICATION_NAME = "stream-count-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String ORDER_STREAM = "order";

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 60000);

        StreamsBuilder builder = new StreamsBuilder();
        KStream<String, String> testStream = builder.stream(ORDER_STREAM);
        KTable<Windowed<String>, Long> countTable = testStream
                .groupByKey()
                .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofSeconds(60)))
                .count();

        countTable.toStream().foreach((key, value) -> {
            log.info(key.key() + " is [" + key.window().startTime() + " ~ " + key.window().endTime() + "] count : " + value);
        });

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```  

애플리케이션을 실행하고, `order` 토픽에 프로듀서를 사용해서 `key:value` 
로 데이터를 생성하면 애프릴케이션에서 아래와 같은 로그가 출력되는 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic order --property "parse.key=true" --property "key.separator=:"
>jack:pc
>lisa:laptop
>jack:monitor
>lisa:pc
>lisa:monitor
```  

```
[stream-count-application-StreamThread-1] INFO com.windowforsun.kafkastreamsdsl.exam.KStreamCount - jack is [2023-01-01T07:58:00Z ~ 2023-01-01T07:59:00Z] count : 2
[stream-count-application-StreamThread-1] INFO com.windowforsun.kafkastreamsdsl.exam.KStreamCount - lisa is [2023-01-01T07:58:00Z ~ 2023-01-01T07:59:00Z] count : 3
```  


#### KTable store()
`KTable` 혹은 `GlobalKTable` 은 토픽에 있는 데이터를 `store()` 메소드를 통해 다른 곳에 저장해 둘 수 있다. 
아래 애플리케이션은 `address` 토픽의 데이터를 `store()` 로 `Kafka` 외부인 스트림 애플리케이션에 저장하고 주기적으로 출력하는 예제이다.  

```java
public class KTableQueryableStore {
    private final static Logger log = LoggerFactory.getLogger(KTableQueryableStore.class);
    private static String APPLICATION_NAME = "ktable-query-store-application";
    private static String BOOTSTRAP_SERVER = "localhost:9092";
    private static String ADDRESS_TABLE = "address";
    private static boolean initialize = false;
    private static ReadOnlyKeyValueStore<String, String> keyValueStore;

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVER);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        KTable<String, String> addressTable = builder.table(ADDRESS_TABLE, Materialized.as(ADDRESS_TABLE));
        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();

        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                if(!initialize) {
                    keyValueStore = streams.store(
                            StoreQueryParameters.fromNameAndType(ADDRESS_TABLE,
                            QueryableStoreTypes.keyValueStore())
                    );
                    initialize = true;
                }

                printKeyValueStoreData();
            }
        };

        Timer timer = new Timer("Timer");
        long delay = 10000L;
        long interval = 1000L;
        timer.schedule(task, delay, interval);
    }

    static void printKeyValueStoreData() {
        log.info("=================");
        KeyValueIterator<String, String> address = keyValueStore.all();
        address.forEachRemaining(keyValueStore -> log.info(keyValueStore.toString()));
    }
}
```  

```bash
$ docker exec -it myKafka kafka-console-producer.sh --bootstrap-server localhost:9092 --topic address --property "parse.key=true" --property "key.separator=:"
>peter:jeju
>jack:seoul
>lisa:newyork
```  

```
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - =================
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - KeyValue(peter, jeju)
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - KeyValue(jack, seoul)
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - KeyValue(lisa, newyork)
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - =================
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - KeyValue(peter, jeju)
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - KeyValue(jack, seoul)
[Timer] INFO com.windowforsun.kafkastreamsdsl.exam.KTableQueryableStore - KeyValue(lisa, newyork)

.......
```  


---  
## Reference
[Streams DSL](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#transform-a-stream)  
[아파치 카프카](https://search.shopping.naver.com/book/catalog/32441032476?cat_id=50010586&frm=PBOKPRO&query=%EC%95%84%ED%8C%8C%EC%B9%98+%EC%B9%B4%ED%94%84%EC%B9%B4&NaPm=ct%3Dlct7i9tk%7Cci%3D2f9c1d6438c3f4f9da08d96a90feeae208606125%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3D60526a01880cb183c9e8b418202585d906f26cb4)  

