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
toc: true 
use_math: true
---  

## Reactor Backpressure
`Reactive Stream` 에서 `Backpressure` 란 `Publisher` 와 `Subscriber` 사이에서 서로 과부하기 걸리지 않도록, 
제어하는 것을 의미한다. 
이는 `Reactive Stream` 에서는 매오 중요한 특성으로, 
`Publisher` 의 데이터를 받아 처리하는 `Subscriber` 간의 적절한 속도를 맞추는 것으로 
시스템 간의 `Overloading` 이 발생하는 것을 방지한다.  

`Backpressure` 는 `Reactive Stream` 에서만 존재하는 개념은 아니다. 
어떻게 보면 유체역학에서 유량제어에 관련된 [Backpressure](https://en.wikipedia.org/wiki/Back_pressure)
를 소프트웨어 데이터 흐름제어 관점으로 차용한 것이라고 할 수 있다.  

말그대로 `Backpressure` 는 `Publisher` 와 `Subscriber` 사이에서 오고가는 데이터의 양을 컨트롤 하는 것인데, 
아래와 같은 일반적인 특성을 기억해야 한다. 

- `Publisher` 에서 방출한 데이터는 `Common Buffer` 에 추가된다. 
- `Subscriber` 는 `Common Buffer` 에서 데이터를 읽어온다. 
- `Common Buffer` 의 크기는 제한돼 있다. 


> `Backpressure` 의 동작은 `Multi-thread` 환경에서 비로서 의미가 있음을 기억해야 한다. 


