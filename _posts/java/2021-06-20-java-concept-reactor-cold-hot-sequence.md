--- 
layout: single
classes: wide
title: "[Java 개념] "
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

## Hot / Cold Sequence
`Reactor` 에서는 시퀀스를 제공하는 `Mono`, `Flux` 의 2가지 타입을 제공한다. 
위 두가지 타입의 시퀀스를 생성할때 `Hot`, `Cold` 의 두가지 `Sequence`(`Publisher`)로 생성 방식을 나눌 수 있다.  
두가지 방식을 개념적으로 먼저 설명하면 아래와 같다. 
- `Hot Sequence` : 실제 구독(`Subscribe`) 전 시퀀스의 데이터 생성 및 동작이 수행되고, 한번 생성된 데이터는 구독시 계속해서 사용된다. 
  - 여러 구독자가 하나의 시퀀스의 데이터를 함께 구독해야하는 경우 `Hot Sequence` 방식으로 생성하는 것이 좋다. 
- `Cold Sequence` : 실제 구독(`Subscribe`) 전까지 시퀀스는 아무 동작을 수행하지 않고, 구독이 이뤄질 때 그때마다 시퀀스의 데이터생성 및 동작이 수행된다. 
  - 시퀀스가 매번 구독 때마다 가장 최신 혹은 초기화 과정까지 필요하다면 `Cold Sequence` 방식으로 생성하는 것이 좋다. 

우리가 흔히 알고 있는 `Reactive Streams` 의 동작방식은 `Cold Sequence`(?)


### Cold Sequence

### Hot Sequence

### Cold to Hot


---
## Reference
[Hot vs Cold](https://projectreactor.io/docs/core/snapshot/reference/#reactive.hotCold)  
[Hot Versus Cold](https://projectreactor.io/docs/core/snapshot/reference/#reactor.hotCold)  
[Reactor Hot Publisher vs Cold Publisher](https://www.vinsguru.com/reactor-hot-publisher-vs-cold-publisher/)  

