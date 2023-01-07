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
    - Processor API
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
public interface Processor<K, V> {

    void init(ProcessorContext context);

    void process(K key, V value);

    void close();
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






---  
## Reference
[PROCESSOR API](https://kafka.apache.org/21/documentation/streams/developer-guide/processor-api.html)  
