--- 
layout: single
classes: wide
title: "Sync/Async, Blcking/Non-Blcking"
header:
  overlay_image: /img/blog-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - ETC
tags:
  - ETC
  - Concept
  - Synchronous
  - Asynchronous
  - Blcking
  - Non-Blcking
toc: true
use_math: true
---  

## 용어차이
이번 포스트에서는 `Synchronous/Asynchronous`(동기/비동기) 와 `Blocking/Non-Blocking`(블로킹/논블로킹) 의 용어가 의미하는 개념에 대해서 알아본다. 
최근 `MSA` 가 각광받고 대부분의 서비스의 구성이 발맞춰 따라가는 만큼 비동기와 논블로킹에 관련 기술과 언급은 계속해서 많아지고 있다. 
문론 위 개념은 컴퓨터공학에서 모두가 배우는 운영체제 과목에서부터 계속 언급되고 배워온 용어이기도 하다. 
하지만 이를 혼용해서 사용하거나 바꿔서 잘못 사용하는 경우를 심심치 않게 많이 볼 수 있다. (필자 포함)
막상 위 4가지를 나열하고 정의해 보라고 하면 조금씩 헷갈리지만 찾아 개념을 살펴보면 또 당연한 듯이 넘어가기도 한다. 
지금부터 정리하는 내용은 백과사전과 같은 바이블을 작성하겠다는 것은 아니지만, 나와 혹은 누군가의 개념정리를 돕는 포스팅이라고 생각하고 시작한다.  

먼저 아주 간단하게 정리하면 아래와 같다.  
- `Sync/Async` 와 `Blocking/Non-Blocking` 의 관계는 `==` 의 관계가 아니다. 서로의 관심사가 다르기 때문이다. 
- `Sync/Async` 는 호출된 함수의 작업 완료 여부를 신경 쓰느냐 마느냐로 나뉠 수 있다. 
  - `Sync` : 호출된 함수의 작업 완료 여부를 신경쓴다면
  - `Async` : 호출된 함수의 작업 완료 여부를 직접 적으로 신경쓰지 않고, 작업 완료에 대한 `Callback` 등을 사용
- `Blocking/Non-Blocking` 은 호출된 함수가 바로 리턴 하느냐 마느냐로(제어권을 바로 주느냐 마느냐) 나뉠 수 있다. 
  - `Blocking` : 호출된 함수가 바로 리턴하지 않고 제어권을 완료될 떄까지 넘겨주지 않음
  - `Non-Blocking` : 호출된 함수가 바로 리턴해서 제어권을 넘겨줌
	


## Synchronous/Asychronous
`Synchronous` 에서 `Syn` 단어의 사전적 의미를 찾아보면 `모두(together), 같이(with)` 와 같은 뜻을 가지고 있다. 
그리고 `chrono` 는 사전적인 의미로 `때(time)` 라는 뜻을 가지고 있다. 
이를 합쳐서 의미를 파악하면 모두 같이 시간을 맞춘 다는 의미로 해석할 수 있다. 
`Asynchronous` 는 `A` 라는 접두사가 붙으면서 부정하는 의미로 해석 할 수 있기 때문에, 모두가 같이 시간을 맞추지 않는 다는 의미로 해석할 수 있다.  

우리가 주로 사용하는 `Sync/Async` 에서 시간이라는 것은 실제로 `Task` 를 수행하는 `CPU` 의 입장에서 바라보는 시간이라고 할 수 있다. 
즉 `Sync` 는 `CPU` 시간을 모두 함께 적절히 맞춰서 사용하게 될 것이고, 
`Async` 는 `CPU` 시간을 적절히 맞추지 않고 모두 함께 한번에 사용하게 되는 경우도 발생한다고 생각하면 쉽다.  

### Synchronous
`Synchronous` 는 `Main` 이 호출하는 `Task` 인 `Sub` 의 작업 완료에 대한 리턴을 기다리거나, `Sub` 의 리턴은 바로 받더라도 이후에 작업 완료 여부를 계속해서 확인하는 동작을 의미한다.  







### Asynchronous
`Asynchronous` 는 `Main` 이 호출하는 `Task` 인 `Sub` 에 `Callback`(주로 사용하는 방법)을 전달해서, 
`Main` 이 `Sub` 의 리턴값 혹은 작업 완료 여부를 기다리거나 확인 하는 동작 없이 `Sub` 이 완료되면 `Callback` 을 수행해서 완료 동작을 수행하는 것을 의미한다.  





## Blocking/Non-Blocking

## 다양한 조합과 종류

이후 설명에서는 주로 2개의 `Task` 인 `Main`, `Sub` 을 사용해서 설명을 진행한다.
`Main` 에서 새롭게 생성되고 실행되는 `Task` 가 `Sub` 이라고 생각하면 된다.

---

## Reference
[Blocking-NonBlocking-Synchronous-Asynchronous](https://homoefficio.github.io/2017/02/19/Blocking-NonBlocking-Synchronous-Asynchronous/)  
[Sync async-blocking-nonblocking-io](https://www.slideshare.net/unitimes/sync-asyncblockingnonblockingio)  
[asynchronous and non-blocking calls? also between blocking and synchronous](https://stackoverflow.com/questions/2625493/asynchronous-and-non-blocking-calls-also-between-blocking-and-synchronous)  
[Boost application performance using asynchronous I/O](https://developer.ibm.com/articles/l-async/)  








