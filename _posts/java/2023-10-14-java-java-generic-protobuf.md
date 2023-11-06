--- 
layout: single
classes: wide
title: "[Java 실습] "
header:
  overlay_image: /img/java-bg.jpg 
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactive Stream
  - Reactor
  - Backpressure
  - Flux
toc: true 
use_math: true
---  

## Java Generic to Protobuf
`Protobuf` 는 직렬화 대상이 되는 데이터 크기를 많이 줄 일 수 있고, 
직렬화 성능 또한 좋다. 
하지만 이런 이점들을 얻기 위해서는 직렬화 대상이 되는 데이터에 대해 스키마를 별도로 지정해야 한다. 
또한 스키마에서 데이터 필드는 명시적인 데이터 타입을 가져야 한다는 점이 있다.  

위와 같은 이유로 `Protobuf` 에서 런타임에 타입이 결정되는 `Java Generic` 에 대해서는 공식적으로 
지원하는 기능은 없다. 
하지만 `Protobuf` 에서 제공하는 몇가지 기능을 함께 사용하는 트릭을 사용하면, 
어느정도 `Java Generic` 을 `Protobuf` 스키마로 표현해서 사용 할 수 있다.  

### Oneof
[Oneof](https://protobuf.dev/programming-guides/proto3/#oneof)
는 `Protobuf` 메시지 특정 필드에 여러 필드나 타입이 올 수 있을 때, 
`oneof` 집합 중 하나의 필드를 사용 할 수 있도록 해준다.  


```protobuf
message Sample {
  oneof test_oneof {
    string str = 1;
    int32 num = 2;
    OtherMessage other_message = 3;
  }
}
```  

위 예시처럼 `test_oneof` 라는 `oneof` 집합을 구성하는 3개의 필드중 하나의 필드만 선택적으로 사용 될 수 있고, 
사용되지 않은 나머지 필드는 모두 지워진다.  

### Wrapper Message
`Wrapper Message` 란 표현 그대로 `Protobuf` 의 `Message` 를 동적 타입 표현을 목적으로 
한번 더 `Message` 로 감싼 형태를 의미한다. 
그 예시는 아래와 같다.  

```protobuf
message MyMessage {
  SampleWrapper sample = 1;
}

message SampleWrapper {
  oneof sample_oneof {
    SampleA sample_a = 1;
    SampleB sample_b = 2;
  }
}
```  

`MyMessage` 메시지 스키마를 기준으로 `sample` 이름을 가진 필드는 
필요에 따라 런타임에 `SampleA`, `SampleB` 2가지 타입이 올 수 있는 필드로 사용 할 수 있다.  
