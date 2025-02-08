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

## Kafka Streams Versioned State Store
`Kafka Streams` 는 기본적으로 오프셋 순서대로 데이터를 처리한다. 
만약 일련의 이유로 데이터의 순서가 뒤바뀐 경우, 오프셋 기준 순서는 타임스탬프의 기준과 순서가 달라지게 된다. 
그러므로 기존에는 이런 타임스탬프 순으로 데이터 스트림을 처리가 필요한 비지니스에서는 주의가 필요했다. 
`Apache Kafka 3.5` 부터는 `Kafka Streams` 에 버전 관리 상태 저장소(`Versioned State Store`)가 도입되어, 
타임스탬프 기준 순서가 중요할 때 크게 활용 할 수 있다.  

`Versioned State Store` 는 아래와 같은 주요 특징이 있다. 

- 타임스탬프 기반 조회 : 버전된 상태 저장소는 특정 시점에 데이터가 어떤 상태였는지를 조회할 수 있다. 이는 과거 데이터를 분석하거나 특정 시점의 상태로 롤백하는 데 유용할 수 있다. 
- 다중 버전 관리 : 동일한 키에 대해 시간에 따라 변하는 여러 버전을 저장할 수 있다. 
- 기록 보존 기간(`Retention Period`) : 버전된 상태 저장소는 각 데이터 버전을 특정 기간 동인 유지한다. 이 기간이 지나면 해당 버전은 자동으로 삭제된다. 이런 기록 보존 기간은 저장소를 생성할 때 설정 가능하다. 
- 성능 : 기존 상태 저장소와 달리 여러 버전을 저장하고 관리하기 때문에, 성능은 비버전 저장소보다 다소 낮을 수 있다. 하지만 최신 데이터는 별도의 저장소에 관리되므로 조회 시 성능저하가 크지 않다.  


### Un-Versioned Stream-Table Join
먼저 `Un-Versioned State Store` 를 사용 했을 때 발생 할 수 있는 문제점에 대해 알아본다. 
이를 설명하기 위해 식당 주문 시스템을 예로 든다. 
시당 메뉴 가격의 변동은 `price-topice` 을 통해 가격 테이블로 만들어진다. 
그리고 고객의 주문읜 `order-topic` 을 통해 주문 스트름이 생성되고, 
고객 주문이 들어오면 주문 스트림과 가격 테이블을 조인해 최종 가격이 계산되는 방식이다. 
예시 구현을 위해서 `price-topic` 은 `key:메뉴,value:가격` 과 같은 구성이고, 
`order-topic` 은 `key:메뉴,value:고객명-수량` 과 같은 구성이다. 
이러한 메시지 구조에서 키를 기준으로 조인을 수행해 해당 고객의 최종 가격을 알아낸다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-1.drawio.png)


위 처럼 타임스탬프를 신경쓰지 않는 경우에는 식당 주문 시스템의 스트림 처리는 아무 문제 없다. 
이제 타임스템프 정보를 추가해서 순서가 꼬이는 상황을 가정하면 아래와 같다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-2.drawio.png)

`t=0` 시점에 `pizza` 의 가격은 `8` 이기 때문에 `t=1` 에 들어온 `Anna` 의 `pizza` 2개 주문은 총 `16` 이 된다. 
그리고 `t=4` 에 `pizza` 의 가격이 `10` 으로 인상되고, `Ben` 이 `t=5` 에 주문한 1개 주문은 정상적으로 `10` 으로 계산 된다. 
하지만 `Casandra` 의 `t=3` 의 주문이 문제로 `pizza` 의 가격이 `10` 으로 인상이 처리된 다음에 메시지가 전달된다. 
이 경우 `Un-Versioned State Store` 은 `pizza` 의 가격이 이미 `10` 으로 업데이트가 된 상태이므로, 
가격 인상은 `t=4` 에 이뤄지고 주문은 가격 인상 이전인 `t=3` 에 이뤄졌더라도 이전 가격정보가 없기 때문에 `24` 가 아닌 `30` 으로 계산되는 문제가 발생한다.  

앞선 예시는 단순한 식당의 주문이지만 시간에 따라 가격이 계속해서 변하는 주식과 같은 비지니스에서는 위와 같은 문제는 매우 심각한 상황일 것이다.  

### Versioned State Store
`Versioned State Store` 는 이런 문제를 바로 해결할 수 있다. 
각 키에 대한 여러 버전의 레코드를 저장하고 저장된 각 레코드 버전은 관련 값과 타임스템프 정보가 있다. 
그리고 각 키의 가장 최신 값을 조회하는 일반적인 `get(key)` 외에도, 
`Versioned KeyValue Store` 는 `get(key, asOfTimestamp)` 라는 타임스탬프기반 조회 메서드를 지원하여, 
해당 타임스탬프에 활성화된 레코드 버전을 반환한다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-3.drawio.png)

`Versioned State Store` 의 각 레코드는 버전과 관련된 `validFrom` 타임스탬프와 `validTo` 라는 타임스탬프 2개가 존재한다. 
`validFrom` 은 해당 레코드의 타임스탬프이고, `validTo` 타임스탬프는 해당 레코드의 다음 버전 레코드의 타임스탬프이다. 
만약 현재 레코드가 가장 최신 레코드라면 `undefined/infinity` 와 같이 설정된다. 
이러한 버전 관련 타임스탬프정보를 바탕으로 레코드 버전의 유효 기간을 정의하고, 
특정 타임스탬프에 활성화된 레코드를 명시적으로 파악해 결과를 내어줄 수 있다. 
이러한 연산은 앞서 언급한 `get(key, asOfTimestamp)` 에서 수행된다.  

인프라의 자원은 유한하기 때문에 무한한 스트림의 모든 버전을 상태 저장소에 저장할 수는 없다. 
그래서 `Versioned State Store` 를 생성할 때는 보존 기간인 `History Retention` 를 필수적으로 설정해야 한다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-4.drawio.png)


위 그림에서 `History Retention` 는 `30` 이고 현재 스트림의 시간은 `t=63` 인 상태이다. 
현 시점을 기준으로는 `t=60`, `t=50` 그리고 최소 `t=33` 의 시간까지 타임스탬프 조회가 유효하다. 
이는 `현재 스트림 시간(63) - 히스토리 보존 기간(30) = 33` 보다 같거나 크기 때문이다. 
그리고 `t=33` 으로 조회하면 히스토로 보존 기간 밖에 있는 `t=17` 에 업데이트된 레코드를 반환한다는 점을 기억해야 한다. 
이는 `t=33` 시점에 유효한 레코드가 `t=17` 이기 때문에 `Versioned State Store` 는 필요한 버전을 충분히 유지한다.  

하지만 `t=30` 으로 조회하는 경우 `null` 을 반환한다. 
이는 `t=30` 은 히스토리 보존 기간을 벗어났기 때문이다. 
즉 이는 `Versioned State Store` 는 히스토리 보존 기간 범위에 있는 시간 조회에 대해서만 레코드를 반환하고, 
그렇지 않은 경우는 앞선 설명과 같이 `null` 을 반환하게 된다. 
그리고 `Versioned State Store` 의 업데이트 또한 히스토리 보존 기간으로 적용된다. 
즉 히스토리 보존 기간보다 오래된 타임스템프의 쓰기 요청은 거부된다.  


#### get(key)
`Versioned State Store` 기반 테이블은 기존 `Un-Versioned` 와 비교했을 때 조회 기준이 최신 오프셋에서 
최산 타임스탬프로 변경됐기 때문에, 주어진 키에 대해 최신 값을 가져오는 `get(key)` 메서드는 최신 오프셋 레코드가 아닌 최신 타임스탬프 레코드를 반환한다.  


### Versioned Stream-Table Join
앞선 주문 시스템의 문제를 `Versioned State Store` 를 사용하면 문제로 늦기 도착한 주문에 대해서도 해당 주문이 들어온 시간에 가용할 수 있는 가격으로 금액을 계산할 수 있다. 


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-5.drawio.png)

`Versioned State Store` 로 구성된 테이블과 스트림이 조인될 때, 
`Kafka Streams` 는 스트림 레코드의 타임스탬프를 사용해 테이블 저장소에 타임스탬프를 기반으로 조회를 수행한다. 
늦게 도착한 `Casandra` 의 주문이 들어왔을 때 해당 스트림 레코드와 조인할 가격 테이블의 레코드는 
`get(key=pizza, asOfTimestamp=3)` 을 통해 찾고, 
해당 조회의 결과는 `8` 값을 가지기 때문에 `Casandra` 의 최종 지불 금액은 기존 문제가 있었던 `30` 이 아닌 정상적인 `24` 로 계산된다.  

### Latest Timestamp
`Versioned State Store` 는 타임스탬프 기반 조회외에도, 
애플리케이션이 최신 오프셋 기준이 아닌 최신 타임스탬프 기준으로 동작할 수 있도록 한다. 
이는 메시지 전달에서 순서가 변경되더라도, `Table Aggregation` 혹은 `Table-Table Join` 연산에서 타임스탬프 처리를 개선할 수 있다.  

#### Table Aggregation
`Versioned State Store` 기반 테이블에서 `Aggregation` 동작이 `최신 오프셋`아 아닌, 
`최신 타임스탬프` 레코드를 기준으로 반영되는 것을 보이기 위해 식당에서 선호 메뉴 투표상황을 가정해본다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-6.drawio.png)

`Aggregation` 을 수행하면 각 키에 대한 집계 결과를 추적하는 상태 저장소를 유지한다. 
새로운 고객인 `sandwich` 에 투표하면 `Aggregation` 은 상태 저장소에서 `sandwich` 의 투표 수를 조회하고 증가시킨 후, 
이를 다시 상태 저장소에 쓰게 된다. 
그리고 `Aggregation` 을 수행하는 프로세서의 업스트림 테이블 또한 상태 저장소로 구현이 필요하다. 
만약 이미 `pizza` 에 투표한 고객이 `sandwich` 로 투표를 변경한다면 `Aggregation` 은 `pizza` 의 투표 수는 감소시키고, 
`sandwich` 의 투표 수는 증가시켜야 하기 때문이다. 
이는 해당 고객의 이전 투표가 `pizza` 라는 사실을 결정하기 위해 고객의 투표 정보도 테이블로 물리화하고, 
투표의 결과도 테이블로 물리화가 필요한 것이다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-7.drawio.png)

위 그림은 고객 투표 정보 일부가 늦게 도착해 순서가 뒤바뀐 상황을 가정한 것이다. 
`Anna` 는 `t=3` 에 `pizza` 에 투표했었다. 
그리고 `t=4` 에 다시 자신의 투표를 `sandwich` 로 변경했다. 
하지만 `sandwich` 로 변경하기 전인 `t=3` 에 `icecream` 으로 투표를 변경했지만, 
해당 메시지는 시스템의 이슈로 늦게 도착했다. 
위 그림처럼 `Un-Versioned State Store` 를 사용한 경우에는 `Kafka Streams` 의 `Aggregation` 동작을 
이를 다른 메시지와 동일하게 처리한다. 
늦게 도착한 `icecream` 메시지 시점에 `icecream` 투표는 증가하고 `sandwich` 의 투표는 감소하게 되는 것이다. 
이는 `icecream` 으로 변경한 메시지가 늦게 도착한 만큼 오프셋이 `sandwich` 보다 크기 때문에 여타 메시지와 동일하게 처리가 된다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-8.drawio.png)

하지만 `Versioned State Store` 을 기반으로 구성했다면 `Aggregation` 을 수행할 때 최신 오프셋이 아닌 최신 타임스탬프를 기준으로 연산이 된다. 
그러므로 `Anna` 의 늦게 도착한 `icecream` 투표는 투표에 반영되지 않고 투표 결과는 최신 타임스템프를 기준으로 구성된다.  


#### Table-Table Join
`Table-Table Join` 에서도 앞서 살펴본 `Table Aggregation` 과 동일한 효과를 `Versioned State Store` 를 사용함으로써 얻을 수 있다. 
`Versioned State Store` 를 사용해서 테이블 조인을 수행할 때 조인 결과는 키별로 각 소스 테이블의 최신 타임스탬프 레코드를 기반으로 반영하므로, 
순서가 뒤바뀐 메시지가 있는 경우 그 차이를 확인할 수 있다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-9.drawio.png)

투표를 진행할 때 고객의 위치와 조인해 지역별 선호도도 함께 고려한다고 가정하보자. 
`Table-Table Join` 의 경우 조인에 포함되는 어떤 테이블이 업데이트 되든 조인의 트리거 된다. 
이때 기존 키에 대해 순서가 뒤바뀐 메시지 전달을 통한 테이블 업데이트는 조인 동작을 트리거하지 않는다. 
이는 최신 조인 결과가 순서가 바뀐 메시지가 있더라도 각 테이블의 최신 타임스템프를 반영하도록 보장하기 때문이다.  


### Case of Versioned Table
`Versioned State Store` 를 기반으로 테이블을 구성 할때 중간 처리 과정으로 인해 버전 테이블이 아닌 상태에서 스트림 처리가 이뤄질 수 있다. 
그러므로 `Versioned Table` 을 사용 할 때는 어떠한 경우가 `Versioned Table` 이고 아닌지 구분이 필요하므로 이를 판별 할 수 있는 몇가지 경우에 대해 알아보고자 한다.  

#### Stateless Processor
먼저 가장 기본적인 경우는 소스 테이블이 `Versioned State Store` 로 물리화되면 이는 `Versioned Table` 이고, 
소스 테이블이 `Un-Versioned State Store` 로 물리회되면 `Un-Versioned Table` 이다. 
그리고 테이블로 물리화되지 않는 경우도 `Un-Versioned Table` 임을 기억해야 한다. 
이러한 기본적인 경우를 정리하면 아래와 같다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-10.drawio.png)


소스 테이블이 `Versioned State Store` 로 물리화된 다음 해당 테이블에 대한 변환 연산이 수행된 경우에는 좀 더 세분화가 필요하다. 
만약 `mapValues`, `transformValues`, `filter` 와 같이 `Stateless` 한 연산이라면, 
이는 `Versioned Table` 상태를 유지한다. 
즉 `Versioned Table` 을 `Stateless` 연산을 수행한 결과는 여전히 `Versioned Table` 이라는 것이다.  

하지만 변환 결과를 `Un-Versioned State Store` 로 저장하면 `DownStream` 관점에서 이는 `Un-Versioned Table` 이고, 
테이블로 물리화를 수행할 때 명시적으로 저장소 타입을 지정하지 않고 저장소 이름으로만 물리화해도 동일하다. (`Materialized.as("store-name")`) 
반대로 변환 결과를 `Versioned State Store` 로 명시적으로 물리화한다면 이는 `DownStream` 관점에서 `Versioned Table` 이다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-11.drawio.png)


#### Stateful Processor
중간 변환 과정에서 `Aggregation` 혹은 `Join` 과 같은 `Stateful` 한 연산이 있는 경우, 
소스 테이블이 변환 전 `Versioned State Store` 를 사용했더라도 `DownStream` 관점에서는 `Un-Versioned Table` 로 간주된다. 
이는 `Stateful` 연산의 경우 기본적으로 상태 저장을 위해 `State Store` 를 사용하는데 기본으로 사용되는 `State Store` 가 `Un-Versioned State Store` 이기 때문이다. 
즉 `Versioned Table` 에서 `Stateful` 한 연산이 수행될 때 명시적으로 물리화를 지정하지 않아도 내부적으로 물리화를 진행하기 때문에, 
그 결과는 `Un-Versioned Table` 이 된다.  

만약 `Stateful` 한 연산을 수행하고 변환 결과를 명시적으로 `Versioned State Store` 로 저장한다면 이는 `Versioned Table` 로 간주된다. 
하지만 `Stateful` 연산이 생성하는 버전 이력이 불완전할 수 있으므로, `Stateful` 한 변관 결과를 `Versioned Table` 로 물리화할 때는 주의가 필요하다.  


![그림 1]({{site.baseurl}}/img/kafka/kafka-streams-versioned-state-store-12.drawio.png)


#### Versioned Table to Stream
`Versioned Table` 을 `KTable.toStream()` 을 사용해 스트림으로 변환한 다음 `KStream.toTable()` 을 통해 다시 테이블로 변환한다면, 
이는 다시 `Versioned State Store` 를 명시적으로 사용하지 않는 한 `Versioned Table` 이 되지 않는다.  


#### Summary
`Versioned Table` 변환을 결과가 `Versioned Table` 인지에 대한 내용을 요약하면, 
`DownStream` 관점에서 테이블이 `Versioned Table` 로 간주되려면 대부분 `UpStream` 에서 명시적으로 `Versioned State Store` 를 통해 물리화 돼야 한다. 
그리고 `Un-Versioned State Store` 를 통한 테이블 물리화, 명시적인 저장소 타입 지정이 없는 테이블 물리화, `Stateful` 연산, `KTable.toStream()` 이후 `KStream.toTable()` 
중 하나라도 스트림 처리 과정에 존재해서는 안된다.  

### Versioned State Store Demo
`Versioned State Store` 는 `Apache Kafka 3.5` 부터 `Kafka Streams` 에서 사용 할 수 있다. 
기존 이전 버전 사용자들이 `3.5` 버전으로 업그레이드하더라도 기존 `State Store` 는 이전 상태 저장소의 성격 그대로 사용가능하고, 
`Versioned State Store` 는 별도로 명시해야 사용 할 수 있다.  

`Streams DSL` 에서 `Versioned State Store` 사용을 위해서는 `Materialized` 를 통해 `Stores.persistentVersionedKeyValueStore` 를 전달하여 
생성할 수 있다.  

```java
streamsBuilder
    .table(
        topicName,
        Materialized.as(
            Stores.persistentVersionedKeyValueStore(
                STORE_NAME, 
                Duration.ofMillis(HISTORY_RETENTION)
            )
        )
    );
```  

`Processor API` 를 사용하는 경우 `addStateSotre` 에 `Stores.versionedKeyValueStoreBuilder` 를 사용해 생성할 수 있다.  

```java
streamsBuilder
    .addStateStore(
        Stores.versionedKeyValueStoreBuilder(
            Stores.persistentVersionedKeyValueStore(
                STORE_NAME, 
                Duration.ofMillis(HISTORY_RETENTION)),
            Serdes.Integer(),
            Serdes.String()
        )
    );
```  

`Versioned State Store` 를 생성할 때는 `history retention` 값을 꼭 적절하게 설정해야 하고, 
`3.5` 버전에서는 아직 `Versioned State Store` 에 대해 `In-Memory` 구현은 제공되지 않는다.  
