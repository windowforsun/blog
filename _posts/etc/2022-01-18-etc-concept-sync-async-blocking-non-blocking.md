--- 
layout: single
classes: wide
title: "Synchronous/Asychronous, Blcking/Non-Blcking"
header:
  overlay_image: /img/blog-bg.jpg
excerpt: 'Sync/Async 와 Blocking/Non-Blocking 의 각 개념과 차이 및 조합 등에 대해 알아보자 '
author: "window_for_sun"
header-style: text
categories :
  - ETC
tags:
  - ETC
  - Concept
  - Synchronous
  - Asynchronous
  - Blocking
  - Non-Blocking
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
`Synchronous` 는 `Main` 이 호출하는  `Sub` 의 작업 완료에 대한 리턴을 기다리거나, `Sub` 의 리턴은 바로 받더라도 이후에 작업 완료 여부를 계속해서 확인하는 동작을 의미한다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-1.drawio.png)  

위 그림은 `Synchronous thread` 에 대한 그림으로 `Main thread` 가 `Sub thread` 생성 후, 
`Sub thread` 가 종료될 떄까지 대기하고 `Sub thread` 가 종료가 된 후에야 `Main thread` 가 작업을 수행이 가능하다는 그림이다.  

이는 `Thread` 의 관계에서만 적용되는 것이아니라 함수가 다른 함수를 호출하는 관계에 대입해도 동일한 설명과 개념이 될 수 있다.  

앞서 설명한 개념중 `Sub` 호출 후 `Main` 에서 완료 여부를 계속해서 확인하는 경우 또한 `Main` 에서 확인 하는 과정 중간 중간 다른 작업은 수행 할 수는 있지만, 
이 또한 `Sync` 동작임을 기억해야 한다.  


### Asynchronous
`Asynchronous` 는 `Main` 이 호출하는 `Sub` 에 `Callback`(주로 사용하는 방법)을 전달해서, 
`Main` 이 `Sub` 의 리턴값 혹은 작업 완료 여부를 기다리거나 확인 하는 동작 없이 `Sub` 이 완료되면 `Callback` 을 수행해서 완료 동작을 수행하는 것을 의미한다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-2.drawio.png)  

위 그림은 `Asynchronous thread` 에 대한 그림으로 `Main thread` 가 `Sub thread` 생성 후, 
`Main`, `Sub` 모두 동시에 작업 수행이 가능하다는 그림이다.  

이 또한 `thread` 가 아닌 함수의 관게에서 보더라도 동일한 설명과 개념을 의미한다.  


## Blocking/Non-Blocking
`Blocking/Non-Blocking` 은 제어권의 관점에서 볼 수 있다. (다른 말로 하자만 리턴을 바로 하느냐 마느냐)
즉 `Sync/Async` 와 서로 관점이 다르기 때문에 서로 비슷한 개념을 언급하는 것 같지만 서로가 다른 부분에 관심을 두고 있다는 의미이다.  
주로 `Blocking/Non-Blocking` 은 직접적으로 제어할수 없는 대상을 어떤 식으로 처리하느냐에 따라 나뉜다고 할 수 있다. 
앞서 제어권을 언급한 것도 직접적으로 제어할 수 없는 대상을 처리할때 제어권을 넘겨 주는지, 제어권을 넘겨주지 않고 온전히 가지고 있는지에 따라 나뉜다고 할 수 있다. 
여기서 직접적으로 제어할 수 없는 대상의 대표적인 예로는 `IO`, `멀티쓰레드 동기화` 등이 있다.  

### Blocking
`Blocking` 은 `Main` 이 호출하는 `Sub` 에 제어권을 넘겨줘서 `Sub` 이 작업을 수행하고, 
`Sub` 이 완료 될때까지 제어권을 돌려주지 않기 때문에 `Main` 은 계속 대기하게 되는 동작을 의미한다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-3.drawio.png)  

위 그림을 보면 `Synchronous` 와 거의 유사하지만 `제어권` 에 대한 설명만 포함된 그림이라고 할 수 있다. 
`Main` 이 제어권을 `Sub` 으로 넘겨준 순감 부터 `Sub` 이 주체적으로 작업을 수행하게 되고 `Main` 은 `Sub` 이 다시 제어권을 돌려주길 만을 기다리고 있는 것이다. 
`Sub` 의 작업이 완료되고 제어권을 돌려 받으면 그때서야 `Main` 은 자신의 작업을 이어서 할 수 있다.  

> `Synchronous` 는 작업의 완료 여부를 기다린 것이지, 제어권을 기다린 것이 아니다. 

### Non-Blocking
`Non-Blocking` 은 `Main` 이 호출하는 `Sub` 이 호출과 동시에 바로 리턴해서 `Main` 에 바로 제어권을 넘겨 준다. 
그러므로 `Main` 과 `Sub` 은 동시에 각자 작업을 수행할 수 있는 것을 의미한다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-4.drawio.png)  

`Non-Blocking` 그림도 `Asynchronous` 와 거의 유사하지만 `제어권` 에 대한 설명만 포함된 그림이라고 할 수 있다. 
`Main` 이 제어권을 `Sub` 에 넘겨주면 `Sub` 은 바로 리턴을 수행하며 제어권을 다시 `Main` 에서 넘겨주게 되면서, 
`Main` 또한 작업을 수행할 수 있는 기회를 얻는 것이다.  


## 중간 정리
중간 개념 정리와 추가 적인 설명을 더하고 다음 설명을 진행하려고 한다. 
`Multithreading` 모델을 보면 대부분 중간에 `Waiting Queue` 를 가지고 있는 경우가 있다. 
`Waiting Queue` 가 존재하는 작업의 경우 `Wating Queue` 에 추가된 작업은 대기 상태에 들어가기 때문에 `Blcoking` 이라고 볼 수 있다. 
즉 반대로 말하면 `Waiting Queue` 에 들어가지 않는 경우를 `Non-Blocking` 이라고 할 수 있다.  

그리고 `Asychronous` 와 `Non-Blocking` 는 모두 즉시 리턴한다는 보장이 있는 동작이다. 
이 둘의 차이는 결과값을 함께 리턴 하느냐 마느냐로 나뉠 수 있다. 
`Asynchronous` 는 `Callback` 이 결과값 처리를 하기 때문에 당연히 결과값이 리턴떄 함께 올 수 없지만, 
`Non-Blocking` 은 결과값과 함께 즉시 리턴되는 동작을 의미한다.  

마지막으로 `Sychronous` 와 `Blocking` 모두 호출한 결과 및 리턴을 기다린다는 공통점을 가지고 있다. 
이를 `Waiting Queue` 의 관점에서 본다면 `Synchronous` 는 `Waiting Queue` 가 필수가 아니고, `Blocking` 은 `Waiting Queue` 가 필수적으로 필요하다.  

위 내용을 다시 한번 표로 정리하면 아래와 같다.  


구분|Synchronous|Blocking|Asynchronous|Non-Blocking
---|---|---|---|---
시스템 콜의 완료를 기다리는지|O|O|X|X
값과 함께 리턴하는지|O|O|X|O
즉시 리턴 하는지|X|X|O|O
대기큐에서 기다리는지|X|O|X|X


## 다양한 조합과 종류

### Sync-Blocking
가장 일반적이면서 자주 사용되는 흐름으로 단순한 동작이다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-3.drawio.png)  

`Main` 은 `Sub` 의 리턴값을 필요로 하기 때문에 `Blocking` 이라고 할 수 있다. 
그러므로 제어권을 `Sub` 으로 넘겨주고 `Sub` 이 완료 될 때까지 기다려야 하므로 `Sync` 이다. (제어권을 넘겨줄 떄까지)  

일반적으로 함수가 다른 함수를 호출하는 경우를 생각하면 쉽다.  

### Sync-NonBlocking
`Sync` 를 `Non-blcoking` 처럼 동작 시킬 수도 있는데, 
호출 되는 함수는 바로 리턴하지만 호출하는 함수는 호출된 함수의 완료 여부를 계속해서 확인하는 경우가 될 수 있다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-5.drawio.png)  

`Main` 은 `Sub` 의 실행과 동시에 리턴을 받으면서 제어권도 돌려 받는다. (`Non-Blocking`)
하지만 `Main` 은 계속해서 `Sub` 에게 완료 여부를 묻는 동작을 수행한다. (`Sync`) 
물론 `Sub` 의 완료 여부를 검사하며 다른 동작도 수행은 가능하다.  

이와 관련되 간단한 예시로는 아래 코드처럼 `Future` 를 `Non-Blocking` 하게 실행했지만, 
`while` 문을 통해 종료 여부를 검사하는 것과 같다.  

```java
Future future = Executors.newSingleThreadExecutor().submit(() -> sub());

while(!future.isDone()) {
    // do something..
}
```  

### Async-NonBlocking
`Async-NonBlocking` 은 호출 된 함수에게 바로 제어권을 받고, 
호출 된 함수의 결과 처리는 `Callback` 으로 처리하기 때문에 
성능과 자원의 효율적 사용관점에서 가장 유리한 모델인  이다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-6.drawio.png)  

`Main` 은 `Sub` 의 실행과 동시에 제어권 및 리턴을 바로 받는다. (`Non-Blocking`)
그러므로 `Main` 또한 계속해서 다른 동작을 수행할 수 있다. 
그리고 `Sub` 을 호출 할때 `Callback` 을 함께 넘겼고, 
`Sub` 은 완료되면 전달받은 `Callback` 을 호출해서 결과값 처리를 하게 될것이다. (`Async`)

최근 들어 많은 프로그래밍 또는 서버 애플리케이션에서 사용하는 모델이라고 할 수 있다. 
`Node.js`, `Java Reactor`, `Spring Webflux`, `Netty` 등  더 많음 위 모델과 관련된 기술과 라이브러리, 프레임워크, 서버가 존재한다.  


### Async-Blocking
가장 마지막은 `Async` 이지만 `Blocking` 이여서 제어권을 받지 못해 대기 하는 경우인 `Async-Blocking` 이다.  

![그림 1]({{site.baseurl}}/img/etc/concept-sync-async-blocking-non-blocking-7.drawio.png)  

`Main` 은 `Sub` 의 완료와 리턴값을 대기하지 않기 위해 `Callback` 을 함께 전달한다. (`Async`)
하지만 `Main` 은 `Sub` 을 실행할 때 제어권을 넘겨주고 돌려 받지 못해 제어권을 받을 때까지(`Sub` 완료) 계속 대기하게 된다. (`Blocking`) 

이 모델은 굉장히 아이러니 한 모델이라고 할 수 있다. 
의도적으로는 마주하기 쉽지 않다고 말할 수 있지만, 
`Non-Blocking`(`Reactive`) 모델을 사용하다 보면 한번 쯤은 마주 해본 모델일 수도 있다. 
바로 `Async-NonBlocking` 과정에서 하나의 `Blocking` 이라도 포함돼 있다면, 
이는 `Async-Blocking` 이 되버리기 때문이다. 
`Webflux + JDBC` 을 사용하는 경우라던가, `Node.js + MySQL` 등이 있을 수 있다.  



---

## Reference
[Blocking-NonBlocking-Synchronous-Asynchronous](https://homoefficio.github.io/2017/02/19/Blocking-NonBlocking-Synchronous-Asynchronous/)  
[Sync async-blocking-nonblocking-io](https://www.slideshare.net/unitimes/sync-asyncblockingnonblockingio)  
[asynchronous and non-blocking calls? also between blocking and synchronous](https://stackoverflow.com/questions/2625493/asynchronous-and-non-blocking-calls-also-between-blocking-and-synchronous)  
[Boost application performance using asynchronous I/O](https://developer.ibm.com/articles/l-async/)  








