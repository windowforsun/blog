--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Reset"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams Reset Tool 을 사용해 스트림 애플리케이션을 리셋하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Kafka Streams Reset
    - Kafka Streams Reset Tool
    - Reset Tool
toc: true
use_math: true
---  

## Kafka Streams Reset
`Kafka Streams Reset` 이란 `Kafka` 의 `Streams API` 애플리케이션이 입력 데이터를 처음부터 다시 처리하도록 설정하는 것을 의미한다. 
실제 운영/개발 환경에서 스트림 처리를 처음 부터 다시 처리하는 것은 흔한 상황으로, 
개발 테스트 중 혹은 프로덕션의 버그 발생, 데모 구현 등 목적은 다양할 수 있다.  

`Kafka Streams` 는 이런 작업을 수동으로 개발자가 직접 처리할 수도 있지만, 
간편한 리셋 도구를 제공한다. 
그러므로 개발자는 `Apache Kafka 0.10.0.1` 버전 이후라면 리셋도구를 사용해 보다 손쉽고 오류 발생없이 스트림을 처음부터 다시 처리할 수 있다.  

`Kafka Streams Reset Tool` 은 `Kafka Streams Application` 을 구성하는 각 유형의 토픽에 대해서 아래와 같은 동작을 수행한다. 

- `Input Topics` : 오프셋을 툴에서 지정한 위치로 리셋한다. 기본적으론 토픽의 시작위치로 리셋한다. 
- `Intermediate Topics` : 토픽 끝으로 건너뛴다. 애플리케이션의 커밋된 소비자 오프셋을 모든 파티션의 로그 크기로 설정한다. (`applicaiton.id` 이름으로 구성된 토픽)
- `Internal Topics` : 내부 토픽은 삭제한다.  

위와 다르게 수행하지 않는 동작은 아래와 같다.  

- `Output Topics` : 애플리케이션이 출력 토픽으로 결과를 쓰는 경우 해당 토픽은 리셋하지 않는다. `upstream` 애플리케이션이 리셋되어 `downstream` 으로 중복데이터가 전달되는 것에 대한 처리는 사용자의 책임이다. 
- `Local State Store` : 애플리케이션 인스턴스의 내부 상태 저장소는 리셋하지 않는다. 
- `Schema Registry for Internal Topics` : 내부 토픽이 사용하는 스키마 레지스트리의 스키마는 삭제하지 않는다. 이는 수동으로 삭제해야 한다. `Reset Tool` 에서 `dry-run` 옵션으로 초기화기 필요한 `Internal Topics` 를 확인 할 수 있다.  


`Kafka Streams` 의 `Reset` 은 아래와 같은 경우에 필요할 수 있다.  

- 개발 테스트 목적 : 애플리케이션 로직을 변경하고 처음부터 데이터를 다시 처리해야 할 때 사용한다. 이는 새로운 기능을 테스트하거나 버그를 수정한 후 결과를 확인하는데 유용하다. 
- 토폴로지 변경 : 스트림 처리 토폴로지를 크게 변경한 경우, 특히 상태 저장 연산이나 조인 연산이 변경되었을 때 리셋이 필요할 수 있다. 
- 데이터 재처리 : 입력 데이터의 처리 방식을 번경하거나, 과거 데이터를 새로운 로직으로 다시 처리해야 할 때 사용한다. 
- 상태 초기화 : 애플리케이션의 상태를 완전히 초기화하고 처음부터 다시 시작해야 할 때 사용한다. 
- 버그 수정 : 버그 수정 후, 잘못 처리된 데이터를 바로잡기 위해 전체 데이터를 재처리해야 할 때 사용한다. 
- 스키마 변경 : 입력/출력 데이터의 스키마가 크게 변경되어 기존 처리 결과와 호환되지 않을 때 사용한다. 
- 설정 변경 : 주요 설정(상태 저장소 구성)을 적용한 후 애플리케이션을 리셋해야 할 때 사용한다. 

### Before Reset
`Reset Tool` 을 사용하기 전 리셋 대상인 `application.id` 에 해당하는 모든 인스턴스는 중지돼야 한다. 
실행된 상태에서는 `Reset Tool` 이 에러를 출력한다. 
이는 `kafka-consumer-groups.sh` 을 사용해 리셋 대상인 `application.id` 이 조회되는 지로 확인 할 수 있다.  

`Intermdiate Topics` 의 경우 이를 구독하는 `downstream` 이 있는 경우 혹은 개발환경과 같이 테스트인 경우를 제외하고는 
삭제 및 재생성을 해주는 것이 좋다.  


### Topology Changes
`Kafka Streams Reset` 관점에서 `Compatible` 과 `Incompatible` 의 의미는 아래와 같다. 
여기서 호환 된다는 것은 스트림 데이터의 관점에서 변경된 토폴로지로 리셋 후 재시작을 수행 했을 때, 
데이터의 호환성과 일관성 관점에서 문제가 없다는 것을 의미한다. 


#### Compatible Topology Changes
`Kafka Streams Reset Tool` 의 호환성은 주로 토폴로지 변경 후 데이터의 처리의 일관성과 정확성을 유지할 수 있는지에 관한 것이다. 

1. 필터 조건 변경
2. 새로운 필터 추가(레코드별 작업)
3. 데이터 타입이 호환되는 새로운 `map()` 연산 추가 (`repartition` 이 발생하지 않는 상황)
4. 값 타입을 변경하지 않는 `mapValues()` 연산 추가

#### Incompatible Topology Changes
`Kafka Streams Reset Tool` 과 호환되지 않는 토폴로지 변경은 `Streams Application` 의 정상 실행 여부와 `Reset Tool` 의 정상 실행을 의미하지 않는다. 
이는 토폴로지 변경 후 상태 저장소 구조 변경, 또는 데이터 타입 변경등으로 기존 데이터와 새로운 애플리케이션 로직 간의 불일치가 발생하는 상황, 
상태 저장소의 일관성 문제와 같은 데이터 관점에서 `Reset` 및 애플리케이션 재시작 후 데이터의 일관성과 호환성 측면에서 문제가 발생할 수 있음을 의미한다.  

1. `DAG` 토폴로지 구조 변경
2. 상태 저장 작업(집계, 조인)의 입출력 데이터 타입 변경
3. 파티션 수 변경(예외 있음)
4. 키 또는 값 타임 변경
5. 상태 저장소 구성 변경(이름, 보존 정책, 변경 로그 토픽 등)

### Reset Tools
`Kafka Streams Reset Tools` 는 `Kafka Broker` 서버 인스턴스에서 `bin/kafka-streams-application-reset.sh` 를 통해 실행 할 수 있다. 
`Apache Kafka 3.0` 기준으로 현재 사용할 수 있는 옵션과 그 설명은 아래와 같다.  

| 옵션 | 설명                                           |
|---|----------------------------------------------|
| --application-id <String: ID> | (필수) Kafka Streams 애플리케이션 ID(application.id) |
| --bootstrap-server <String: 서버> | (필수) 연결할 서버. 형식: HOST1,HOST2,...             |
| --by-duration <String> | 현재 타임스탬프로부터 기간만큼 오프셋을 리셋. 형식: PnDTnHnMnS     |
| --config-file <String: 파일명> | 관리자 클라이언트 및 내장된 소비자에게 전달할 구성 포함 파일           |
| --dry-run | 리셋 명령을 실행하지 않고 수행할 작업을 표시                    |
| --force | 소비자 그룹 멤버를 강제로 제거                            |
| --input-topics <String: 목록> | 사용자 입력 토픽 목록                                 |
| --intermediate-topics <String: 목록> | 사용자 중간 토픽 목록(through() 메서드로 사용된 토픽)          |
| --internal-topics <String: 목록> | 삭제할 내부 토픽 목록(Apache Kafka 3.0 이상)            |
| --shift-by <Long: 오프셋 수> | 현재 오프셋을 n만큼 이동                               |
| --to-earliest | 가장 초기의 오프셋으로 리셋                              |
| --to-latest | 가장 최근의 오프셋으로 리셋                              |
| --to-offset <Long> | 지정된 오프셋으로 리셋                                 |

`Reset Tool` 을 사용해서 스트림 애플리케이션을 다시 시작할 때는 입력 토픽의 오프셋 리셋 시나리오가 중요하다. 
이는 아래 옵션 중 하나만 선택할 수 있고, 별도로 정의하지 않은 경우 기본적으로 `to-earliest` 로 실행된다.  

- `by-duration`
- `from-file`
- `shift-by`
- `to-datetime`
- `to-earliest`
- `to-latest`
- `to-offset`

그외 옵션들은 필요에 따라 조합해 사용할 수 있다. 
만약 이전 데이터를 다시 처리하지 않고 스트림 애플리케이션만 빈 내부 상태에서 다시 시작하려면 `--input-topics`, 
`--intermediate-topics` 옵션을 제거한 체로 실행할 수 있다. 
위 상황 처럼 토픽관련 옵션에 대한 각 상황별 동작을 정리하면 아래와 같다.  

토픽 옵션|제외 했을 때|사용 했을 때
---|---|---
--input-topics|- 입력 토픽의 오프셋이 리셋되지 않는다.<br>- 애플리케이션은 마지막으로 커밋된 오프셋부터 계속해서 데이터를 처리한다.<br>- 이전 데이터는 다시 처리되지 않는다.|- 사용자가 지정한 옵션으로 오프셋을 리셋한다.
--intermediate-topics|- 중간 토픽의 오프셋이 조종되지 않는다.<br>- 애플리케이션은 중간 토픽의 마지막 커밋된 오프셋부터 계속해서 데이터를 처리한다.<br>- 중간 토픽의 데이터가 중복 처리될 수 있다.|- 중간 토픽의 오프셋을 토픽의 끝으로 이동시킨다. 
--internal-topics|- 모든 내부 토픽(application.id로 시작)이 자동으로 삭제 대상이 된다.|- 삭제할 내부 토픽을 명시적으로 지정 가능 하다. 
all|- 입력 토픽과 중간 토픽의 오프셋이 변경되지 않는다.<br>- 모든 내부 토픽이 삭제된다.<br>- 애플리케이션의 내부 상태는 초기화되지만, 입력 데이터의 처리 위치는 유지된다.|- 입력 토픽 오프셋이 리셋된다<br>- 중간 토픽의 오프셋이 끝으로 이동해, 기준 중간 데이터는 건너 뛴다.<br>- 지정된 내부 토픽들이 삭제되어, 해당하는 내부 상태가 초기화된다.<br>- 입력 데이터 전체를 다시 처리하고 새로운 내부 상태를 구축한다. 중간 데이터는 무시하고 새로운 처리 결과로 시작한다. 


### How to use
> 전체 코드 내용은 [여기](https://github.com/windowforsun/kafka-streams-reset-exam)
> 에서 확인할 수 있다.
`Kafka Streams Reset Tool` 을 사용해 `Streams Application` 을 리셋하는 방법에 대해 알아본다. 
간략하게 수행이 필요한 스텝을 나열하면 아래와 같다.  

1. 리셋 대상인 `Streams Application` 인스턴스 모두 종료(`application.id` 기준)
2. `Reset Tool` 을 사용해서 리셋 수행 
3. 리셋 대상인 `Streams Application` 재시작(`KafkaStreams.clear()` 호출 필요)

예제로 사용할 `Streams Application` 의 변경 전 스트림 처리는 아래와 같고 `application.id` 는 `dem-app` 을 사용한다.  

```java
public KStream<String, String> origin(StreamsBuilder builder) {
    KStream<String, String> inputStream = builder.<String, String>stream("input-topic").peek((k, v) -> log.info("input {} {}", k, v));

    return inputStream.mapValues((k, v) -> String.valueOf(v.length())).peek((k, v) -> log.info("length {} {}", k, v));
}

public KStream<String, String> filterByLengthGT5(KStream<String, String> valueLengthStream) {
	return valueLengthStream.filter((k, v) -> Integer.parseInt(v) > 5).peek((k, v) -> log.info("filterByLengthGT5 {} {}", k, v));
}


public KStream<String, String> countKeyKeyValueStore(KStream<String, String> valueLengthStream) {
	KGroupedStream<String, String> keyGroupedStream = valueLengthStream.peek((k, v) -> log.info("countKeyKeyValueStore {} {}", k, v)).groupByKey();
	KTable<String, Long> keyValueLengthSumTable = keyGroupedStream
		.count(Materialized.<String, Long>as(Stores.persistentKeyValueStore("my-store"))
			.withKeySerde(Serdes.String())
			.withValueSerde(Serdes.Long()));

	return keyValueLengthSumTable.toStream().map((k, v) -> KeyValue.pair(k, String.valueOf(v)));
}

resultStream = this.countKeyKeyValueStore(this.filterByLengthGT5(this.origin(builder)));

resultStream
    .peek((k, v) -> log.info("output {} {}", k, v))
    .to("output-topic", Produced.with(Serdes.String(), Serdes.String()));
```  

`inpu-topic` 으로 들어오는 레코드 값 중 문자열 길이가 5이상인 레코드만 필터링해 중간 토픽인 `value-length-gt5` 토픽에 저장한다. 
그리고 `KTable.count()` 를 사용해 키별 레코드의 수를 카운트해 결과 토픽인 `output-topic` 으로 전송한다.  
애플리케이션을 실행 후 아래와 같은 데이터를 `input-topic` 으로 넣으면 스트림 처리 결과인 `output-topic` 으로 결과가 전송된다.  

```bash
$  docker exec -it myKafka \
> kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --property print.key=true \
> --property key.separator="-" \
> --topic input-topic \
> --from-beginning 
a-desktop
b-mouse
c-keyboard
d-graphics card
a-hello world
b-kafka
c-desktop
d-mouse
a-keyboard

$  docker exec -it myKafka \
> kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --property print.key=true \
> --property key.separator="-" \
> --topic value-length-gt5 \
> --from-beginning 
a-7
c-8
d-13
a-11
c-7
a-8

$  docker exec -it myKafka \
> kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --property print.key=true \
> --property key.separator="-" \
> --topic output-topic \
> --from-beginning 
a-1
c-1
d-1
a-2
c-2
a-3
```  
