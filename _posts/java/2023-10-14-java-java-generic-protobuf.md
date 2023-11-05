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
