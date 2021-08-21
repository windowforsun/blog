--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Splitting Operator"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 제공하는 Operator 중 시퀀스를 분할하는 Operator 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Operator
  - Splitting a Flux
toc: true 
use_math: true
---

## Reactor Operators
[Reactor Operator 전체 리스트](https://windowforsun.github.io/blog/java/java-concept-reactor-operator)

## Splitting a Flux(`Flux` 나누기)

구분|메소드|타입|설명
---|---|---|---
`Flux<T>`를 `Flux<Flux<T>>` 로 분리|window|Flux|방출되는 요소 크기 혹은 시간을 기준으로 분리
 |windowTimeout|Flux|방출되는 요소 크기와 시간을 기준으로 분리
 |windowUntil|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소 다음 부터 분리
 |windowWhile|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소는 제외하고 다음 부터 분리
 |windowWhen|Flux|`Publisher` 의 신호의 경계 값을 기준으로 분리
`Flux<T>`를 `Flux<List<T>` 로 분리|buffer|Flux|방출되는 요소 크기 혹은 시간을 기준으로 분리
 |bufferTimeout|Flux|방출되는 요소 크기와 시간을 기준으로 분리
 |bufferUntil|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소 다음 부터 분리
 |bufferWhile|Flux|`Predicate` 의 `Boolean` 값이 `true` 인 요소는 제외하고 다음 부터 분리
 |bufferWhen|Flux|`Publisher` 의 신호의 경계 값을 기준으로 분리
`Flux<T>`를 `Flux<GroupedFlux<K, T>>` 로 같은 특정을 가진 것들로 분리|groupBy|Flux|`Function` 로직에서 같은 특정을 갖는 요소들 끼리 그룹지어 분리


---
## Reference
[Appendix A: Which operator do I need?](https://projectreactor.io/docs/core/release/reference/#which-operator)  


