--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Processor API"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에서 좀 더 상세한 구현이 가능한 Processor API 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Processor API
    - StateStores
toc: true
use_math: true
---  

## Processor API
`Processor API` 는 `Streams DSL` 보다 좀 더 많은 구현 코드가 요구되지만, 
토폴로지를 기준으로 데이터를 처리한다는 점에서 동일한 역할을 한다. 
`Streams DSL` 은 데이터 처리, 분기, 조인을 처리할 수 있는 다양한 메서드를 제공하지만, 
이보다 더 상세한 구현이 필요한 경우 `Processor API` 를 사용해서 구현할 수 있다. 
`Streams DSL` 에서 언급했던 것처럼 `KStream`, `KTable`, `GlobalKTable` 은 `Streams DSL` 에서만 사용되는 개념이므로 
`Processor API` 에서는 사용할 수 없다.  

그리고 `Processor API` 를 사용해서 임의의 스트림 프로세서를 정의할 수 있고, 프로세서를 연관된 `State Store` 와 연결 할 수도 있다. 
스트림 프로세서는 `Processor` 인터페이스 구현으로 정의할 수 있고, 
`Processor` 인터페이스는 `Processor API` 의 메소드를 제공하는 역할을 한다. 
아래는 `Processor` 인터페이스의 내용이다.  

```java
public interface Processor<KIn, VIn, KOut, VOut> {

    default void init(final ProcessorContext<KOut, VOut> context) {}
    
    void process(Record<KIn, VIn> record);

    default void close() {}
}
```  

`process()` 메소드는 각 `record` 별로 호출되고, 
`init()` 메소드는 `Kafka Streams` 라이브러리에 의해 `Task` 생성 단계에서 호출 된다. 
`Processor API` 구현에 있어서 초기화 작업은 `init()` 메소드에서 처리해야 한다. 
그리고 `init()` 메소드는 인자로 `ProcessorContext` 객체를 전달 받는데, 
`ProcessorContext` 는 현재 처리중인 `recrod` 의 메타데이터를 가져 올 수 있고, 
`ProcessorContext.forward()` 메소드를 통해 다운 스트림으로 새로운 `record` 를 전달 할 수 있다.  

`ProcessorContext.schedule()` 메소드는 일정 주기로 특정 메소드를 실행할 때 사용 할 수 있다. 

```java
public interface ProcessorContext<KForward, VForward> {
    /**
     * @param interval the time interval between punctuations (supported minimum is 1 millisecond)
     * @param type one of: {@link PunctuationType#STREAM_TIME}, {@link PunctuationType#WALL_CLOCK_TIME}
     * @param callback a function consuming timestamps representing the current stream or system time
     * @return a handle allowing cancellation of the punctuation schedule established by this method
     * @throws IllegalArgumentException if the interval is not representable in milliseconds
     */
    Cancellable schedule(final Duration interval,
                         final PunctuationType type,
                         final Punctuator callback);
}
```  

`ProcessorContext.schedule()` 메소드는 인자로 `Puncuator` 콜백 인터페이스를 받는데 그 구현은 아래와 같다.  

```java
public interface Punctuator {

    /**
     * @param timestamp when the operation is being called, depending on {@link PunctuationType}
     */
    void punctuate(long timestamp);
}
```  

`Punctuator` 의 `puntuate()` 메소드는 스케쥴링에서 어떤 시간 개념을 사용할지를 의미하는 `PuntuationType` 을 기반해서 주기적으로 호출된다. 

```java
public enum PunctuationType {
   STREAM_TIME,
   WALL_CLOCK_TIME,
}
```  

- stream-time(default) : 이벤트 시간 사용. `puntuate()` 메소드는 `recrod` 에 의해서 트리거 된다. `recrod` 의 타임스템프 값으로 주기가 결졍되기 때문이다. 만약 `recrod` 가 들어오지 않으면 `puntuate()` 는 호출되지 않는다. 그리고 `recrod` 를 처리하는데 걸리는 시간과 관계없이 `puntuate()` 메소드를 트리거 한다. 
- wall-clock-time : 실제 시간으로 트리거 한다. 

### 예제 환경 구성
예제 진행을 위해 로컬에서 `Kafka` 실행과 애플리케이션의 `build.gradle` 내용에 대해서만 간단하게 나열한다.  

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

```bash
$ docker-compose up --build

.. Kafka, Zookeeper 실행 ..
```  

`Processor API` 는 `Streams DSL` 에서 동일하게 사용한 `kafka-streams` 라이브러리를 사용한다.
`build.gradle` 내용은 아래와 같다.

```groovy
plugins {
    id 'java'
}

group 'org.example'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.slf4j:slf4j-simple:1.7.30'
    implementation 'org.apache.kafka:kafka-streams:3.2.0'
    implementation 'org.apache.kafka:kafka-clients:3.2.0'
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.1'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.1'


    compileOnly "org.projectlombok:lombok"
    testCompileOnly "org.projectlombok:lombok"
    annotationProcessor "org.projectlombok:lombok"
    testAnnotationProcessor "org.projectlombok:lombok"
}

test {
    useJUnitPlatform()
}
```  

### Filter 예제
토픽 데이터 중 문자에 `-` 가 포함된 데이터만 필터링 해서 다른 토픽으로 저장하는 애플리케이션을 개발해 본다. 
애플리케이션 구현을 위해 `FilterProcessor` 와 `WordFilterKafkaProcessor` 2개의 클래스를 작성한다. 
`Streams DSL` 에서는 기본으로 제공하는 `filter()` 메소드를 사용해서 구현할 수 있었지만, 
`Processor API` 에서는 스트림 프로세서 역할을 하는 클래스를 직접 만들어야 하고 이 역할을 `FilterProcessor` 가 담당한다.  

```java
public class FilterProcessor implements Processor<String, String, String, String> {
    private ProcessorContext<String, String> context;

    @Override
    public void process(Record<String, String> record) {
        if(record.value().split("-").length > 1) {
            this.context.forward(record);
        }

        context.commit();
    }

    @Override
    public void init(ProcessorContext<String, String> context) {
        this.context = context;
    }

    @Override
    public void close() {
        Processor.super.close();
    }
}
```  

그리고 `FilterProcessor` 를 사용해서 실제 `Kafka Streams` 에 토폴로지를 구성하는 `WordFilterKafkaProcessor` 내용은 아래와 같다.  

```java
public class WordFilterKafkaProcessor {
    private static String APPLICATION_NAME = "processor-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String MY_LOG = "my-log";
    private static String MY_LOG_FILTER = "my-log-filter";

    public static void main(String[] args) {

        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        Topology topology = new Topology();
        topology.addSource("Source",
                        MY_LOG)
                .addProcessor("Process",
                        () -> new FilterProcessor(),
                        "Source")
                .addSink("Sink",
                        MY_LOG_FILTER,
                        "Process");

        KafkaStreams streaming = new KafkaStreams(topology, props);
        streaming.start();
    }
}
```  

애플리케이션 코드 작성이 완료 됐으면, 실행 전 아래 명령으로 소스 토픽인 `my-log` 를 생성해 준다.  

```bash
$ docker exec -it myKafka kafka-topics.sh \
--create \
--bootstrap-server localhost:9092 \
--partitions 3 \
--topic my-log
```  

그리고 애플리케이션을 실행해주고, 프로듀서로 아래와 같이 데이터를 토픽으로 넣어준다.  

```bash
docker exec -it myKafka kafka-console-producer.sh \
--bootstrap-server localhost:9092 \
--topic my-log
> helloworld
> hello-world
> kafka-streams
> kafkastreams
```  

컨슈머로 `my-log-filter` 토픽의 내용을 확인해 보면 `-` 가 포함된 데이터들만 필터링 되어 들어간 것을 확인 할 수 있다.  

```bash
docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic my-log-filter
--from-beginning
hello-world
kafka-streams
```  

### StateStore
`Stateful` 한 `Processor`, `Transformer` 구현을 위해서는 `StateStore` 가 1개 이상 필요하다. 
`StateStore` 은 최근 인입된 레코드 혹은 인입된 레코드의 중복 제거 등 다양한 용도로 상태관리가 필요한 상황에서 사용될 수 있다. 
그리고 `StateStore` 의 데이터는 외부 애플리케이션에서 쿼리해서 사용하는 것도 가능하다.  

`StateStore` 은 서로다른 특징이 있는 구현체를 제공한다. 

- `KeyValueStore`
- `SessionStore`
- `WindowStore`

그리고 위 구현체는 아래 2가지로 저장방식을 선택할 수 있다.  

- `Persistent`
  - `Rocks DB Storage`, `Fault-tolerant` 를 기본으로 사용한다. 
  - 데이터를 로컬 디스크에 저장한다.
  - 저장되는 데이터의 크기는 애플리케이션 메모리(힙) 크기 보다 커질 수 있다. 하지만 로컬 디스크 보다는 커질 수 없다. 
- `Inmemory`
  - `Fault-tolerant` 를 기본으로 사용한다. 
  - 데이터를 메모리에 저장한다. 
  - 저장되는 데이터의 크기는 애플리케이션 메모리(힙) 크기 보다 커질 수 없다. 
  - 애플리케이션이 로컬 디스크를 사용 못하는 환경에서 사용 할 수 있다. 

`StateStore` 를 생성하는 방법은 아래와 같다.  

```java
StoreBuilder<KeyValueStore<String, String>> persistentBuilder = Stores.keyValueStoreBuilder(
        Stores.persistentKeyValueStore("my-persistent-store"), 
        Serdes.String(), 
        Serdes.String()
);

StoreBuilder<KeyValueStore<String, String>> inMemoryBuilder = Stores.keyValueStoreBuilder(
        Stores.inMemoryKeyValueStore("my-in-memory-store"),
        Serdes.String(),
        Serdes.String()
);
```  

### Fault-tolerant State Stores
- `Fault-tolerant` 를 설정하게 되면 데이터 손실 없이 `State Store` 을 마이그레이션해서 지속적으로 카프카 토픽에 백업 가능하다. 
- 여기서 사용되는 토픽을 `changelog topic(changelog)` 라고 한다. 
- 애플리케이션이 비정상 종료되면 `State Store` 와 애플리케이션의 상태는 `changelog` 를 통히 완전히 복구 될 수 있다. 
- `changelog` 토픽은 `compacted` 토픽으로 토픽의 데이터가 무한히 증가하는 것을 방지하고, 복구 시간을 최소화 한다. 
- `WindowStores` 도 `Fault-tolerant` 한데 `키 + window 타임스탬프` 와 같은 키 구조로 인해 `compaction + deletion` 의 조합을 사용한다. 
- `deletion` 을 통해 `WindowStores` 에서 만료된 키들을 삭제 한다. (보관 설정은 `Windows.maintainMs() + 1`)
  - `StreamsConfig.WINDOW_STORE_CHANGE_LOG_ADDITIONAL_RETENTION_MS_CONFIG` 을 통해 설정 가능하다. 
- `Fault-tolerant` 의 활성화 비활성화는 `enableLogging()`, `disableLogging()` 을 통해 가능하다. 

  ```java
        StoreBuilder<KeyValueStore<String, String>> persistentBuilder = Stores.keyValueStoreBuilder(
                        Stores.persistentKeyValueStore(
                                "my-persistent-store"),
                        Serdes.String(),
                        Serdes.String()
                )
                .withLoggingDisabled();
  ```  

- 커스텀한 `Fail-tolerant` 설정은 아래와 같이 가능하다. 

    ```java
        StoreBuilder<KeyValueStore<String, String>> persistentBuilder = Stores.keyValueStoreBuilder(
                        Stores.persistentKeyValueStore(
                                "my-persistent-store"),
                        Serdes.String(),
                        Serdes.String()
                )
                .withLoggingEnabled(Map.of("min.insyc.replicas", "1"));
    ```  



### StateStore 사용하기
`my-log`(`Source`) 토픽으로 문자열이 몇번 인입됬는지 `Processor API` 와 `StateStore` 를 사용해서 카운트하는 스트림 애플리케이션을 개발해본다. 
만약 아래와 같이 `my-log` 에 문자열이 들어오면 

```
hello
world
helloworld
hello
hello
helloworld
```  

`my-log-count`(`Sink`) 토픽에는 `StateStore` 에 저장되 있는 데이터들이 모두 아래와 같이 들어가게 된다. 

```
hello:3
world:1
helloworld:2
```  

`WordCountProcessor` 는 `Source` 토픽으로 부터 들어오는 문자열이 몇번 들어 왔는지 `StateStore` 를 사용해서 카운트 한 후 `StateStore` 에 갱신하고, 
그 정보들을 `Sink` 토픽으로 `forward`(전달) 한다. 

```java
public class WordCountProcessor implements Processor<String, String, String, String> {
    private ProcessorContext<String, String> context;
    private KeyValueStore<String, Long> keyValueStore;

    @Override
    public void process(Record<String, String> record) {
        String word = record.value();
        Long wordCount = this.keyValueStore.get(word);

        if(wordCount == null) {
            this.keyValueStore.put(word, 1L);
        } else {
            this.keyValueStore.put(word, wordCount + 1);
        }

        KeyValueIterator<String, Long> iter = this.keyValueStore.all();

        while(iter.hasNext()) {
            KeyValue<String, Long> entry = iter.next();
            this.context.forward(new Record<>(entry.key, entry.value.toString(), System.currentTimeMillis()));
        }

        iter.close();

        this.context.commit();
    }

    @Override
    public void init(ProcessorContext<String, String> context) {
        this.context = context;
        this.keyValueStore = this.context.getStateStore("word-count");
    }

    @Override
    public void close() {
        this.keyValueStore.close();
    }
}
```  

`WordCountKafkaProcessor` 는 구현에 필요한 `Topology` 를 구성과 `StateStore` 생성한다.    

```java
public class WordCountKafkaProcessor {

    private static String APPLICATION_NAME = "processor-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String MY_LOG = "my-log";
    private static String MY_LOG_COUNT = "my-log-count";

    public static void main(String[] args) {

        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        Topology topology = new Topology();
        topology.addSource("Source",
                        MY_LOG)
                .addProcessor("Process",
                        () -> new WordCountProcessor(),
                        "Source")
                .addStateStore(Stores
                        .keyValueStoreBuilder(Stores.persistentKeyValueStore("word-count"), Serdes.String(), Serdes.Long()), "Process")
                .addSink("Sink",
                        MY_LOG_COUNT,
                        "Process")
        ;

        KafkaStreams streaming = new KafkaStreams(topology, props);
        streaming.start();
    }

}
```  

스트림 애플리케이션 실행 전 우선 `Source` 토픽인 `my-log` 를 피티션 개수 1개로 생성해 준다. 
파티션을 1개만 사용하는 이유는 원활한 예제 진행을 위해서이다. 
`StateStore` 는 파티션 수만큼 생성되기 때문에 1개 이상의 파티션으로 토픽을 생성하는 경우 예제 테스트 결과에 혼란을 줄 수 있다. 

```bash
$ docker exec -it myKafka kafka-topics.sh \
--create \
--bootstrap-server localhost:9092 \
--partitions 1 \
--topic my-log
```  

애플리케이션을 실행해 준 뒤 프로듀서를 사용해서 `Source` 토픽에 아래와 같이 문자열을 넣어 준다.  

```bash
$ docker exec -it myKafka kafka-console-producer.sh \
--bootstrap-server localhost:9092 \
--topic my-log
>hello
>helloworld
>world
>helloworld
>world
>helloworld
>helloworld
```  

그리고 `Sink` 토픽을 컨슈머로 조회하면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ docker exec -it myKafka kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic my-log-count \
--property print.key=true \
--property key.separator=":" \
--from-beginning
hello:1
helloworld:4
world:2
```  


---  
## Reference
[PROCESSOR API](https://kafka.apache.org/21/documentation/streams/developer-guide/processor-api.html)  
[아파치 카프카](https://search.shopping.naver.com/book/catalog/32441032476?cat_id=50010586&frm=PBOKPRO&query=%EC%95%84%ED%8C%8C%EC%B9%98+%EC%B9%B4%ED%94%84%EC%B9%B4&NaPm=ct%3Dlct7i9tk%7Cci%3D2f9c1d6438c3f4f9da08d96a90feeae208606125%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3D60526a01880cb183c9e8b418202585d906f26cb4)  

