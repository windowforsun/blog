--- 
layout: single
classes: wide
title: "[Kafka] Kafka CDC Outbox"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka CDC Outbox 를 활용한 데이터 변경 캡쳐 및 전파 방식에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Connect
    - Debezium
    - CDC
    - Outbox
    - MSA
    - Event Driven
    - Kafka Topic
    - Kafka Producer
    - Kafka Consumer
    - Kafka Streams
toc: true
use_math: true
---  

## Kafka CDC Outbox
`Kafka CDC Outbox`(데이터 변경 캡쳐)는 `MSA` 에서 데이터의 일관성 있는 전파와 통합을 위한 강력한 패턴이다. 
`MSA` 를 구성하는 각 서비스에서 데이터를 변경 할 떄 발생하는 이벤트를 캡쳐하고 이를 `Kafka` 를 통해 다른 서비스로 전달하는 방식으로 동작한다. 
이를 활용하면 데이터 일관성을 유지하면서 서비스간 느슨한 결합을 구현 할 수 있다.  

`MSA` 는 각 서비스가 다루는 데이터를 전파하고 데이터 변경 사항을 전달하는 방식으로 구성된다. 
구매 주문을 관리하는 `MSA` 가 있다고 가정하면, 새로운 주문이 접수되면 그 주문에 대한 정보가 배승 서비스와 고객 서비스로 전달되는 것이 필요하다.  

주문 서비스가 새로운 주문에 대해 다른 서비스에 해당 사항을 전파하는 방식에는 다양한 방법이 있다. 
가장 대표적인 방법이 `REST API` 와 같은 것을 이용해 동기적으로 호출하는 것이다. 
하지만 이런 방식에는 의도치 않은 강한 결합을 생성할 수 있다. 
주문 서비스가 `REST API` 를 통해 각 서비스에 전파하기 위해서는 전파해야 하는 대상 서비스의 위치(도메인)과 
해당 서비스가 전파를 받을 수 있는 상태인지 등을 고려해야하 한다. 
만약 일시적으로 가용불가 상태라면 이후 재전송처리는 어떻게 할지도 고민이 필요하다.  

이러한 동기적인 전달방식의 문제점은 두 엔드포인트가 모두 존재하고 가용 가능상태여야 이벤트 전달이 가능하다는 것이다. 
또한 두 엔드포인트간 이벤트를 전달 할때 한쪽이 빠르게 전달하면, 받는 쪽은 그 속도에 맞춰 이벤트를 받아 처리해야 한다. 
즉 이벤트 전달에 있어서 버퍼링 혹은 `backpressure` 의 역할이 없으므로 이에 대한 고민도 필요하다. 
또한 새로 실행된 인스턴스의 경우 이전 이벤트를 수신할 수 없다는 점도 있다.  

이렇게 동기적인 방식에 따른 문제점들은 비동기 이벤트 전달 방식을 사용하면 해결 가능하다. 
즉 이벤트를 각 목적에 맞는 `Kafka Topic` 과 같은 것을 사용해서 이벤트를 전파하는 것이다. 
그리고 이벤트가 필요한 서비스는 해당 토픽을 구독해 이벤트 스트림상에 전달되는 변경사항을 받아 목적에 맞는 처리를 수행 할 수 있다.  

이러한 비동기 방식이 장점 중 하나는 이벤트를 전달하는 서비스와는 별도로 해당 이벤트가 필요로하는 서비스를 필요에 따라 추가할 수 있다는 점이다. 
이는 초기 구현에는 고려되지 못했던 다양한 케이스에 대해 유연하게 대응 가능하다는 장점이 있다. 
주문에 대한 이벤트를 기존에는 배송 서비스에서만 변경사항을 받아 처리했는데 
이후 더 추가적인 로깅 혹은 분석을 위해 `Elasticsearch` 에 저장하는 등으로 확장이 가능하다. 
또한 `Kafka Topic` 은 일정기간 메시지를 보존 할 수 있는데 이를 통해 이전 이벤트 히스토리를 파악한다던가, 
새롭게 투입되는 서비스에서도 이벤트 히스토리를 재생해 기반 데이터를 구축 할 수 있다.  

해당 포스팅에서 사용하는 코드 예시 및 데모 프로젝트의 전체 내용은 [여기](https://github.com/windowforsun/cdc-outbox-exam)
에서 확인 할 수 있다.  

### Dual Writes Issue
`Dual Writes` 이중 쓰기는 비동기 방식으로 이벤트를 전달 할 때 흔히 발생할 수 있는 문제점이다. 
주문 서비스와 배송 서비스가 비동기로 이벤트를 주고 받는다고 가정해보자. 
주문 서비스에 새로운 주문이 들어오면 주문 테이블에 새로운 레코드롤 `INSERT` 한 후 주문에 대한 `Kafka Topic` 에 이벤트를 발생 할 것이다. 
그리고 배송 서비스는 주문 `Topic` 에서 이벤트를 받아 배송에 대한 처리를 수행한다.  

하지만 이러한 비동기 방식은 [Kafka Consumer Duplication Patterns](2024-02-01-kafka-practice-kafka-duplication-patterns.md)
에서 설명한 것처럼 비동기라는 특성으로 인해 서로간 불일치가 발생 할 수 있다. 
이는 주문 서비스에서 `DB` 에 레코드를 `INSERT` 하고 `Apache Kafka` 에 메시지를 보내는 두 동작을 의미한다. 
그 이유는 `Apache Kafka` 와 데이터베이스를 하나의 트랜잭션으로 묶을 수 없기 때문이다. 
주문 레코드는 `INSERT` 됐지만, `Apache Kafka` 로 주문에 대한 이벤트 전달이 실패하는 것과 같은 증상이 충분히 발생 할 수 있다는 의미이다.  

이러한 문제점의 간단한 해결방법은 주문 서비스에서 `Apache Kafka` 혹은 데이터베이스 둘 중 하나에만 쓰기 동작을 수행해 일관성을 보장하는 것이다. 
우선 주문 서비스에서 `Apache Kafka` 에만 새로운 주문에 대한 이벤트를 전송하고 해당 이벤트를 받은 이후 새로운 주문 레코드를 `INSERT` 하는 경우이다. 
하지만 이런 방식을 사용하면 전체 주문을 조회하는 경우 비동기라는 특성으로 인해 새로운 주문이 누락될 수 있다는 문제점이 있다.  

다음으로 알아볼 방법이 데이터베이스에 동기적으로 쓰고, 이를 기반으로 `Apache Kafka` 에 메시지를 보내는 경우인데 이것을 `Outbox Pattern` 이라고 부른다. 
이에 대한 내용은 아래 챕터에서 별도로 다룬다.  


### Outbox Pattern
`Outbox Pattern` 의 기반 아이디어는 데이터베이스에 이벤트를 담는 `Outbox` 테이블을 두는 것이다. 
주문 서비스에서 새로운 주문이 들어오면 신규 주문에 대한 레코드를 `INSERT` 하고, 
동일 트랜잭션에서 `Outbox` 테이블에도 해당 이벤트를 나타내는 레코드를 `INSERT` 하는 방식이다.  

`Outbox` 테이블에 추가되는 레코드는 발생한 이벤트를 의미한다. 
이는 신규 주문을 의미하는 이벤트를 나타내는 `JSON` 일 수도 있고, 주문 변경에 대한 이벤트일 수도있다. 
그리고 필요에 따라 주문을 식별하는 식별자나 타입등의 이벤트 처리를 위한 다양한 컨텍스트를 포함 할 수 있다.  

그리고 `Outbox` 테이블의 변경을 모니터링하는 비동기 프로세스가 `Apache Kafka` 로 해당 테이블의 변경사항을 전파한다. 
이러한 개념을 통해 우리는 신규 주문에 대한 데이터를 `INSERT` 하는 동작과 신규 주문 이벤트를 전파하는 것을 하나의 데이터베이스 트랜잭션으로 묶을 수 있다. 
이는 신규 주문에 대한 데이터베이스 쓰기 동작과 이벤트 전파 동작이 모두 커밋되어 성공하거나, 둘중 하나라도 실패해 모두 실패하는 결과를 얻을 수 있다. 
또한 신규 주문 이후 바로 전체 주문을 조회하더라도 자신이 쓴 주문을 포함해 모두 조회 할 수도 있다.  

여기서 우리가 좀 더 알아봐야 할 것은 바로 `Outbox` 테이블의 변경사항을 어떻게 모니터링 하고, 
이를 `Apache Kafka` 로 밀어 넣어주냐인데 우리는 `Debezium` 과 `CDC`(데이터 변경 캡쳐)를 사용해 이를 손쉽게 해결하고 활용 할 수 있다.  


### CDC
`CDC` 는 소스의 변경을 캡쳐해 타겟이 되는 곳으로 전송하는 것을 의미한다. 
여기서는 `Apache Kafka` 에서 활용할 수 있는 `Debezium CDC` 를 사용할 예정이므로 이를 기준으로 내용을 기술한다. 
`Debezium CDC` 는 테이블을 폴링(조회)하는 방식이 아닌 변경 로그를 기반으로 소스가 되는 데이터 테이블의 변경사항을 캡쳐한다. 
그러므로 다른 방식의 캡쳐와 비교해 낮은 오버해드와 실시간성 및 정확도에 큰 이점이 있다. 
관련해서는 [여기](https://debezium.io/blog/2018/07/19/advantages-of-log-based-change-data-capture/)
에서 상세한 내용을 확인 할 수 있다.   

이후 예제에서는 앞서 설명한 `Outbox Table` 을 `CDC` 를 통해 `Apache Kafka Topic` 에 실시간 스트리밍하는 목적으로 활용할 예정이다. 
그러면 `Outbox Topic` 을 구독하는 여타 서비스에서 이벤트를 받아 필요한 비지니스를 목적에 맞게 수행할 수 있다.  

아래 그림을 보면 데모 애플리케이션 구성과 전체적인 이벤트 흐름을 파악할 수 있다. 
`order-service` 에서 새로운 주문 레코드를 `INSERT` 하고 동일 트랜잭션에서 `Outbox` 테이블에도 해당 이벤트를 나타내는 레코드를 `INSERT` 한다. 
그리면 `CDC` 가 `Outbox` 테이블의 변경을 캡쳐해 이를 `Apache Kafka Topic` 으로 전송하게 된다. 
그리고 해당 `Topic` 을 구독하는 `shipment-service` 는 이벤트 내용을 받아 새로운 배송 레코드를 `INSERT` 하게된다. 
이는 주문이 수정(취소)되거나 배송이 완료되어 주문 서비스에 알려주는 경우도 동일한 흐름으로 진행된다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-cdc-outbox-1.drawio.png)  


### Outbox Table
데모에서 사용할 `Outbox Table` 의 스키마는 아래와 같다. 

Column|	Type|	Modifiers
---|---|---
id|	uuid|	not null
aggregatetype|	character varying(255)|	not null
aggregateid|	character varying(255)|	not null
type|	character varying(255)|	not null
payload|	jsonb|	not null

각 스키마 필드의 설명은 아래와 같다. 

- `id` : 각 이벤트 메시지의 고유 `ID` 로 소비자의 실패 상황 등에 재시도가 이뤄질 때 중복 이벤트를 김자하는 목적으로 사용될 수 있다. 
- `aggregatetype` : 이벤트를 `Kafka Topic` 으로 라우팅 할때 사용된다. `주문 서비스` 와 `배송 서비스` 가 있다면 구매에 대한 `Topic` 과 배송에 대한 `Topic` 으로 나눌 수 있다. 해당 타입은 동일한 `aggregate` 내에 포함되는 이벤트라면 동일한 유형을 사용해야 한다. 만약 주문 취소 이벤트가 있다면 이또한 주문과 동일한 유형을 사용해야 한다. 즉 주문 취소도 주문 `Topic` 에 들어가도록 해야한다. 
- `aggregateid` : 각 `aggregatetype` 에서 고유하게 식별될 수 있는 아이디로 이벤트 아이디라고 할 수 있다. 이는 이후 `Kafka Topic` 의 키로 사용된다는 점을 기억해야 한다. `주문 서비스`와 `배송 서비스` 가 있다면 개별 주문의 `ID` 와 개별 배송의 `ID` 가 사용 될 수 있다. 만약 주문 취소 이벤트라면 이또한 동일한 `aggregatetype` 을 사용하므로 주문시에 사용된 주문 `ID` 를 동일하게 사용해야 한다. 이러한 방식으로 각 이벤트를 구독해서 소비할 때 동일한 `aggregatetype`(`Topic`) 이라면 모든 이벤트를 생성된 순서대로 소비할 수 있게 된다. 
- `type` : 이벤트의 유형으로 `OrderCreate` 구매 주문, `OrderCancel` 주문 취소가 될 수 있다. 이는 `Topic` 에서 이벤트를 소비했을 때 적절한 이벤트 핸들러를 트리거할 수 있도록 한다. 
- `payload` : 실제 이벤트 내용을 담는 `JSON` 필드이다. 구매 주문 이벤트라면 주문 테이블에 대한 정보와 더불어 추가적으로 구매자 정보 등이 포함 될 수 있다.  

### Send Outbox Events
이벤트를 `Outbox` 로 전송한다는 것은 각 서비스에서 전파하고 싶은 이벤트를 `Outbox Table` 에 `INSERT` 하는 것을 의미한다. 
이벤트는 이후 변경사항에 대해 유연성을 가지는 것이 필요하므로 추상화된 `API` 를 사용하는 것이 좋다. 
그래서 `OutboxEvent` 라는 인터페이스를 정의해 `Outbox` 로 전송이 필요한 세부 이벤트들은 이를 구현해 `Outbox Table` 에 추가 될 수 있도록 한다.  

```java
public interface OutboxEvent {
	static ObjectMapper objectMapper = new ObjectMapper();
	String getAggregateId();
	String getAggregateType();
	String getOutboxPayload() throws JsonProcessingException;
	String getType();
	Instant getTimestamp();
}
```  

주문 서비스에서 구매 주문과 주문 변경을 처리하는 서비스 코드에서 아래와 같이 목적에 맞는 비지니스 처리 후 `Outbox` 이벤트를 추가하는 데 사용된다. 

```java
public class OrderService {
	private final OrderLineRepository orderRepository;
	private final OutboxService outboxService;

	@Transactional
	public OrderLine addOrder(OrderLine order) throws JsonProcessingException {
		log.info("addOrder {}", order);
		// 구매 주문 데이터 추가
		this.orderRepository.save(order);

		OrderCreateEvent orderCreateEvent = OrderCreateEvent.of(order);

		// 구매 주문 이벤트 전파
		this.outboxService.exportEvent(Outbox.of(orderCreateEvent));

		return order;
	}

	@Transactional
	public OrderLine updateOrder(long orderId, OrderStatus newStatus) throws JsonProcessingException {
		OrderLine order = this.orderRepository.findById(orderId).orElse(null);

		if (order == null) {
			throw new RuntimeException("order id " + orderId + " not found");
		}

		OrderStatus oldStatus = order.getStatus();
		order.setStatus(newStatus);
		// 주문 변경 데이터 수정
		order = this.orderRepository.save(order);

		OrderUpdateEvent orderUpdateEvent = OrderUpdateEvent.of(orderId, newStatus, oldStatus);
        // 주문 변경 이벤트 전파
		this.outboxService.exportEvent(Outbox.of(orderUpdateEvent));

		return order;
	}
}
```  

`addOrder()` 와 `updateOrer()` 메소드는 구매 주문처리와 주문 변경 처리를 수행하고 데이터베이스에 저장한다. 
그리고 해당 데이터를 사용해서 `OrderCreateEvent` 와 `OrderUpdateEvent` 를 생성하고 `Outbox Table` 에 `INSERT` 하여 전파한다. 
여기서 각 비지니스 데이터를 `INSERT` 하는 것과 이벤트 전파를 위해 `Outbox Table` 에 `INSERT` 하는 것은 동일한 트랜잭션에서 수행된다. 
즉 구매 주문 1건이 들어오면 주문 테이블과 `Outbox Table` 에 각 1개씩 총 2개의 레코드가 추가된다.  

아래는 `OrderCreateEvent` 를 생성하는 구현 클래스의 내용이다.  

```java
public class OrderCreateEvent  implements OutboxEvent{
	private final OrderCreateMessage orderCreateMessage;
	private final Instant timestamp;

	private OrderCreateEvent(long id, OrderLine order) {
		this.orderCreateMessage = OrderCreateMessage.builder()
			.id(order.getId())
			.item(order.getItem())
			.status(order.getStatus())
			.quantity(order.getQuantity())
			.totalPrice(order.getTotalPrice())
			.build();
		this.timestamp = Instant.now();
	}

	public static OrderCreateEvent of(OrderLine order) {
		return new OrderCreateEvent(order.getId(), order);
	}

	@Override
	public String getAggregateId() {
		return String.valueOf(this.orderCreateMessage.getId());
	}

	@Override
	public String getAggregateType() {
		return "Order";
	}

	@Override
	public String getOutboxPayload() throws JsonProcessingException {
		return OutboxEvent.objectMapper.writeValueAsString(this.orderCreateMessage);
	}

	@Override
	public String getType() {
		return "OrderCreate";
	}

	@Override
	public Instant getTimestamp() {
		return this.timestamp;
	}
}
```  

이벤트의 정보를 담고 있는 `payload` 는 `ObjectMapper` 를 사용해서 이벤트 데이터를 `JSON` 형식으로 표한하고 있다. 
그리고 아래는 이벤트 전파를 수행하는 서비스 코드이다.  

```java
public class OutboxService {
	private final OutboxRepository outboxRepository;

	@Transactional
	public void exportEvent(Outbox outbox) {
		this.outboxRepository.save(outbox);
		this.outboxRepository.deleteById(outbox.getId());
	}
}
```  

`exportEvent()` 에 전파하고자 하는 이벤트 객체를 넣어주면 `Outbox Table` 에 해당 레코드가 추가되고 바로 삭제된다. 
이러한 방식으로 이벤트를 전파할 수 있는 이유는 앞서 설명한 `CDC` 동작 방식의 특성때문이다. 
`CDC` 는 실제 테이블을 검사하는 것이 아니라 트랜잭션 로그를 추적하는 방식을 사용한다. 
그러기 때문에 `save()` 후 바로 `delete()` 를 수행하더라도 해당 트랜잭션이 커밋되는 순간 `INSERT` 와 `DELETE` 로그가 생성된다. 
그 후 `CDC` 는 이러한 이벤트 로그를 사용해서 `INSERT` 인 경우 `Apache Kafka` 로 전송해주면 테이블에서 레코드는 삭제되지만 이벤트 전파는 가능하다. 
`DELETE` 의 이벤트도 `CDC` 에서 감지가 가능하지만 이는 옵션을 통해 `INSERT` 에 대한 내용만 감지하도록 할 수 있다. 
이러한 방식의 장점은 `CDC Outbox` 방식으로 이벤트는 계속해서 전파 가능하지만 `Outbox Table` 는 항상 비어 있다는 점이다. 
이는 추가적인 디스크 공간이 필요하지 않기 때문에 별도 관리를 위한 프로세스를 구축해 줄 필요없다. 
하지만 만약 `Outbox Table` 에서 이벤트의 이력관르를 하고 싶다면 `save()` 만 수행하고, 
이후 별도의 프로세스에서 `Outbox Table` 의 보존기한을 두고 주기적으로 삭제해주는 관리가 필요하다.  


### Debezium CDC Connector
위 내용까지해서 `Outbox Table` 에 이벤트를 `INSERT` 하는 구현까지는 알아보았다. 
이제 `Debezium CDC Connector` 를 등록해서 `Outbox Table` 에 추가되는 이벤트를 캡쳐하고 이를 `Apache Kafka` 로 전달하는 방법에 대해 알아본다. 
아래 `JSON` 요청은 `Kafka Connect` 에 `Debezium CDC Connector` 를 등록하는 `REST API` 의 `POST Body` 내용이다.  

```json
{
  "name": "cdc-outbox",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "exam-db",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.name": "exam-db",
    "database.history.kafka.topic": "cdc-outbox",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.allowPublicKeyRetrieval" : "true",
    "tombstones.on.delete" : "false",
    "table.include.list": "exam.outbox",
    "value.converter" : "org.apache.kafka.connect.storage.StringConverter",
    "key.converter" : "org.apache.kafka.connect.storage.StringConverter",
    "transforms": "outbox",
    "transforms.outbox.type" : "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.fields.additional.placement" : "type:header:type",
    "value.converter.schemas.enable": "false",
    "value.converter.delegate.converter.type": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```  

`io.debezium.connector.mysql.MySqlConnector` 를 사용해서 `Kafka Connector` 를 등록한다. 
그러면 `JSON` 에 설정된 `DB` 와 해당하는 `Table` 의 변경사항을 해당 `Connector` 가 캡쳐하고 이를 설정된 `Apache Kafka` 의 `Topic` 으로 전송한다. 
이후에 설명하겠지만 전송하는 `Topic` 은 `io.debezium.transforms.outbox.EventRouter` 를 사용하면 간편하게 각 `aggregatetype` 별로 전송 `Topic` 을 라우팅 할 수 있다. 


### Topic Routing
`Outbox Table` 을 기반으로 하는 이벤트 전파는 해당 이벤트의 `root context` 를 의미하는 `aggregatetype` 을 기반으로 라우팅이 필요하다. 
초창기에는 이러한 라우팅을 위해 커스텀한 `SMT` 을 구현하였지만, 이제 `Debeizum` 에서 제공하는 [io.debezium.transforms.outbox.EventRouter](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
를 사용해 쉽게 적요 할 수 있다. 
이 `SMT` 는 `Outbox Table` 의 변경 이벤트를 캡쳐해서 각 이벤트 유형(`aggregatetype`)에 맞는 `Kafka Topic` 으로 메시지를 라우팅한다. 
그러므로 특정 `aggregatetype` 이벤트에 관심있는 소비자는 해당하는 이벤트 유형만 받아 처리 할 수 있다.  


### Events in Kafka
실제로 `Apache Kafka` 를 통해 전파되는 이벤트에 대해 알아본다. 
전파가 되는 이벤트로는 `OrderCreate`, `OrderUpdate`, `ShipmentUpdate` 가 있다. 
이벤트를 받기 위해서는 우선 환경 구축이 필요하다. 
전체 구성 및 프로젝트는 [여기](https://github.com/windowforsun/cdc-outbox-exam)
에서 확인 할 수 있다. 
우선 아래 명령을 프로젝트 루트 경로에서 실행해 `order-service` 와 `shipment-service` 를 로컬 `Docker Image` 로 빌드한다. 

```bash
$ ./gradlew order-service:jibDockerBuild

$ ./gradlew shipment-service:jibDockerBuild
```  

그리고 `/docker` 경로로 이동 후 `docker-compose` 명령으로 전체 구성을 실행한다.  

```bash
$ docker-compose up --build
```  

데모 실행에 필요한 모든 컨테이너가 실행 됐으면 우선 모드 정상적으로 구성 됐는지 아래 명령어들로 확인 한다.  

```bash
$ docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "show tables"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------------+
| Tables_in_exam   |
+------------------+
| consumed_message |
| order_line       |
| outbox           |
| shipment         |
+------------------+

$ curl localhost:8083/connector-plugins | jq
[
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "1.5.0.Final"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "type": "sink",
    "version": "7.0.10-ccs"
  },
  {
    "class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "type": "source",
    "version": "7.0.10-ccs"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "1"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "1"
  }
]

$ docker exec -it myKafka \
> kafka-topics.sh --bootstrap-server localhost:9092 --list
__consumer_offsets
outbox.event.Order
outbox.event.Shipment
```  

위 `outbox.event.Order`, `outbox.event.Shipment` 토픽은 `order-service`, `shipment-service` 에서 해당 토픽을 구독하는 `Consumer` 가 있기 때문에 초기 구성부터 생성될 수 있다. 
모든 구성의 정상 실행이 확인 되면 먼저 `Kafka Connect` 컨테이너에 `Debeizum CDC Connector` 를 등록해 실행 한다.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
> --data @docker/cdc-outbox.json \
> http://localhost:8083/connectors | jq
{
  "name": "cdc-outbox",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "exam-db",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.name": "exam-db",
    "database.history.kafka.topic": "cdc-outbox",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.allowPublicKeyRetrieval": "true",
    "tombstones.on.delete": "false",
    "table.include.list": "exam.outbox",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.fields.additional.placement": "type:header:type",
    "value.converter.schemas.enable": "false",
    "value.converter.delegate.converter.type": "org.apache.kafka.connect.json.JsonConverter",
    "name": "cdc-outbox"
  },
  "tasks": [],
  "type": "source"
}
```  

`Kafka Connector` 등록 및 실행 상태는 `/status` 를 통한 요청과 필요한 `Kafka Topic` 이 생성됐는지로 확인 할 수 있다.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
> --data @docker/cdc-outbox.json \
> http://localhost:8083/connectors | jq
{
  "name": "cdc-outbox",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "exam-db",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.name": "exam-db",
    "database.history.kafka.topic": "cdc-outbox",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.allowPublicKeyRetrieval": "true",
    "tombstones.on.delete": "false",
    "table.include.list": "exam.outbox",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.fields.additional.placement": "type:header:type",
    "value.converter.schemas.enable": "false",
    "value.converter.delegate.converter.type": "org.apache.kafka.connect.json.JsonConverter",
    "name": "cdc-outbox"
  },
  "tasks": [],
  "type": "source"
}

$  docker exec -it myKafka kafka-topics.sh \
> --bootstrap-server localhost:9092 \
>  --list
__consumer_offsets
cdc-outbox
cdc-outbox-config
cdc-outbox-offset
cdc-outbox-status
exam-db
outbox.event.Order
outbox.event.Shipment
```  

`cdc-outbox` 로 시작하는 토픽과 `exam-db` 관련 토픽이 생성된 것을 확인 할 수 있다.  

이제 필요한 모든 구성은 완료된 상태이다. 
이벤트 흐름을 살펴보기 위해 `Outblx Table` 에 추가되는 레코드는 지우지 않고 유지하는 형태로 수정 후 진행한다. 
먼저 `OrderCreate` 이벤트는 `order-service` 에 `POST /order` 를 통해 구매 주문이 들어왔을 때 수행되는 이벤트로 아래와 같은 흐름을 갖는다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-cdc-outbox-2.drawio.png)

구매 주문을 수행한 후 `DB` 테이블과 `Kafka Topic` 을 조회하면 아래와 같다.  

```bash
$  curl -X POST \
> -H "Content-Type: application/json" \
> localhost:8080/order \
> --data '{
quote>  "item" : "test1",
quote>  "quantity" : 1,
quote>  "totalPrice" : 101,
quote>  "status" : "ENTERED"
quote> }' | jq
{
  "id": 1,
  "item": "test1",
  "quantity": 1,
  "totalPrice": 101,
  "status": "ENTERED"
}

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from order_line"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------+----------+-------------+---------+
| id | item  | quantity | total_price | status  |
+----+-------+----------+-------------+---------+
|  1 | test1 |        1 |         101 | ENTERED |
+----+-------+----------+-------------+---------+

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from shipment"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+---------+
| id | order_id | status  |
+----+----------+---------+
|  1 |        1 | ENTERED |
+----+----------+---------+

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from outbox"
+----+---------------+-------------+-------------+--------------------------------------------------------------------------+
| id | aggregatetype | aggregateid | type        | payload                                                                  |
+----+---------------+-------------+-------------+--------------------------------------------------------------------------+
|  1 | Order         | 1           | OrderCreate | {"id":1,"item":"test1","quantity":1,"totalPrice":101,"status":"ENTERED"} |
+----+---------------+-------------+-------------+--------------------------------------------------------------------------+

$  docker exec -it myKafka kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --topic outbox.event.Order \
> --property print.key=true \
> --property print.headers=true \
> --from-beginning 
id:1,type:OrderCreate   1       {"id":1,"item":"test1","quantity":1,"totalPrice":101,"status":"ENTERED"}
```  

구매 주문이 들어오게 되면 `OrderLine` 테이블에 구매 주문에 대한 레코드가 생성되고 해당 이벤트는 `Outbox Table` 를 통해 `outbox.event.Order` 토픽으로 전달된다. 
그러면 해당 토픽을 구독하는 `shipment-service` 에서 `Shipment` 테이블에 배송에 대한 레코드를 생성하게 된다.  

`OrderUpdate` 는 `order-service` 에 `PUT /order` 요청을 통해 기존 구매 주문내용을 갱신했을 떄 발생하는데 그 흐름을 도식화하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-cdc-outbox-3.drawio.png)

주문 변경을 수행한 후 `DB` 테이블과 `Kafka Topic` 을 조회하면 아래와 같다.

```bash
$  curl -X PUT \
> -H "Content-Type: application/json" \
> localhost:8080/order \
> --data '{
quote>   "orderId" : 1,
quote>   "newStatus" : "CANCELLED"
quote> }' | jq
{
  "id": 1,
  "item": "test1",
  "quantity": 1,
  "totalPrice": 101,
  "status": "CANCELLED"
}

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from order_line"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------+----------+-------------+-----------+
| id | item  | quantity | total_price | status    |
+----+-------+----------+-------------+-----------+
|  1 | test1 |        1 |         101 | CANCELLED |
+----+-------+----------+-------------+-----------+

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from shipment"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+----------+
| id | order_id | status   |
+----+----------+----------+
|  1 |        1 | CANCELED |
+----+----------+----------+

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from outbox"
+----+---------------+-------------+-------------+--------------------------------------------------------------------------+
| id | aggregatetype | aggregateid | type        | payload                                                                  |
+----+---------------+-------------+-------------+--------------------------------------------------------------------------+
|  1 | Order         | 1           | OrderCreate | {"id":1,"item":"test1","quantity":1,"totalPrice":101,"status":"ENTERED"} |
|  2 | Order         | 1           | OrderUpdate | {"orderId":1,"newStatus":"CANCELLED","oldStatus":"ENTERED"}              |
+----+---------------+-------------+-------------+--------------------------------------------------------------------------+

$  docker exec -it myKafka kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --topic outbox.event.Order \
> --property print.key=true \
> --property print.headers=true \
> --from-beginning 
id:1,type:OrderCreate   1       {"id":1,"item":"test1","quantity":1,"totalPrice":101,"status":"ENTERED"}
id:2,type:OrderUpdate   1       {"orderId":1,"newStatus":"CANCELLED","oldStatus":"ENTERED"}
```  

주문 변경이 들어오면 이후 수행되는 흐름은 먼저 살펴본 구매 주문 흐름과 동일하다. 
예시는 주문 상태가 `CANCELED` 로 변경되는 상황인데 해당 이벤트는 `OrderUpdate` 값이 `Outbox` 테이블의 `type` 필드에 사용된다. 
그리소 해당 이벤트를 구독하는 `shipment-service` 는 주문 변경에 해당하는 자신의 `Shipment` 테이블에 레코드의 상태도 `CANCELED` 로 변경한다.  

마지막으로 `ShipmentUpdate` 이벤트는 배송이 완료된 경우 `shipment-service` 로 `PUT /shipment` 요청을 통해 
배송 상태를 변경했을 때의 이벤트이다. 이를 흐름을 도식화 하면 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-cdc-outbox-4.drawio.png)

배송 완료을 수행한 후 `DB` 테이블과 `Kafka Topic` 을 조회하면 아래와 같다. 
`id` 가 2변인 새로운 구매 주문을 생성 후 진행한다.  

```bash
$  curl -X POST \
> -H "Content-Type: application/json" \
> localhost:8080/order \
> --data '{
quote>  "item" : "test2",
quote>  "quantity" : 1,
quote>  "totalPrice" : 101,
quote>  "status" : "ENTERED"
quote> }' | jq
{
  "id": 2,
  "item": "test2",
  "quantity": 1,
  "totalPrice": 101,
  "status": "ENTERED"
}

$  curl -X PUT \
> -H "Content-Type: application/json" \
> localhost:8081/shipment \
> --data '{
quote>   "shipmentId" : 2,
quote>   "newStatus" : "DONE"
quote> }' | jq
{
  "id": 2,
  "orderId": 2,
  "status": "DONE"
}

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from order_line"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------+----------+-------------+-----------+
| id | item  | quantity | total_price | status    |
+----+-------+----------+-------------+-----------+
|  1 | test1 |        1 |         101 | CANCELLED |
|  2 | test2 |        1 |         101 | SHIPPED   |
+----+-------+----------+-------------+-----------+

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from shipment"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+----------+----------+
| id | order_id | status   |
+----+----------+----------+
|  1 |        1 | CANCELED |
|  2 |        2 | DONE     |
+----+----------+----------+

$  docker exec -it exam-db \
> mysql -uroot -proot -Dexam -e "select * from outbox"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+---------------+-------------+----------------+--------------------------------------------------------------------------+
| id | aggregatetype | aggregateid | type           | payload                                                                  |
+----+---------------+-------------+----------------+--------------------------------------------------------------------------+
|  1 | Order         | 1           | OrderCreate    | {"id":1,"item":"test1","quantity":1,"totalPrice":101,"status":"ENTERED"} |
|  2 | Order         | 1           | OrderUpdate    | {"orderId":1,"newStatus":"CANCELLED","oldStatus":"ENTERED"}              |
|  3 | Order         | 2           | OrderCreate    | {"id":2,"item":"test2","quantity":1,"totalPrice":101,"status":"ENTERED"} |
|  4 | Shipment      | 2           | ShipmentUpdate | {"shipmentId":2,"orderId":2,"newStatus":"DONE","oldStatus":"ENTERED"}    |
+----+---------------+-------------+----------------+--------------------------------------------------------------------------+

$  docker exec -it myKafka kafka-console-consumer.sh \
> --bootstrap-server localhost:9092 \
> --topic outbox.event.Shipment \
> --property print.key=true \
> --property print.headers=true \
> --from-beginning 
id:4,type:ShipmentUpdate        2       {"shipmentId":2,"orderId":2,"newStatus":"DONE","oldStatus":"ENTERED"}
```  

배송 완료 요청이 들어오면 `Shipment` 테이블에 해당 배송 레코드의 상태가 `DONE` 으로 변경된다. 
그리고 배송 완료에 대한 이벤튼는 `outbox.event.Shipment` 토픽으로 전달된다. 
그러면 해당 토픽을 구독하는 `order-service` 는 `payload` 에 있는 `orderId` 를 사용해서 해당하는 구매 주문의 상태를 `DONE` 으로 변경한다. 

### Duplication Event in Consuming Service
`Apache Kafka` 의 `Topic` 을 구독해 처리하는 서비스에서는 중복 메시지 처리에 대한 방안 마련이 필요하다. 
지금까지 알아본 `CDC Outbox` 는 구매 주문이 발생 했을 때 새로운 구매 주문 레코드 추가와 구매 주문 발생 이벤트를 하나의 트랜잭션으로 묶는 역할을 하기 때문이다. 
`outbox.event.Order` 토픽에 메시지가 추가 됐다는 것은 새로운 구매 주문이 들어왔다는 것은 절대적으로 보장하지만, 
이를 구독하는 `Consumer` 가 메시지를 한번만 받는다는 것은 보장하지 않는다. 

이에 대한 가장 간단한 예시로 `outbox.event.Order` 를 구독하는 `shipment-service` 의 `Consumer` 가 배포와 같은 경우로 재시작 되는 겅우를 가정해보자. 
새로운 `shipment-service` 가 실행되면 `Consumer` 도 새롭게 초기화 된다. 
그러면 `Kafka Consumer` 의 정책에 따라 새로운 `Consumer` 는 `outbox.event.Order` 토픽의 처음 부터 끝까지 메시지를 다시 수신하게 된다. 
위와 같은 상황에 만약 `shipment-service` 의 `Consumer` 에서 이벤트 중복 처리에 대한 내용이 없다면 다수의 이벤트가 중복 처리 될 것이다.  

아래 `shipment-service` 에서 `outbox.event.Order` 토픽을 처리하는 구현 내용을 살펴보자. 

```java
public class OutboxConsumer {
	private final ShipmentService shipmentService;
	private final MessageLogService messageLogService;
	private static ObjectMapper objectMapper = new ObjectMapper();

	@Transactional
	@KafkaListener(topics = "outbox.event.Order")
	public void listen(String outboxEvent, @Header(name = "type") String type, @Header(name = "id") String id) {
		log.info("shipment received outboxEvent : {}", outboxEvent);
		long eventId = Long.parseLong(id);

		if(this.messageLogService.isProcessed(eventId)) {
			log.info("Event id : {} was already processed, ignored", eventId);
			return;
		}

		switch (type) {
			case "OrderCreate":
				OrderCreateMessage orderCreateEvent = this.<OrderCreateMessage>parseMessage(OrderCreateMessage.class, outboxEvent);
				log.info("received orderCreateEvent : {}", orderCreateEvent);
				this.shipmentService.addShipment(orderCreateEvent);
				break;
			case "OrderUpdate":
				OrderUpdateMessage orderUpdateEvent = this.<OrderUpdateMessage>parseMessage(OrderUpdateMessage.class, outboxEvent);
				log.info("received orderUpdateEvent : {}", orderUpdateEvent);
				this.shipmentService.updateShipment(orderUpdateEvent);
				break;
		}

		this.messageLogService.processed(eventId);
	}

	public <T> T parseMessage(Class<T> clazz, String event) {
		T msg = null;
		try {
			msg = this.objectMapper.readValue(event, clazz);
		} catch (Exception e) {
			log.error("order event convert fail : " + event, e);
		}

		return msg;
	}
}
```  

이벤트 중복을 판별하기 위해 `Header` 에 있는 `id` 를 받아 `isProcessed()` 를 통해 중복 검사를 수행한다. 
처리한 적이 있다면 현재 수신한 메시지는 아무런 처리를 하지 않고 리턴된다. 
그리고 처리한 적이 없는 메시지는 기존 처리를 수행 완료 후 `processed()` 를 호출해 해당 `id` 의 처리가 완료 됐음을 표시한다. 
아래는 이러한 중복 검사가 구현된 `MessageLogService` 의 구현 내용이다. 

```java
public class MessageLogService {
	private final ConsumedMessageRepository consumedMessageRepository;

	@Transactional(value = Transactional.TxType.MANDATORY)
	public void processed(Long eventId) {
		this.consumedMessageRepository.save(ConsumedMessage.builder()
				.eventId(eventId)
				.timeOfReceived(Instant.now())
			.build());
	}

	@Transactional(value = Transactional.TxType.MANDATORY)
	public boolean isProcessed(Long eventId) {
		return this.consumedMessageRepository.findById(eventId).orElse(null) != null;
	}
}
```  

이벤트 처리 중복검사를 위해서는 `ConsumedMessage` 라는 `Entity` 와 `Repository` 를 구성해 `DB` 테이블에 저장해서 관리하는 방식을 사용한다. 
그리고 정의된 모든 메소드는 `@Transactional` 을 선언해 이벤트 처리시 사용하는 트랜잭션에서 수행될 수 있도록 한다. 
이러한 방식을 사용하면 이벤트 처리 트랜잭션이 어떠한 사유로든 롤백되면 메시지도 처리된 것으로 표시되지 않고, 재시도 등을 통해 사후 처리를 진행 할 수 있다.  

데모에서 구현된 내용보다 보다 완성도를 높이고 싶다면 이벤트 처리 실패시 재시도 횟수와 실패에 따른 `dead-letter-topic` 으로 전송하는 등의 구현을 고려 할 수 있다. 
이런 `Consumer` 의 메시지 재처리에 대한 내용은 [여기]({{site.baseurl}}{% link _posts/kafka/2024-04-15-kafka-practice-kafka-consumer-non-block-retry-with-spring.md %})
에서 확인 할 수 있다. 
또한 `consumed_message` 테이블에 대한 관리 정책도 필요한데, 
이는 현재 `Consumer` 의 `Offsets` 로 커밋된 메시지보다 오래된 이벤트는 테이블에서 삭제하는 방식으로 관리 할 수 있다.  


---  
## Reference
[Reliable Microservices Data Exchange With the Outbox Pattern](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern)  
[Outbox Event Router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
[Five Advantages of Log-Based Change Data Capture](https://debezium.io/blog/2018/07/19/advantages-of-log-based-change-data-capture/)


