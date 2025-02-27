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
    - Kafka Streams
toc: true
use_math: true
---  

## Kafka Streams Stateless Transformations
`Kafka Streams` 에서 `Stateless Transformations` 는 데이터 처리에 이전 상태나 컨텍스트를 참조하지 않고, 
각 이벤트나 메시지를 독립적으로 변환하는 방식으로 상태 저장소(`State Store`)가 필요하지 않다. 
이러한 데이터 변환은 특정 레코드나 처리 결과가 다른 레코드의 처리 결과에 영향을 주지 않고, 
즉작적이고 독립적인 처리가 이뤄지는게 특징이다.  

`Stateless` 라는 용어는 말 그대로 상태를 유지하지 않는다는 의미로, 
데이터의 처리 과정에서 앞서 처리한 레코드나 상태를 기억하지 않고 각 레코드를 독릭접으로 변환한다. 
이는 비교적 빠르고 단순한 데이터 변환에 적합하고, 
데이터의 순서나 이전 레코드의 상태를 고려하지 않는다. 
주요 특징을 정리하면 아래와 같다.  

- 독립적인 레코드 처리 : 각 레코드는 독립적으로 처리되고, 앞서 처리된 레코드나 다른 레코드의 결과와 관계없이 별도의 처리 로직이 적용된다. 
- 상태 정보 없음 : 상태를 기억하지 않기 때문에 상태 유지 및 동기화 관련 복잡성이 제거된다. 이에 따라 처리 로직이 단순해지며, 성능도 형상된다. 
- 빠르고 간결한 처리 : 간단한 필터링, 매핑, 변환 등의 작업을 빠르게 처리할 수 있어 대용량 스트리밍 데이터 처리에 유리하다.  
- 분산 처리 : 상태를 유지하지 않기 때문에 더 쉽게 분산 처리가 가능하여 시스템 확장성에 유리하다.  

`Stateless Transformation` 이 갖는 한계는 아래와 같다.  

- 복잡한 처리 : 상태를 고려해야 하는 복잡한 처리 로직은 구현이 어렵다. 이런 경우에는 `Stateful Transfomration` 을 활용해야 한다. 
- 순서 의존적인 처리 : 각 레코드가 독립적으로 처리되기 때문에 레코드의 순서를 고려해야 하는 처리 로직에는 적합하지 않다. 

### Stateless Transformations
각 `Transformation` 의 사용 예시를 보며 사용했을 때 스트림의 데이터가 어떤식으로 변환이 되는지 알아본다. 
전체 코드 내용은 [여기]()
에서 확인 가능하다.  

#### branch
`KStream.branch()` 는 주어진 여러 개의 조건에 따라 `KStream` 을 여러 하위 스트림으로 분기하는 처리이다. 
각 조건에 맞는 레코드는 해당 분기로 전달되고, 조건에 맞지 않는 레코드는 다른 분기로 전달된다. 
즉 하나의 `KStream` 을 하위 여러 `KStream` 으로 나누는 역할을 하고, 
조건이 여러 개일 때, 하나의 레코드가 첫 번째 조건에 부합하면 그 후 조건은 확인하지 않는다. 
위와 같은 특징으로 조건의 순서가 중요할 수 있다. 
한 레코드가 여러 조건에 부합할 수 있더라도, 가장 먼저 부합되는 조건을 기준으로 처리되기 때문이다. 
그리고 분기가 복잡하고 많을 수록 조건 평가에 따른 성능 비용이 커질 수 있기 때문에 적절하게 조절이 필요하다.  

```
KStream -> KStream[]
```  

소개할 예제에서 `branch` 의 조건 순서는 다음과 같다. 
레코드의 `value` 가 `a` 문자를 포함하는지, 
`b` 문자를 포함하는지 
`a`, `b` 문자를 모두 포함하지 않는 경우로 구성돼있다. 
이를 그림으로 도식화하면 아래와 같다.  

.. 그림 .. 

코드 구현 예시와 이를 검증하는 테스트 코드는 아래와 같다.  

```java
public void branch(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");

    Map<String, KStream<String, String>> branches = inputStream.split(Named.as("branch-"))
        .branch((k, v) -> v.contains("a"), Branched.as("a"))
        .branch((k, v) -> v.contains("b"), Branched.as("b"))
        .defaultBranch(Branched.as("other"));

    branches.get("branch-a").to("output-a-topic");
    branches.get("branch-b").to("output-b-topic");
    branches.get("branch-other").to("output-other-topic");
}

@Test
public void branch() {
	this.statelessTransforms.branch(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputATopic = this.topologyTestDriver.createOutputTopic("output-a-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());
	TestOutputTopic<String, String> outputBTopic = this.topologyTestDriver.createOutputTopic("output-b-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());
	TestOutputTopic<String, String> outputOtherTopic = this.topologyTestDriver.createOutputTopic("output-other-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, String>> outputA = outputATopic.readKeyValuesToList();
	assertThat(outputA, hasSize(3));
	assertThat(outputA.get(0), is(KeyValue.pair("voter1", "a")));
	assertThat(outputA.get(1), is(KeyValue.pair("voter4", "a")));
	assertThat(outputA.get(2), is(KeyValue.pair("voter5", "a")));

	List<KeyValue<String, String>> outputB = outputBTopic.readKeyValuesToList();
	assertThat(outputB, hasSize(2));
	assertThat(outputB.get(0), is(KeyValue.pair("voter2", "b")));
	assertThat(outputB.get(1), is(KeyValue.pair("voter5", "b")));

	List<KeyValue<String, String>> outputOther = outputOtherTopic.readKeyValuesToList();
	assertThat(outputOther, hasSize(3));
	assertThat(outputOther.get(0), is(KeyValue.pair("voter3", "c")));
	assertThat(outputOther.get(1), is(KeyValue.pair("voter6", "c")));
	assertThat(outputOther.get(2), is(KeyValue.pair("voter6", "d")));
}
```  

#### filter
`KStream.filter()`, `KTable.filter()` 는 주어진 조건을 만족하는 레코드만 통과시키는 연산이다. 
조건을 만족하지 않는 레코드는 버려진다. 
이는 각 레코드를 게별적으로 판별해서 조건을 만족하는 레코드만 `downstream` 으로 넘기게 된다. 
이를 위해서 판별하고자 하는 식을 `true/false` 를 반환하는 조건식으로 구현해야 한다.  

```
KStream -> KStream
KTable -> KTable
```  

소개할 예제는 레코드의 `value` 값이 `a` 문자열을 포함하는 경우는 스트림형 결과 토픽으로 넘기게되고, 
`b` 문자열을 포함하는 경우에는 테이블형 결과 토픽으로 넘기게 된다. 
이를 도식화하면 아래와 같다.  

.. 그림 ..

코드 구현 예시와 이를 검증하는 테스트 코드는 아래와 같다.

```java
public void filter(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");
    KTable<String, String> inputTable = inputStream.toTable();

    inputStream.filter((k, v) -> v.contains("a")).to("output-stream-topic");

    inputTable.filter((k, v) -> v.contains("b")).toStream().to("output-table-topic");
}

@Test
public void filter() {
	this.statelessTransforms.filter(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputStreamTopic = this.topologyTestDriver.createOutputTopic("output-stream-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());
	TestOutputTopic<String, String> outputTableTopic = this.topologyTestDriver.createOutputTopic("output-table-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, String>> outputStream = outputStreamTopic.readKeyValuesToList();
	assertThat(outputStream, hasSize(3));
	assertThat(outputStream.get(0), is(KeyValue.pair("voter1", "a")));
	assertThat(outputStream.get(1), is(KeyValue.pair("voter4", "a")));
	assertThat(outputStream.get(2), is(KeyValue.pair("voter5", "a")));

	List<KeyValue<String, String>> outputTable = outputTableTopic.readKeyValuesToList();
	assertThat(outputTable, hasSize(8));
	assertThat(outputTable.get(0), is(KeyValue.pair("voter1", null)));
	assertThat(outputTable.get(1), is(KeyValue.pair("voter2", "b")));
	assertThat(outputTable.get(2), is(KeyValue.pair("voter3", null)));
	assertThat(outputTable.get(3), is(KeyValue.pair("voter4", null)));
	assertThat(outputTable.get(4), is(KeyValue.pair("voter5", null)));
	assertThat(outputTable.get(5), is(KeyValue.pair("voter5", "b")));
	assertThat(outputTable.get(6), is(KeyValue.pair("voter6", null)));
	assertThat(outputTable.get(7), is(KeyValue.pair("voter6", null)));
}
```  

#### filterNot
`KStream.filterNot()`, `KTable.filterNot()` 은 위에서 먼저 알아본 `filter` 의 반대 연산이다. 
주어진 조건을 만족하지 않는 레코드를 통과시키고, 
조건을 만족하는 레코드는 버려진다. 
이는 조건을 만족하지 않는 레코드만 `downstream` 으로 넘기기 때문에, 
판별 조건식에 부정연산을 붙여야 하는 경우 사용하기 좋다.  

```
KStream -> KStream
KTable -> KTable
```  

소개할 예제는 레코드의 `value` 값이 `a` 문자열을 포함하지 않는 경우 스트림형 결과 토픽으로 넘기게되고,
`b` 문자열을 포함하지 않는 경우 테이블형 결과 토픽으로 넘기게 된다.
이를 도식화하면 아래와 같다.

.. 그림 ..

코드 구현 예시와 이를 검증하는 테스트 코드는 아래와 같다.  

```java
public void inverseFilter(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");
    KTable<String, String> inputTable = inputStream.toTable();

    inputStream.filterNot((k, v) -> v.contains("a")).to("output-stream-topic");

    inputTable.filterNot((k, v) -> v.contains("b")).toStream().to("output-table-topic");
}

@Test
public void inverseFilter() {
	this.statelessTransforms.inverseFilter(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputStreamTopic = this.topologyTestDriver.createOutputTopic("output-stream-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());
	TestOutputTopic<String, String> outputTableTopic = this.topologyTestDriver.createOutputTopic("output-table-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 30L);

	List<KeyValue<String, String>> outputStream = outputStreamTopic.readKeyValuesToList();
	assertThat(outputStream, hasSize(5));
	assertThat(outputStream.get(0), is(KeyValue.pair("voter2", "b")));
	assertThat(outputStream.get(1), is(KeyValue.pair("voter3", "c")));
	assertThat(outputStream.get(2), is(KeyValue.pair("voter5", "b")));
	assertThat(outputStream.get(3), is(KeyValue.pair("voter6", "c")));
	assertThat(outputStream.get(4), is(KeyValue.pair("voter6", "d")));

	List<KeyValue<String, String>> outputTable = outputTableTopic.readKeyValuesToList();
	assertThat(outputTable, hasSize(8));
	assertThat(outputTable.get(0), is(KeyValue.pair("voter1", "a")));
	assertThat(outputTable.get(1), is(KeyValue.pair("voter2", null)));
	assertThat(outputTable.get(2), is(KeyValue.pair("voter3", "c")));
	assertThat(outputTable.get(3), is(KeyValue.pair("voter4", "a")));
	assertThat(outputTable.get(4), is(KeyValue.pair("voter5", "a")));
	assertThat(outputTable.get(5), is(KeyValue.pair("voter5", null)));
	assertThat(outputTable.get(6), is(KeyValue.pair("voter6", "c")));
	assertThat(outputTable.get(7), is(KeyValue.pair("voter6", "d")));
}
```  

#### flatMap
`KStream.flatMap()` 은 하나의 입력 레코드를 여러 개의 출력으로 변환하는 연산이다. 
각 레코드는 다중 레코드 리스트로 변환될 수 있으며, 리스트에 담긴 모든 레코드는 최정 결과 스트림에 들어간다. 
하나의 레코드에서 여러 레코드르 생성할 수 있기 때문에 반환 값은 레코드의 리스트나 `Collection` 종류를 사용해야 한다. 
그리고 출력 레코드가 너무 많이지면 데이터 양이 급격히 증가 할 수 있으므로, 성능에 영향을 미칠 수 있다.  

`flatMap` 사용 시 `repartition` 이 발생 할 수 있으므로 사용에 주의가 필요하다. 
`flatMap` 의 경우 임의로 `key` 를 변경하는 경우 혹은 `flatMap` 사용 후 `downstream` 에서 
`groupByKey` 은 `join` 조합해서 사용 시 `repartition` 이 발생 할 수 있다. 
그러므로 `flatMap` 보다는 가능 하다면 `flatMapValues` 를 사용하는 것을 권장하고, 
`key` 변경은 필요한 경우에만 수행하거나 `groupByKey`, `join` 과 조합해서 사용하는 충분한 검토를 미리 해봐야 한다.  

```
KStream -> KStream
```  

소개할 예제는 레코드의 `value` 값이 `a` 인 경우 원본의 키를 갖는 레코드와 `<원본 키>-2` 와 같은 키를 갖는 
새로운 레코드를 만들어 총 2개읠 레코드를 반환한다.  
이를 도식화 하면 아래와 같다.  

.. 그림 .. 

```java
public void flatMap(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");

    inputStream.flatMap((key, value) -> {
        List<KeyValue<String, String>> kvs = new ArrayList<>();

        kvs.add(KeyValue.pair(key, value));

        if ("a".equals(value)) {
            kvs.add(KeyValue.pair(key + "-2", value));
        }

        return kvs;
    }).to("output-result-topic");
}

@Test
public void flatMap() {
	this.statelessTransforms.flatMap(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputResultTopic = this.topologyTestDriver.createOutputTopic("output-result-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, String>> outputResult = outputResultTopic.readKeyValuesToList();

	assertThat(outputResult, hasSize(11));
	assertThat(outputResult.get(0), is(KeyValue.pair("voter1", "a")));
	assertThat(outputResult.get(1), is(KeyValue.pair("voter1-2", "a")));
	assertThat(outputResult.get(2), is(KeyValue.pair("voter2", "b")));
	assertThat(outputResult.get(3), is(KeyValue.pair("voter3", "c")));
	assertThat(outputResult.get(4), is(KeyValue.pair("voter4", "a")));
	assertThat(outputResult.get(5), is(KeyValue.pair("voter4-2", "a")));
	assertThat(outputResult.get(6), is(KeyValue.pair("voter5", "a")));
	assertThat(outputResult.get(7), is(KeyValue.pair("voter5-2", "a")));
	assertThat(outputResult.get(8), is(KeyValue.pair("voter5", "b")));
	assertThat(outputResult.get(9), is(KeyValue.pair("voter6", "c")));
	assertThat(outputResult.get(10), is(KeyValue.pair("voter6", "d")));
}
```

#### flatMapValues
`KStream.flatMapValues()` 는 `flatMap` 과 유사하지만, 
레코드의 키는 유지하고 값만 변환하는 연산이다.
각 레코드는 다중 레코드 리스트로 변환될 수 있으며, 리스트에 담긴 모든 레코드는 최정 결과 스트림에 들어간다.
하나의 레코드에서 여러 레코드르 생성할 수 있기 때문에 반환 값은 레코드의 리스트나 `Collection` 종류를 사용해야 한다.

```
KStream -> KStream
```  

소개할 예제는 레코드의 `value` 값이 `a` 인 경우 동일한 키와 값을 갖는
새로운 레코드를 만들어 총 2개읠 레코드를 반환한다. 
`flatMapValues` 는 값만 추가하면 키는 변경없이 동일한 키로 사용된다. 
이를 도식화 하면 아래와 같다.

.. 그림 .. 

```java
public void flatMapValues(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");

    inputStream.flatMapValues(value -> {
        List<String> kvs = new ArrayList<>();

        kvs.add(value);

        if ("a".equals(value)) {
            kvs.add(value);
        }

        return kvs;
    }).to("output-result-topic");
}

@Test
public void flatMapValues() {
	this.statelessTransforms.flatMapValues(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputResultTopic = this.topologyTestDriver.createOutputTopic("output-result-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, String>> outputResult = outputResultTopic.readKeyValuesToList();

	assertThat(outputResult, hasSize(11));
	assertThat(outputResult.get(0), is(KeyValue.pair("voter1", "a")));
	assertThat(outputResult.get(1), is(KeyValue.pair("voter1", "a")));
	assertThat(outputResult.get(2), is(KeyValue.pair("voter2", "b")));
	assertThat(outputResult.get(3), is(KeyValue.pair("voter3", "c")));
	assertThat(outputResult.get(4), is(KeyValue.pair("voter4", "a")));
	assertThat(outputResult.get(5), is(KeyValue.pair("voter4", "a")));
	assertThat(outputResult.get(6), is(KeyValue.pair("voter5", "a")));
	assertThat(outputResult.get(7), is(KeyValue.pair("voter5", "a")));
	assertThat(outputResult.get(8), is(KeyValue.pair("voter5", "b")));
	assertThat(outputResult.get(9), is(KeyValue.pair("voter6", "c")));
	assertThat(outputResult.get(10), is(KeyValue.pair("voter6", "d")));
}
```  

#### foreach
`KStream.foreach()`, `KTable.foreach()` 는 각 레코드에 대해 주어진 동작을 수행하지만, 
결과 스트림을 반환하지 않는다. 
레코드의 변환 없이 부수적인 작업(로깅, 외부 시스템 연동, ..)을 할 때 사용한다. 
`foreach` 는 스트림을 번환하지 않으므로 이어서 후속처리가 필요하다면 다른 연산을 사용해야 한다. 
그리고 외부 시스템과 연동할 때는 `I/O` 성능에 유의해야 한다. 
외부 시스템과의 `I/O` 로 지연이 발생한다면 이는 스트림 처리 전체의 지연으로 이어질 수 있기 때문이다.  

```
KStream -> void
KTable -> void
```

소개할 예제는 스트림으로 들어오는 레코드를 `foreach : {key}-{value}` 와 같은 포맷으로 
로깅하는 예제이다. 
이를 도식화 하면 아래와 같다.  

.. 그림 ..

```java
public void foreach(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");

    inputStream.foreach((key, value) -> log.info("foreach : {}-{}", key, value));
}

@Test
public void foreach() {
	this.statelessTransforms.foreach(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);
}
/*
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter1-a
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter2-b
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter3-c
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter4-a
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter4-a
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter5-a
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter5-b
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter5-b
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter6-c
INFO 39138 --- [           main] c.w.k.s.s.t.StatelessTransforms          : foreach : voter6-d
 */
```  


#### groupByKey
`KStream.groupByKey()` 는 레코드의 키를 기준으로 데이터를 그룹화하는 연산이다. 
주로 이후의 집계 연산(`count`, `aggregate`)에 사용된다. 
그리고 입력 스트림의 키와 값은 그대로 유지 된다.  

`groupByKey` 는 `KGroupedStream` 을 반환하는데 `Serdes` 는 애플리케이션에 기본으로 설정된 `Serdes` 를 사용한다. 
그러므로 만약 `KGroupedStream` 의 키와 값 타입이 일치하지 않는다면, 
명시적으로 적절한 `Serdes` 를 설정해 줘야 한다.  

`groupByKey` 가 수행하는 연산은 `Grouping` 이다. 
이는 키를 기준으로 레코드를 그룹화하는 연산이므로, 
동일한 키를 가진 레코드들을 시간적 기준으로 하위 그룹으로 나누는 방법인 `Windowing` 과는 
결과적으로 차이가 있음을 인지해야 한다.  

`groupByKey` 는 해당 스트림이 이미 `repartition` 대상으로 마킹이 된 경우에만 `repartition` 이 발생한다. 
즉 항상 `repartition` 을 유발하는 것이 아니라, 스트림이 이미 재분배가 필요하다고 지정된 경우에만 발생한다. 
`groupByKey` 의 경우 레코드의 키를 변경하지 않으므로 `groupBy` 와 비교했을 때 불필요한 `repartition` 을 피할 수 있다는 점에서 
`groupBy` 보다 효율적이다. 

```
KStream -> KGroupedStream
```  

소개할 예제는 `groupByKey` 를 사용해 레코드의 키별로 그룹화 하고, 
그룹화된 키별 레코드의 수를 `count` 한 결과를 토픽으로 보내는 흐름이다. 
이를 도식화하면 아래와 같다. 

.. 그림 ..

```java
public void groupByKey(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic",
        Consumed.with(Serdes.String(), Serdes.String()));

    KGroupedStream<String, String> kGroupedStream = inputStream.groupByKey(
        Grouped.with(Serdes.String(), Serdes.String()));

    kGroupedStream
        .count(Materialized.with(Serdes.String(), Serdes.Long()))
        .toStream()
        .to("output-count-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void groupByKey() {
	this.statelessTransforms.groupByKey(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, Long> outputCountTopic = this.topologyTestDriver.createOutputTopic("output-count-topic", this.stringSerde.deserializer(), Serdes.Long()
		.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, Long>> outputCount = outputCountTopic.readKeyValuesToList();

	assertThat(outputCount, hasSize(8));
	assertThat(outputCount.get(0), is(KeyValue.pair("voter1", 1L)));
	assertThat(outputCount.get(1), is(KeyValue.pair("voter2", 1L)));
	assertThat(outputCount.get(2), is(KeyValue.pair("voter3", 1L)));
	assertThat(outputCount.get(3), is(KeyValue.pair("voter4", 1L)));
	assertThat(outputCount.get(4), is(KeyValue.pair("voter5", 1L)));
	assertThat(outputCount.get(5), is(KeyValue.pair("voter5", 2L)));
	assertThat(outputCount.get(6), is(KeyValue.pair("voter6", 1L)));
	assertThat(outputCount.get(7), is(KeyValue.pair("voter6", 2L)));
}
```

#### groupBy
`KStream.groupBy()`, `KTable.groupBy()` 는 키나 값을 변환하여 새로운 키로 데이터를 그룹화하는 연산이다. 
원래의 키를 유지하지 않고, 주어진 함수에 의해 변환된 키를 기준으로 그룹화한다.  

`groupBy` 연산은 레코드를 새롭게 그룹화하는 연산이기 때문에, 
내부적으로 파티션을 재조정하는 과정을 매번 거치게 된다. 
설령 키가 동일하더라도 이 과정에서 레코드의 키를 기준으로 데이터를 `repartition` 하게 된다. 
즉 `groupBy` 연산을 사용하면 항상 `repartition` 이 발생하기 때문에 사용에 주의가 필요하다. 
그러므로 반드시 키를 변경해 다른 기준으로 그룹화가 필요한 경우에만 `groupBy` 를 사용해야 하고, 
그렇지 않은 경우에는 `groupByKey` 를 사용해 `repartition` 을 회피하는게 더 효율적이다.  

`repartition` 외 다른 내용은 모두 `groupByKey` 에서 설명한 내용과 동일하다.  

```
KStream -> KGroupedStream
KTable -> KGroupedTable
```  

소개할 예제는 기존 레코드의 키가 아닌 값을 기준으로 하는 `KGroupedStream` 과 `KGroupedTable` 을 구성한다. 
그리고 값을 기준으로 그룹화된 각 `count` 값을 결과로 전송하는 흐름이다. 
이를 도식화 하면 아래와 같다.  

.. 그림 ..

```java
public void groupBy(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");
    KTable<String, String> inputTable = inputStream.toTable();

    KGroupedStream<String, String> kGroupedStream = inputStream.groupBy((key, value) -> value);

    kGroupedStream
        .count()
        .toStream()
        .to("output-stream-topic", Produced.with(Serdes.String(), Serdes.Long()));

    KGroupedTable<String, String> kGroupedTable = inputTable.groupBy((key, value) -> KeyValue.pair(value, value));

    kGroupedTable
        .count()
        .toStream()
        .to("output-table-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void groupBy() {
	this.statelessTransforms.groupBy(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, Long> outputStreamTopic = this.topologyTestDriver.createOutputTopic("output-stream-topic", this.stringSerde.deserializer(), Serdes.Long().deserializer());
	TestOutputTopic<String, Long> outputTableTopic = this.topologyTestDriver.createOutputTopic("output-table-topic", this.stringSerde.deserializer(), Serdes.Long().deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, Long>> outputStream = outputStreamTopic.readKeyValuesToList();

	assertThat(outputStream, hasSize(8));
	assertThat(outputStream.get(0), is(KeyValue.pair("a", 1L)));
	assertThat(outputStream.get(1), is(KeyValue.pair("b", 1L)));
	assertThat(outputStream.get(2), is(KeyValue.pair("c", 1L)));
	assertThat(outputStream.get(3), is(KeyValue.pair("a", 2L)));
	assertThat(outputStream.get(4), is(KeyValue.pair("a", 3L)));
	assertThat(outputStream.get(5), is(KeyValue.pair("b", 2L)));
	assertThat(outputStream.get(6), is(KeyValue.pair("c", 2L)));
	assertThat(outputStream.get(7), is(KeyValue.pair("d", 1L)));


	List<KeyValue<String, Long>> outputTable = outputTableTopic.readKeyValuesToList();

	assertThat(outputTable, hasSize(10));
	assertThat(outputTable.get(0), is(KeyValue.pair("a", 1L)));
	assertThat(outputTable.get(1), is(KeyValue.pair("b", 1L)));
	assertThat(outputTable.get(2), is(KeyValue.pair("c", 1L)));
	assertThat(outputTable.get(3), is(KeyValue.pair("a", 2L)));
	assertThat(outputTable.get(4), is(KeyValue.pair("a", 3L)));
	assertThat(outputTable.get(5), is(KeyValue.pair("a", 2L)));
	assertThat(outputTable.get(6), is(KeyValue.pair("b", 2L)));
	assertThat(outputTable.get(7), is(KeyValue.pair("c", 2L)));
	assertThat(outputTable.get(8), is(KeyValue.pair("c", 1L)));
	assertThat(outputTable.get(9), is(KeyValue.pair("d", 1L)));
}
```  


#### cogroup
`KStream.cogroup()` 은 여러 `KGroupedStream` 을 결합하여 집계하는 연산이다. 
여러 스트림을 동시에 그룹화하고 집계할 때 사용되고, 
각 스트림의 레코드를 하나의 집계 결과로 결합할 수 있다. 
즉, 서로 다른 입력 스트림을 하나로 묶어서 동시에 처리할 수 있는 연산으로 
입력 스트림은 각각 미리 그룹화된 상태여야 한다. 
키 타입은 동일해야 하지만 값의 타입은 서로 달라도 상관없다.  

`KStream.cogroup()` 은 하나의 새로운 `cogrouped stream` 을 생성하고, 
`CogroupedKStream.cogroup()` 은 이미 생성된 `cogrouped stream` 에 
다른 그룹화된 스트림을 추가하는 역할을 한다. 
즉 이는 두 번째, 세 번째 스트림을 점진적으로 추가할 때 사용할 수 있다.  

앞서 언급한 것과 같이 `cogroup` 의 입력 스트림은 서로 다른 값 타입을 가질 수 있다. 
따라서 각 스트림의 값에 맞는 개별 `aggregator` 를 구현해야 한다. 
구현한 `aggregator` 는 각 스트림의 값을 집계할 때 사용되며, 
이를 통해 각 스트림마다 서로 다른 방식으로 집계를 수행할 수 있도록 해주는 요소이다.  

`CogroupedKStream` 은 집계 전 `Windowing` 을 적용할 수 있다. 
특정 시간 범위의 `Window` 에 속하는 데이터를 그룹화하고 집계하 수 있는 옵션을 제공한다.  

`cogroup` 사용으로 인한 `repartition` 은 발생하지 않는다. 
이는 `cogroup` 연산 자체는 `repartition` 을 유발하지 않는데, 
그 이유는 `cogroup` 연산은 이미 그룹화된 상태의 입력 스트림을 사용하기 때문이다. 
즉 `cogroup` 연산이 수행되기 전에 그룹화를 통해 이미 `repartition` 이 필요한 경우라면 
모두 수행된 상태이므로 다시 `repartition` 이 발생하지는 않는다.  

```
KGroupedStream -> CogroupedKStream
CogroupedKStream -> CogroupedKStream
```  

소개할 예제는 2개의 입력 스트림을 각각 `value` 를 사용해 `groupBy` 를 수행한다. 
그리고 그룹화 된 2개의 입력 스트림을 `cogroup` 을 사용해 값으로 집계된 레코드의 수를 카운트해 
결과 토픽으로 전송하는 흐름이다. 
이를 도식화하면 아래와 같다.  

.. 그림 ..

```java
public void cogroup(StreamsBuilder streamsBuilder) {
	KStream<String, String> region1InputTopic = streamsBuilder.stream("region1-input-topic");
	KStream<String, String> region2InputTopic = streamsBuilder.stream("region2-input-topic");

	KGroupedStream<String, String> region1VoterGroupedStream = region1InputTopic.groupBy((key, value) -> value,
		Grouped.with("region1-group", Serdes.String(), Serdes.String()));
	KGroupedStream<String, String> region2VoterGroupedStream = region2InputTopic.groupBy((key, value) -> value,
		Grouped.with("region2-group", Serdes.String(), Serdes.String()));

	CogroupedKStream<String, Long> cogroupedKStream = region1VoterGroupedStream.<Long>cogroup((key, value, agg) -> agg + 1L)
		.cogroup(region2VoterGroupedStream, (key, value, agg) -> agg + 1L);

	KTable<String, Long> voteCounts = cogroupedKStream.aggregate(
		() -> 0L,
		Materialized.<String, Long, KeyValueStore<Bytes, byte[]>>as("vote-counts-store")
			.withKeySerde(Serdes.String())
			.withValueSerde(Serdes.Long())
	);

	voteCounts.toStream().to("output-result-topic", Produced.with(Serdes.String(), Serdes.Long()));
}

@Test
public void cogroup() {
	this.statelessTransforms.cogroup(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputRegion1Topic = this.topologyTestDriver.createInputTopic("region1-input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestInputTopic<String, String> inputRegion2Topic = this.topologyTestDriver.createInputTopic("region2-input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, Long> outputResultTopic = this.topologyTestDriver.createOutputTopic("output-result-topic", this.stringSerde.deserializer(), Serdes.Long().deserializer());

	inputRegion1Topic.pipeInput("voter1", "a", 1L);
	inputRegion1Topic.pipeInput("voter2", "b", 2L);
	inputRegion1Topic.pipeInput("voter3", "c", 3L);

	inputRegion2Topic.pipeInput("voter4", "a", 5L);

	inputRegion1Topic.pipeInput("voter5", "a", 10L);
	inputRegion1Topic.pipeInput("voter5", "b", 18L);
	inputRegion2Topic.pipeInput("voter6", "c", 30L);
	inputRegion2Topic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, Long>> outputResult = outputResultTopic.readKeyValuesToList();

	assertThat(outputResult, hasSize(8));
	assertThat(outputResult.get(0), is(KeyValue.pair("a", 1L)));
	assertThat(outputResult.get(1), is(KeyValue.pair("b", 1L)));
	assertThat(outputResult.get(2), is(KeyValue.pair("c", 1L)));
	assertThat(outputResult.get(3), is(KeyValue.pair("a", 2L)));
	assertThat(outputResult.get(4), is(KeyValue.pair("a", 3L)));
	assertThat(outputResult.get(5), is(KeyValue.pair("b", 2L)));
	assertThat(outputResult.get(6), is(KeyValue.pair("c", 2L)));
	assertThat(outputResult.get(7), is(KeyValue.pair("d", 1L)));
}
```  

#### map
`KStream.map()` 은 각 레코드를 새로운 `key-value` 쌍으로 변환하는 연산이다. 
입력 스트림의 각 레코드가 주어진 함수에 따라 새롭게 매핑된다. 

`map` 사용 시 `repartition` 이 발생 할 수 있으므로 사용에 주의가 필요하다.
`map` 의 경우 임의로 `key` 를 변경하는 경우 혹은 `map` 사용 후 `downstream` 에서
`groupByKey` 은 `join` 조합해서 사용 시 `repartition` 이 발생 할 수 있다.
그러므로 `map` 보다는 가능 하다면 `mapValues` 를 사용하는 것을 권장하고,
`key` 변경은 필요한 경우에만 수행하거나 `groupByKey`, `join` 과 조합해서 사용하는 충분한 검토를 미리 해봐야 한다.  

키와 값을 모두 변경할 수 있고 이로 인해 `repartition` 에 대한 주의가 필요한 점은 `flatMap` 과 모두 동일하다. 
차이점은 `flatMap` 은 `Collections` 를 리턴해 스트림에 레코드를 추가할 수 있는 반면
`map` 은 각 레코드를 변환 하는 것만 가능하므로 새로운 레코드를 추가하는 등의 동작은 수행 할수 없다.  

```
KStream -> KStream
```  

소개할 예제는 레코드의 값이 `a` 인 경우 키를 `<기존 키>-2` 와 같이 변경해 결과로 보내게 된다. 
이를 도식화 하면 아래와 같다.  

.. 그림 ..

```java
public void map(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");

    inputStream.map((key, value) -> {
        if ("a".equals(value)) {
            return KeyValue.pair(key + "-2", value);
        }

        return KeyValue.pair(key, value);
    }).to("output-result-topic");
}

@Test
public void map() {
	this.statelessTransforms.map(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputResultTopic = this.topologyTestDriver.createOutputTopic("output-result-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, String>> outputResult = outputResultTopic.readKeyValuesToList();

	assertThat(outputResult, hasSize(8));
	assertThat(outputResult.get(0), is(KeyValue.pair("voter1-2", "a")));
	assertThat(outputResult.get(1), is(KeyValue.pair("voter2", "b")));
	assertThat(outputResult.get(2), is(KeyValue.pair("voter3", "c")));
	assertThat(outputResult.get(3), is(KeyValue.pair("voter4-2", "a")));
	assertThat(outputResult.get(4), is(KeyValue.pair("voter5-2", "a")));
	assertThat(outputResult.get(5), is(KeyValue.pair("voter5", "b")));
	assertThat(outputResult.get(6), is(KeyValue.pair("voter6", "c")));
	assertThat(outputResult.get(7), is(KeyValue.pair("voter6", "d")));
}
```  


#### mapValues
`KStream.mapValues()`, `KTable.mapValues()` 는 각 레코드의 값만 변환하는 연산이다. 
키는 그대로 유지되고, 값만 새로운 형태로 매핑된다. `flatMapValues` 와 비슷하지만 차이점은 `flatMapValues` 은 `Collections` 를 리턴해 스트림에 레코드를 추가할 수 있는 반면
`mapValues` 은 각 레코드를 변환 하는 것만 가능하므로 새로운 레코드를 추가하는 등의 동작은 수행 할수 없다.

```
KStream -> KStream
KTable -> KTable
```

소개할 예제는 레코드의 값이 `a` 인 경우 이를 `aa` 와 같이 값만 변경해 결과 토픽으로 보내는 흐름이다. 
이를 도식화하면 아래와 같다.  

.. 그림 ..

```java
public void mapValues(StreamsBuilder streamsBuilder) {
    KStream<String, String> inputStream = streamsBuilder.stream("input-topic");

    inputStream.mapValues(value -> {
        if ("a".equals(value)) {
            return "aa";
        }

        return value;
    }).to("output-result-topic");
}

@Test
public void mapValues() {
	this.statelessTransforms.mapValues(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputTopic = this.topologyTestDriver.createInputTopic("input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputResultTopic = this.topologyTestDriver.createOutputTopic("output-result-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());

	inputTopic.pipeInput("voter1", "a", 1L);
	inputTopic.pipeInput("voter2", "b", 2L);
	inputTopic.pipeInput("voter3", "c", 3L);

	inputTopic.pipeInput("voter4", "a", 5L);

	inputTopic.pipeInput("voter5", "a", 10L);
	inputTopic.pipeInput("voter5", "b", 18L);
	inputTopic.pipeInput("voter6", "c", 30L);
	inputTopic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, String>> outputResult = outputResultTopic.readKeyValuesToList();

	assertThat(outputResult, hasSize(8));
	assertThat(outputResult.get(0), is(KeyValue.pair("voter1", "aa")));
	assertThat(outputResult.get(1), is(KeyValue.pair("voter2", "b")));
	assertThat(outputResult.get(2), is(KeyValue.pair("voter3", "c")));
	assertThat(outputResult.get(3), is(KeyValue.pair("voter4", "aa")));
	assertThat(outputResult.get(4), is(KeyValue.pair("voter5", "aa")));
	assertThat(outputResult.get(5), is(KeyValue.pair("voter5", "b")));
	assertThat(outputResult.get(6), is(KeyValue.pair("voter6", "c")));
	assertThat(outputResult.get(7), is(KeyValue.pair("voter6", "d")));
}
```  

#### merge
`KStream.merge()` 는 두 개 이상의 `KStream` 을 하나의 `KStream` 으로 병합하는 연산이다. 
여러 입력 스트림의 데이터를 하나의 스트림으로 통합하여 처리할 수 있다.  

주의해야 할 점은 서로 다른 스트림에서 온 레코드 간에 순서 보장은 없다는 점이다. 
여러 스트림에서 입력된 데이터가 결합될 때, 
각 스트림에서의 처리 순서는 유지되지만 최종적으로 병합된 스트림에서는 입력된 순서가 섞일 수 있다. 
`stream1` 의 레코드가 먼저 도착했더라도, `stream2` 의 레코드가 병합된 스트림에서는 먼저 처리 될 수 있다는 점을 기억하고 사용해야 한다.  

```
KStream -> KStream
```  

소개할 예제는 2개의 스트림을 `merge` 를 사용해 하나의 스트림으로 구성해 이를 결과 토픽으로 보내는 흐름이다. 
이를 도식화하면 아래와 같다.  

.. 그림 .. 

```java
public void merge(StreamsBuilder streamsBuilder) {
    KStream<String, String> region1InputTopic = streamsBuilder.stream("region1-input-topic");
    KStream<String, String> region2InputTopic = streamsBuilder.stream("region2-input-topic");

    KStream<String, String> inputStream = region1InputTopic.merge(region2InputTopic);

    inputStream.to("output-result-topic", Produced.with(Serdes.String(), Serdes.String()));
}

@Test
public void merge() {
	this.statelessTransforms.merge(this.streamsBuilder);
	this.startStream();

	TestInputTopic<String, String> inputRegion1Topic = this.topologyTestDriver.createInputTopic("region1-input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestInputTopic<String, String> inputRegion2Topic = this.topologyTestDriver.createInputTopic("region2-input-topic", this.stringSerde.serializer(), this.stringSerde.serializer());
	TestOutputTopic<String, String> outputResultTopic = this.topologyTestDriver.createOutputTopic("output-result-topic", this.stringSerde.deserializer(), this.stringSerde.deserializer());


	inputRegion1Topic.pipeInput("voter1", "a", 1L);
	inputRegion1Topic.pipeInput("voter2", "b", 2L);
	inputRegion1Topic.pipeInput("voter3", "c", 3L);

	inputRegion2Topic.pipeInput("voter4", "a", 5L);

	inputRegion1Topic.pipeInput("voter5", "a", 10L);
	inputRegion1Topic.pipeInput("voter5", "b", 18L);
	inputRegion2Topic.pipeInput("voter6", "c", 30L);
	inputRegion2Topic.pipeInput("voter6", "d", 40L);

	List<KeyValue<String, String>> outputResult = outputResultTopic.readKeyValuesToList();

	assertThat(outputResult, hasSize(8));
	assertThat(outputResult.get(0), is(KeyValue.pair("voter1", "a")));
	assertThat(outputResult.get(1), is(KeyValue.pair("voter2", "b")));
	assertThat(outputResult.get(2), is(KeyValue.pair("voter3", "c")));
	assertThat(outputResult.get(3), is(KeyValue.pair("voter4", "a")));
	assertThat(outputResult.get(4), is(KeyValue.pair("voter5", "a")));
	assertThat(outputResult.get(5), is(KeyValue.pair("voter5", "b")));
	assertThat(outputResult.get(6), is(KeyValue.pair("voter6", "c")));
	assertThat(outputResult.get(7), is(KeyValue.pair("voter6", "d")));
}
```  
