--- 
layout: single
classes: wide
title: "[Java 개념] Reactor Schedulers 와 PublishOn, SubscribeOn"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Reactor 에서 Schedulers 의 종류와 PublishOn, SubscribeOn 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Reactor
  - Schedulers
  - PublishOn
  - SubscribeOn
toc: true 
use_math: true
---  

## Schedulers
`Reactor` 에서는 기본적으로 [Schedulers](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html)
라는 것을 통해 비동기 스크림 처리를 지원한다. 
사용자는 `Schedulers` 를 사용해서 작업 성격에 맞는 비동기 및 `Non-Blocking` 작업을 수행할 수 있다. 
`Schedulers` 에서는 팩토리 메소드를 사용해서 성격에 따라 사용할 수 있는 스레드 모델을 제공한다. 

Schedulers Method|Desc
---|---
parallel()|
single()|
elastic()|
boundedElastic()|
immediate()|
fromExecutorService(ExecutorService)|

new ** 를 통해 커스텀 생성 가능


---
## Reference
[Reactor Schedulers](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html)  
[Reactor Schedulers – PublishOn vs SubscribeOn](https://www.vinsguru.com/reactor-schedulers-publishon-vs-subscribeon/)  


