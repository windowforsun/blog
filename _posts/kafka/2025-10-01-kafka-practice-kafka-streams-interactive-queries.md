--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Interactive Queries"
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
    - Kafka Streams
toc: true
use_math: true
---  

## Kafka Streams Interactive Queries
`Kafka Streams` 애플리케이션에서 관리되는 `State Store` 즉 상태는 
여러 애플리케이션 인스턴스에 분산돼 각 인스턴스의 로컬에서 관리된다. 
그 구조를 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-interactive-queries-1.drawio.png)

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-interactive-queries-2.drawio.png)

![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-interactive-queries-3.drawio.png)


그리고 이런 `State Store` 를 별도로 목적에 따라 조회하는 것을 `Interactive Quries` 라고 한다. 
이러한 구조를 갖는 `Kafka Streams` 에서 `State Store` 의 쿼리는 아래와 같이 `Local State` 와 `Remove State` 로 구분될 수 있다.  

- `Local State` : 애플리케이션 인스턴스 로컬에서 관리되는 전체 상태의 일부를 의미한다. 이런 로컬 상태는 직접 쿼리해 필요한 경우 바로 사용할 수 있다. 다만 여기서 별도로 로컬 상태에 접근하는 것은 `read-only` 모드로 일기 전용임을 기억해야 하고, 로컬 상태의 변경은 `Kafka Streams API` 에 의해서만 가능하다. 
- `Remote State` : `Consumer Group` 에서 사용하는 각 `State Store` 의 전체 상태를 의미한다. 이는 여러 `Local State` 를 연결해야 하기 때문에 `Local State` 쿼리, `Network` 기반 모든 애플리케이션 인스턴스 로컬 저장소 탐색, 모든 애플리케이션과 네트워크 통신의 구성이 필요하다. 

정리하면 `Local State` 는 `Kafka Streams` 에서 기본적으로 제공하는 `API` 를 통해 `Local State Store` 에 대한 `Interactive Queries` 가 가능하다. 
하지만 전체 상태에 대한 정보를 조회한다거나 현재 `Local State` 에 존재하지 않고 다른 인스턴스에 존재하는 값이 필요한 경우는 관련 있는 모든 인스턴스에서 
자신의 `Local State` 를 외부에서 접근할 수 있도록 노출하는 별도의 작업이 필요하다. 
그리고 노출된 `Local State` 를 원격으로 접속해 쿼리할 수 있는 `RPC` 구현도 있어야 한다. 
아래는 `Kafka Streams` 를 사용할 때 `Remote State` 사용 절차에 있어서 `Single Application` 와 `Entire Application` 의 `Kafka Streams` 의 기능 지원여부를 정리한 것이다.

구분 | Single Application |Entire Application 
---|--------------------|---
현재 인스턴스에서 현재 로컬 상태 조회| 지원|지원
현재 인스턴스를 다른 인스턴스에 발견 하도록 만들기|지원|지원
모든 실행 중인 인스턴스의 상태 저장소 발견|지원|지원
네트워크를 통한 전체 인스턴스 간 통신|지원| 지원하지 않음(별도 구성 필요)

위 정리 내용에 대해 좀 더 상세히 설명하면 아래와 같다. 

구분 | Single Application                          |Entire Application 
---|---------------------------------------------|---
현재 인스턴스에서 현재 로컬 상태 조회| 자신의 로컬 상태 저장소에는 직접 쿼리할 수 있다.                | 여러 인스턴스의 로컬 상태를 개별적으로 쿼리할 수 있다. 이때 필요한 데이트가 어느 인스턴스에 위치하는 지는 별도로 판별 후 해당 인스턴스에 쿼리해야 한다. 
현재 인스턴스를 다른 인스턴스에 발견 하도록 만들기| 각 인스턴스는 자신의 `호스트:포트` 등 메타데이터를 외부로 제공할 수 있다. |모든 인스턴스의 메타데이터를 관리하고 공유할 수 있다. 
모든 실행 중인 인스턴스의 상태 저장소 검색| 각 인스턴스는 다른 모든 인스턴스와 상태 저장소를 검색할 수 있다.       |네트워크를 통해 전체 인스턴스의 상태 저장소를 검색할 수 있고, 이를 통해 필요한 상태 저장소가 있는 인스턴스를 특정할 수 있다. 
네트워크를 통한 전체 인스턴스 간 통신| 각 인스턴스는 다른 인스턴스와 네트워크를 통해 통신할 수 있다. | 전체 애플리케이션 레벨에서 `RPC` 통신은 `Kafka Streams` 에서 기본 제공하지 않고, 직접 구현해야 한다. 

> 여기서 전체 애플리케이션이라 함은 `Kafka` 의 `Consumer Group` 단위로 봐도 무방하다.


이후 설명에 사용하는 모든 예제의 상새 내용은 [여기](https://github.com/windowforsun/kafka-streams-interactive-queries-exam)
에서 확인 할 수 있다.  

### Query local state stores of an app instance
`Kafka Streams` 에서 현재 인스턴스의 `Local State Store` 라는 것은 전체 `State Store` 의 일부이다. 
현재 로컬 상태의 조회가 필요한 경우 `KafkaStreams.store()` 를 사용해 로컬 상태 저장소를 이름과 저장소 유형에 따라 찾을 수 있다. 

> `Kafka Streams 3.5` 버전 기준으로 `VersionesStateStore` 는 지원되지 않는다. 

조회에 필요한 상태 저장소 이름은 `Processor API` 혹은 `Streams DSL` 을 사용할 때 멍시적으로 설정하거나, 
설정하지 않은 경우 암시적으로 생성되기 때문에 이를 인지하고 사용해야 한다. 
그리고 상태 저장소의 유형의 경우 `QueryableStoreType` 을 통해 결정할 수 있다.  


> `Kafka Streams` 는 스트림 파티션당 하나의 상태 저장소를 구성한다. 
> 즉 해당 애플리케이션 인스턴스가 `N` 개의 파티션을 할당 받았다면 로컬 상태 저장소도 파티션 수에 비례한다는 의미이다. 
> `Interactive Queries` 즉 `KafkaStreams.store()` 를 통해 얻은 상태 저장소 객체의 경우 
> 이름과 저장소 유형에 해당하는 각 파티션 별 상태 저장소가 통합된 상태로 제공하기 때문에 이러한 부분을 크게 고려할 필요는 없다. 


#### Querying Local KeyValueStore
아래는 예제를 위해 구현한 레코드 값의 개별 단어수를 `KeyValueStore` 를 사용해 카운트 하는 `Kafka Streams` 구현이다.  

```java
public void queryLocalKeyValue(KStream<String, String> inputStream) {
    KGroupedStream<String, String> kGroupedStream = inputStream
        .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
        .groupBy((key, value) -> value, Grouped.with(Serdes.String(), Serdes.String()));

    kGroupedStream.count(Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("CountKeyValueStore"));
}
```  

저장소 유형은 `KeyValueStore` 이고, 저장소의 이름은 `CountKeyValueStore` 이다. 
`Kafka Streams` 애플리케이션이 실행되면 해당 상태 저장소를 현재 인스턴스에서 바로 조회해 쿼리할 수 있다.  

```java
@Test
public void queryLocalKeyValue() throws InterruptedException {
    Awaitility.await().atMost(10, TimeUnit.SECONDS)
        .pollDelay(100, TimeUnit.MILLISECONDS)
        .until(() -> kafkaStreams.state() == KafkaStreams.State.RUNNING);

    this.kafkatemplate.send("input-topic", "a", "hello world");
    this.kafkatemplate.send("input-topic", "c", "hi world");
    this.kafkatemplate.send("input-topic", "d", "bye world");
    this.kafkatemplate.send("input-topic", "a", "hello land");
    this.kafkatemplate.send("input-topic", "b", "hi land");
    this.kafkatemplate.send("input-topic", "d", "bye land");
    Thread.sleep(2000);

    ReadOnlyKeyValueStore<String, Long> countKeyValueStore = kafkaStreams.store(
        StoreQueryParameters.fromNameAndType("CountKeyValueStore", QueryableStoreTypes.keyValueStore()));

    assertThat(countKeyValueStore.get("hello"), is(2L));
    assertThat(countKeyValueStore.get("world"), is(3L));
    assertThat(countKeyValueStore.get("hi"), is(2L));
    assertThat(countKeyValueStore.get("bye"), is(2L));
    assertThat(countKeyValueStore.get("land"), is(3L));
}
```  

`ReadOnlyKeyValueStore` 에 대한 상세한 사용법은 [여기](https://kafka.apache.org/35/javadoc/org/apache/kafka/streams/state/ReadOnlyKeyValueStore.html)
에서 확인 할 수 있다. 


#### Querying Local WindowStore
아래는 `WindowStore` 를 사용해 레코드의 값의 개벼 단어 수를 카운트 하는 구현이다. 

```java
public void queryLocalWindow(KStream<String, String> inputStream) {
	KGroupedStream<String, String> kGroupedStream = inputStream
		.flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
		.groupBy((key, value) -> value, Grouped.with(Serdes.String(), Serdes.String()));

	kGroupedStream
		.windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofSeconds(1)))
		.count(Materialized.<String, Long, WindowStore<Bytes, byte[]>>as("CountWindowStore").withRetention(Duration.ofMinutes(1)));
}
```  

`WindowStore` 는 주어진 키에 대해 `Time Window` 별로 여러 결과가 있을 수 있기 때문에
상태 저장소 쿼리에도 앞선 `KeyValueStore` 와는 차이가 있다.  

```java
@Test
public void queryLocalWindow() throws Exception {
    Awaitility.await().atMost(10, TimeUnit.SECONDS)
        .pollDelay(100, TimeUnit.MILLISECONDS)
        .until(() -> kafkaStreams.state() == KafkaStreams.State.RUNNING);

    Instant windowQueryStart = Instant.now().minusSeconds(1);
    Instant windowQueryEnd = Instant.now().plusSeconds(10);
    this.kafkatemplate.send("input-topic", "a", "hello world");
    this.kafkatemplate.send("input-topic", "c", "hi world");
    Thread.sleep(1000);
    this.kafkatemplate.send("input-topic", "d", "bye world");
    this.kafkatemplate.send("input-topic", "a", "hello land");
    this.kafkatemplate.send("input-topic", "b", "hi land");
    this.kafkatemplate.send("input-topic", "d", "bye land");
    Thread.sleep(2000);

    ReadOnlyWindowStore<String, Long> countWindowStore = kafkaStreams.store(
        StoreQueryParameters.fromNameAndType("CountWindowStore", QueryableStoreTypes.windowStore()));

    WindowStoreIterator<Long> helloKeyWindowStore = countWindowStore.fetch("hello", windowQueryStart, windowQueryEnd);
    assertThat(helloKeyWindowStore.next().value, is(1L));
    assertThat(helloKeyWindowStore.next().value, is(1L));

    WindowStoreIterator<Long> worldKeyWindowStore = countWindowStore.fetch("world", windowQueryStart, windowQueryEnd);
    assertThat(worldKeyWindowStore.next().value, is(2L));
    assertThat(worldKeyWindowStore.next().value, is(1L));

    WindowStoreIterator<Long> hiKeyWindowStore = countWindowStore.fetch("hi", windowQueryStart, windowQueryEnd);
    assertThat(hiKeyWindowStore.next().value, is(1L));
    assertThat(hiKeyWindowStore.next().value, is(1L));

    WindowStoreIterator<Long> byeKeyWindowStore = countWindowStore.fetch("bye", windowQueryStart, windowQueryEnd);
    assertThat(byeKeyWindowStore.next().value, is(2L));

    WindowStoreIterator<Long> landKeyWindowStore = countWindowStore.fetch("land", windowQueryStart, windowQueryEnd);
    assertThat(landKeyWindowStore.next().value, is(3L));
}
```  


`ReadOnlyWindowStore` 에 대한 상세한 사용법은 [여기](https://kafka.apache.org/35/javadoc/org/apache/kafka/streams/state/ReadOnlyWindowStore.html)
에서 확인 할 수 있다.  


#### Querying Local CustomStore
`CustomStore` 는 `Kafka Streams` 에서 기본으로 제공하는 상태 저장소와는 별개로 사용자가 커스텀하게 구현이 필요할 때 사용할 수 있다. 
이는 `Processor API` 에서만 활용이 가능하고 `Kafka Streams` 와 연동 및 쿼리를 위해 아래와 같은 인터페이스 구현이 필요하다. 

- `StateStore` 인터페이스 구현 : `Kafka Streams` 가 상태 저장소를 관리하고 상호 작용하는데 필요한 기본 메서드를 정의 해야한다. 
- 사용자 저장소 동작 정의를 위한 인터페이스 정의 : 사용자 정의 저장소의 읽기, 쓰기, 삭제 등 작업을 정의한다. 이곳에서 사용자 정의 동작을 결정할 수 있다.   
- `StoreBuilder` 인터페이스 구현 : `Kafka Streams` 가 필요에 따라 저장소의 새 인스턴스를 생성할 수 있도록 한다. 
- `ReadOnly` 구현체 제공 : 사용자 정의 저장소에 대해 외부 변경을 막아, 무결성을 보호한다. 

아래는 사용자 정의 상태 저장소 구현에 필요한 구현의 일부이다. 
상세한 구현 내용은 [여기](https://github.com/windowforsun/kafka-streams-interactive-queries-exam)
에서 확인 가능하다. 
구현하는 `CusotmStore` 는 `KeyValueStore` 와 동일한 성격을 지닌 상태 저장소로 구현한다.  

```java
public class MyCustomStore<K,V> implements StateStore, MyWriteableCustomStore<K,V> {
  // 실제 저장소의 구현
}

// MyCustomStore의 읽기-쓰기 인터페이스
public interface MyWriteableCustomStore<K,V> extends MyReadableCustomStore<K,V> {
  void write(K Key, V value);
}

// MyCustomStore의 읽기 전용 인터페이스
public interface MyReadableCustomStore<K,V> {
  V read(K key);
}

public class MyCustomStoreBuilder implements StoreBuilder {
  // MyCustomStore에 대한 공급자 구현
}
```  

추가적으로 사용자 정의 저장소에서 `Interactive Queries` 를 제공하기 위해서는 아래와 같은 구현이 필요하다. 

- `QueryableStoreType` 구현 : `Kafka Streams` 가 특정 유형의 상태 저장소를 인식하고 관리할 수 있도록 한다. 
- `Wrapper` 클래스 제공 : 여러 로컬 상태 저장소 인스턴스에 대한 단일 접근 지점을 제공한다. 

```java
public class MyCustomStoreType<K,V> implements QueryableStoreType<MyReadableCustomStore<K,V>> {

  // MyCustomStore 유형의 StateStores만 허용
  @Override
  public boolean accepts(final StateStore stateStore) {
    return stateStore instanceof MyCustomStore;
  }

  @Override
  public MyReadableCustomStore<K,V> create(final StateStoreProvider storeProvider, final String storeName) {
	  return new MyCustomStoreTypeWrapper<K, V>(stateStoreProvider, s, this);
  }

}
```  

`Kafka Streams` 애플리케이션의 각 인스턴스는 여러 스트림 태스크를 실행하고, 
여러 개의 로컬 상태 저장소 인스턴스를 관리할 수 있다. 
`Wrapper` 클래스는 이러한 복잡성을 숨기고 사용자가 쉽게 상태 저장소를 쿼리할 수 있도록, 
즉 모든 개별 로컬 상태 저장소 인스턴스를 알 필요 없이 이름으로만 논리적으로 상태 저장소를 쿼리할 수 있도록 한다.  

`Wrapper` 클래스를 구현할 때는 `StateStoreProvider` 인터페이스를 사용해 실제 상태 저장소 인스턴스에 접근한다. 
`StateStoreProvider#stores(String storeName, QueryableStoreType<T> queryableStoreType)` 를 사용하면 
`List` 형태의 상태 저장소들을 얻을 수 있는데 이러한 결과값을 통해 전체 로컬 상태 인스턴스에 대해 쿼리할 수 있는 구현체를 제공하면 된다.  


```java
public class MyCustomStoreTypeWrapper<K, V> implements MyReadableCustomStore<K, V> {
	private final QueryableStoreType<MyReadableCustomStore<K, V>> customStoreType;
	private final String storeName;
	private final StateStoreProvider provider;

	public MyCustomStoreTypeWrapper(StateStoreProvider provider, String storeName,
		QueryableStoreType<MyReadableCustomStore<K, V>> customStoreType
	) {
		this.customStoreType = customStoreType;
		this.storeName = storeName;
		this.provider = provider;
	}

	@Override
	public V get(K key) {
		final List<MyReadableCustomStore<K, V>> stores = this.provider.stores(this.storeName, this.customStoreType);
		final Optional<V> value = stores.stream()
			.filter(store -> store.get(key) != null)
			.map(store -> store.get(key))
			.findFirst();

		return value.orElse(null);
	}
}
```  

아래는 구현한 `CustomStore` 를 사용해 레코드 값의 개별 단어 수를 카운트하는 예제로 `Processor API` 를 사용했다.  

```java
public void queryLocalCustom(StreamsBuilder builder) {
    Topology topology = builder.build();
    MyCustomStoreBuilder<String, Long> customStoreBuilder = new MyCustomStoreBuilder<>("CustomStore");

    topology.addSource("input", "input-topic")
        .addProcessor("split", SplitProcessor::new, "input")
        .addProcessor("count", CountProcessor::new, "split")
        .addStateStore(customStoreBuilder, "count")
        .connectProcessorAndStateStores("count", "CustomStore");
}

static class SplitProcessor implements Processor<String, String, String, String> {
    private ProcessorContext<String, String> context;

    @Override
    public void process(Record<String, String> record) {
        String value = record.value();
        List<String> list = Arrays.asList(value.toLowerCase().split("\\W+"));

        for (String word : list) {
            this.context.forward(new Record<>(record.key(), word, record.timestamp()));
        }
    }

    @Override
    public void init(ProcessorContext<String, String> context) {
        this.context = context;
    }

    @Override
    public void close() {
    }
}

static class CountProcessor
    implements Processor<String, String, String, Long> {
    private MyCustomStore<String, Long> kvStore;

    private ProcessorContext<String, Long> context;

    @Override
    public void init(ProcessorContext<String, Long> context) {
        this.context = context;
        this.kvStore = context.getStateStore("CustomStore");
    }

    @Override
    public void process(Record<String, String> record) {
        String key = record.key();
        String value = record.value();

        Long count = this.kvStore.read(value);
        if (count == null) {
            count = 0L;
        }
        
        count++;
        this.kvStore.write(value, count);
        this.context.forward(new Record<>(value, count, record.timestamp()));
    }

    @Override
    public void close() {}
}
```  

`CustomStore` 에 대한 `Interactive Queries` 는 아래와 같이 사용할 수 있다.  

```java
@Test
public void queryLocalCustom() throws InterruptedException {
    Awaitility.await().atMost(10, TimeUnit.SECONDS)
        .pollDelay(100, TimeUnit.MILLISECONDS)
        .until(() -> kafkaStreams.state() == KafkaStreams.State.RUNNING);

    this.kafkatemplate.send("input-topic", "a", "hello world");
    this.kafkatemplate.send("input-topic", "c", "hi world");
    this.kafkatemplate.send("input-topic", "d", "bye world");
    this.kafkatemplate.send("input-topic", "a", "hello land");
    this.kafkatemplate.send("input-topic", "b", "hi land");
    this.kafkatemplate.send("input-topic", "d", "bye land");
    Thread.sleep(2000);

    MyReadableCustomStore<String, Long> countCustomStore = kafkaStreams.store(
        StoreQueryParameters.fromNameAndType("CustomStore", new MyCustomStoreType<>()));

    assertThat(countCustomStore.read("hello"), is(2L));
    assertThat(countCustomStore.read("world"), is(3L));
    assertThat(countCustomStore.read("hi"), is(2L));
    assertThat(countCustomStore.read("bye"), is(2L));
    assertThat(countCustomStore.read("land"), is(3L));
}
```  

### Query remote state stores for entire app instance
전체 애플리케이션 인스턴스에 대해 `Remote Store` 를 쿼리하기 위해서는 각 애플리케이션에서 자신의 상태 저장소를 
다른 애플리케이션에게 노출해야 한다. 
전체 애플리케이션 인스턴스에 대해 전체 상태를 쿼리할 수 있도록 하는 주요 스텝은 아래와 같다.  

1. 네트워크를 통해 애플리케이션 인스턴스와 상호작용할 수 있는 `RPC` 레이어를 추가한다. (e.g. `REST`, `Thrift`, 등) 해당 `RPC` 레이어에서는 `Interactive Queries` 에 대한 적절한 쿼리 결과를 응딥해야 한다. 
2. 애플리케이션 인스턴스에서 `RPC` 레이어 엔드포인트를 `application.server` 구성을 통해 노출한다. `RPC` 엔드포인트는 네트워크 내에서 고유하면서 식별 가능하도록 각 인스턴스에서 설정이 필요하다. 이를 통해 `Kafka Streams` 는 다른 인스턴스 애플리케이션에서 필요한 인스턴스를 검색할 수 있다. 
3. `RPC` 레이어에서 원격 애플리케이션 인스턴스와 상태 저장소를 검색하고 사용 가능한 상태 저장소를 쿼리해 애플리케이션의 전체 상태를 쿼리할 수 있도록 한다. 특정 인스턴스에서 쿼리 응답이 부족할 경우 다른 애플리케이션 인스턴스에 쿼리를 연쇄적으로 전달해 충분한 쿼리 결과를 만들어 내도록할 수 있다.  

