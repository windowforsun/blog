--- 
layout: single
classes: wide
title: "[Kafka] Kafka Consumer Non-Blocking Retry Pattern"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Consumer 에서 사용할 수 있는 Non-Blocking Retry 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Retry
    - Non Blocking
    - Consumer
toc: true
use_math: true
---

## Kafka Consumer Non-Blocking Retry
`Kafka` 의 `Topic Partition` 에서 에서 메시지를 소비하고 재시도를 수행하고자 한다면, 
`Blocking Retry` 와 `Non-Blocking Retry` 중 하나의 방식을 택해야 한다. 
`Blocking Retry` 는 재시도가 완료 되어야 `Topic Partition` 의 다음 메시지를 소비해서 이벤트 처리를 재개할 수 있다. 
이와 비교해서 `Non-Blocking Retry` 는 `Topic Partition` 의 메시지는 계속해서 소비하고, 
재시도가 필요한 메시지는 별도의 비동기 방식을 사용해 구현한다. 
`Non-Blocking Retry` 는 기존 `Topic Parition` 메시지 소비를 중단 없이 할 수 있지만, 
기존 메시지 순서는 보자될 수 없을 기억해야 한다. 

[Non-Blocking Retry Spring Topics]({{site.baseurl}}{% link _posts/kafka/2024-04-15-kafka-practice-kafka-consumer-non-block-retry-with-spring.md %})
에서도 `Non-Blocking Retry` 에 대한 내용과 간단한 구현 방법에 대해 알아보았다. 
하지만 위 예제는 `Spring Kafka` 를 사용하는 환경에서만 간단한 `Annotation` 정으를 통해서 적용 가능하다. 
이번 포스팅에서는 `Spring` 이나 `Kafka` 가 아닌 다른 메시지 미들웨어를 사용하더라도 도입할 수 있는 
`Non-Blocking Retry Pattern` 에 대해 알아볼 것이다.  

예제의 내용은 [Kafka Consumer Retry](({{site.baseurl}}{% link _posts/kafka/2024-04-02-kafka-practice-kafka-consumer-retry.md %})
동일하므로 관련 내용을 참고 할 수 있다.  

### Non-Blocking Retry Pattern
`Create Event` 와 `Update Event` 가 있을 떄, 
`Create Event` 로 아이템이 저장되고, `Update Event` 가 전달된다면 모든 이벤트는 정상 처리 될 수 있다. 
하지만 `Create Event` 전 `Update Event` 가 먼저 온다면 `Update Event` 는 `Create Event` 가 수행 완료 되는 시점까지 대기해야 한다. 
그리고 이벤트가 처리 될 수 있는 최대 시간과 이벤트의 재시도 주기의 시간값을 통해 재시도를 조절 한다.  

`Non-Blocking Retry` 를 위해서는 재시도용 `Topic` 을 사용한다. 
특정 이벤트의 재시도가 필요한 경우 해당 이벤트를 재시도용 토픽으로 보내고, 
이를 다시 소비해서 재시도 여부 판별 후 재시도 수행이 필요하다면 다시 원본 토픽으로 이벤트를 전송하게 된다. 
아래는 이러한 과정을 도식화한 내용이다.


![그림 1]({{site.baseurl}}/img/kafka/kafka-consumer-non-blocking-retry-pattern-1.png)

- 현재 시간과 `원본 이벤트 수신 시간` 차이가 `maxRetryDuration` 을 넘어가면 해당 이벤트는 버린다. 
- 위에서 버려지지 않은 이벤트는 현재 시간과 `재시도 수신 시간` 차이가 `retryInterval` 보다 크면 재시도를 수행한다. 
- 재시도 판별이 완료된 이벤트는 원본 토픽에 쓰여지고 다시 소비되고 처리된다. 

아래 그림은 `UpdateEvent` 가 수신 됐을 때 처리되는 전체 괴정을 보여준다.   


![그림 1]({{site.baseurl}}/img/kafka/kafka-consumer-non-blocking-retry-pattern-2.png)


소개한 `Non-Blocking Retry` 패턴은 원본 토픽에서 처음 수신된 시간을 사용해서 
`maxRetryDuration` 을 초과했는지 검사하는데 이를 위해
처음 수신 시점의 시간 값을 계속해서 재시도 토픽, 원본 토픽에 쓰여질 때마다 해당 값을 해더로 지니고 있는다. 
그리고 원본 토픽 이름도 해더에 추가해서 재시도가 필요한 시점에 다시 원본 토픽으로 이벤트를 전송하게 된다.  

이 패턴의 핵심은 이벤트가 재시도 자격을 충복하면 원본 토픽으로 전송해서, 다시 주어진 처리가 수행될 수 있도록 한다. 
그리고 처리 수행 도중 재시도가 필요하다고 판단되면 다시 재시도 토픽으로 전달돼 재시도 여부 판별 부터 
`maxRetryDuration` 로 만료 될 떄까지 수행되는 것이다. 


## Demo
`Non-Blocking Retry Pattern` 의 개념을 바탕으로 구현한 예제 애플리케이션은 아래와 같이 재시도 상황을 테스트 한다. 

구현된 데모의 상세한 코드는 [여기](https://github.com/windowforsun/kafka-consumer-non-blocking-retry-pattern)
를 통해 확인 가능하다.  

- `CreateEvent` 가 발생하지 않는 아이템에 대해 `UpdateEvent` 가 발생한다. 
- `UpdateEvent` 는 생성된 아이템이 없으므로 처리에 실패한다. 
- `UpdateEvent` 는 `CreateEvent` 가 수신되고 처리되기 전까지 재시도한다. 
- 이후 `CreateEvent` 가 수신되고 아이템이 생성된다. 
- `UpdateEvent` 도 성공한다. 

아래 그림은 위와 같은 상황을 도식화해 표현한 그림이다.  

![그림 1]({{site.baseurl}}/img/kafka/kafka-consumer-non-blocking-retry-pattern-3.drawio.png)

1. `Update Event Inbound Topic` 으로 부터 `UpdateEvent` 가 소비된다. 
2. `DB` 상 아직 `UpdateEvent` 수행에 해당하는 아이템이 생성되기 전이므로 처리작업은 실패한다. 
3. 실패한 `UpdateEvent` 이벤트 재시도를 위해 `RetryTopic` 으로 보내진다. 
4. `RetryTopic` 으로부터 `UpdateEvent` 가 소비된다. 
5. 수신한 재시도 이벤트가 재시도시간이 만료됐는지, 재시도를 수행할 시점인지, 이후 다시 폴링해야 하는지 검증한다. 재시도 수행 시점으로 판별된다면 이벤트는 다시 원본 토픽(`Update Event Inbound Topic`)에 쓰여진다. 
6. `Create Event Inbound Topic` 으로 부터 실패한 `UpdateEvent` 에 필요한 `CreateEvent` 가 소비된다. 
7. `CreateEvent` 에 해당하는 아이템이 `DB` 에 저장된다. 
8. `Update Event Inbound Topic` 으로 부터 재시도로 추가된 `UpdateEvent` 가 다시 소비된다. 
9. 이 시점에는 `UpdateEvent` 에 해당하는 아이템이 `DB` 에 존재하므로 이벤트는 성공적으로 처리된다. 

아래 코드는 `RetryTopic` 으로 부터 재시도 이벤트가 소비된 후 실제 재시도 여부를 판별하고 처리하는 코드의 일부이다.  

```java
    public void retry(final String payload, final MessageHeaders headers) {
        final Long verifiedOriginalReceivedTimestamp = (Long) headers.getOrDefault(RetryableHeaders.ORIGINAL_RECEIVED_TIMESTAMP, (Long) headers.get(KafkaHeaders.RECEIVED_TIMESTAMP));
        this.retryableKafkaClient.sendMessage(retryTopic, payload,
                Map.of(RetryableHeaders.ORIGINAL_RECEIVED_TIMESTAMP, verifiedOriginalReceivedTimestamp,
                        RetryableHeaders.ORIGINAL_RECEIVED_TOPIC, headers.get(KafkaHeaders.RECEIVED_TOPIC)));
    }

    public void handle(final String payload, final Long receivedTimestamp, final Long originalReceivedTimestamp, final String originalTopic) {
        if(this.shouldDiscard(originalReceivedTimestamp)) {
            log.debug("Item {} has exceeded total retry duration - item discarded.", payload);
        } else if(this.shouldRetry(receivedTimestamp)) {
            log.debug("Item {} is ready to retry - sending to update-item topic.", payload);
            this.retryableKafkaClient.sendMessage(originalTopic, payload, Map.of(RetryableHeaders.ORIGINAL_RECEIVED_TIMESTAMP, originalReceivedTimestamp));
        } else {
            log.debug("Item {} is not yet ready to retry on the update-item topic - delaying.", payload);
            throw new RetryableMessagingException("Delaying attempt to retry item " + payload);
        }
    }

    private boolean shouldDiscard(final Long originalReceivedTimestamp) {
        long cutOffTime = originalReceivedTimestamp + (this.maxRetryDurationSeconds * 1000);
        return Instant.now().toEpochMilli() > cutOffTime;
    }

    private boolean shouldRetry(final Long receivedTimestamp) {
        long timeForNextRetry = receivedTimestamp + (this.retryIntervalSeconds * 1000);
        log.debug("retryIntervalSeconds: {} - receivedTimestamp: {} - timeForNextRetry: {} - now: {} - (now > timeForNextRetry): {}", retryIntervalSeconds, receivedTimestamp, timeForNextRetry, Instant.now().toEpochMilli(), Instant.now().toEpochMilli() > timeForNextRetry);
        return Instant.now().toEpochMilli() > timeForNextRetry;
    }
```  

`RetryConsumer` 에서 소비된 재시도 이벤트는 `handle()` 를 호출한다. 
그리고 해당 메소드에서는 최초 이벤트 수신시간인 `ORIGINAL_RECIVED_TIMESTAMP` 를 통해 해당 이벤트가 재시도 처리 만료시간이 지났는지 판별해서, 
지났으면 해당 재시도 이벤트는 폐기된다. 
재시도 처리 만료시간이 지나지 않은 이벤트를 대상으로 해당 재시도 이벤트가 `RetryTopic` 에 전달된 시간값인 `KafkaHeaders.RECEIVED_TIMESTAMP` 를 통해 
재시도 처리 시간 주기를 만족하는지 보고 만족한다면, 
원본 토픽으로 이벤트를 다시 전송한다. 
그리고 재시도 처리 시간 주기를 만족하지 않는다면 예외와 함께 다시 폴링될 수 있도록 한다.  


```java
    public void updateItem(final UpdateItem event, final MessageHeaders headers) {
        final Optional<Item> item = this.itemRepository.findById(event.getId());

        if(item.isPresent()) {
            item.get().setStatus(event.getStatus());
            this.itemRepository.save(item.get());
            log.debug("Item updated in database with Id: {}", event.getId());
        } else {
            this.retryableService.retry(JsonMapper.writeToJson(event), headers);
            log.debug("Item sent to retry with Id: {}", event.getId());
        }
    }
```  

`Update Event Inbound Topic` 으로 소비된 이벤트는 `updateItem` 을 통해 `UpdateEvent` 처리가 수행된다. 
업데이트하고자 하는 아이템이 `DB` 에 존재하는지 검증 후 존재한다면 업데이트를 수행하고, 
존재하지 않는 경우 `retry()` 를 통해 처리 될 수 없는 이벤트를 `RetryTopic` 으로 전송한다.  


이번 포스팅에서 소개한 `Non-Blocking Retry Pattern` 은 일반적인 구현법을 소개한 내용이다. 
데모 또한 `Java + Spring` 기반을 통해 구현했지만, 동일한 환경이라면 특별한 기능이 필요하지 않은 한
[Spring Kafka Retry Topics]({{site.baseurl}}{% link _posts/kafka/2024-04-15-kafka-practice-kafka-consumer-non-block-retry-with-spring.md %})
사용하는 것을 권장한다. 
재시도 구현을 위한 코드 작업이 적고 단순하다는 점과 다양한 설정으로 원하는 재시도 스펙을 적용할 수 있기 때문이다.  
`Spring Kafka Retry Topics` 또한 `Non-Blocking Retry` 과정에서 원본 토픽의 메시지 순서는 보장되지 않는 다는 점을 기억해야 한다.  








---  
## Reference
[Kafka Consumer Non-Blocking Retry Pattern](https://www.lydtechconsulting.com/blog-kafka-non-blocking-retry.html)   
