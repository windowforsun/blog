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
