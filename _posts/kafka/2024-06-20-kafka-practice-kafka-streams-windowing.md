--- 
layout: single
classes: wide
title: "[Kafka] Kafka Streams Windowing"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Streams 에서 특정 범위의 데이터를 모아 처리하는 Window 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
    - Window
    - Windowing
    - Tumbling Window
    - Hopping Window
    - Sliding Window
    - Session Window
toc: true
use_math: true
---  

## Kafka Streams Windowing
`Window` 란 정해진 시간 범위 안에 있는 데이터를 의미한다. 
윈도우에 대한 시간 범위, 유형 등과 같은 특성은 필요에 따라 정의할 수 있다. 
그리고 윈도우 처리에 대상이 되는 데이터는 시간 범위에 해당하는 데이터라는 점을 기억해야 한다. 
아래는 이러한 시간과 이벤트 그리고 윈도우의 관계를 도식화 해서 보여주는 그림이다.  

![kafka-stream-windowing-1.drawio.png](..%2F..%2Fimg%2Fkafka%2Fkafka-stream-windowing-1.drawio.png)

`t0 ~ t60` 까지 시간이 있을 때, `A-E` 이벤트가 발생했다. 
여기서 윈도우범위가 `t10 ~ t40` 이라면 해당 윈도우에 포함되는 이벤트는 `B, C, D, E` 가 된다.  
즉 윈도우 기반 처리를 한다고 했을 때 해당 윈도우에서 처리하는 데이터는 4개인 것이다.  

그리고 윈도우 유형을 어떤 것을 선택하냐에 따라 
하나의 이벤트가 단 한개의 윈도우에만 포함될 수 있고, 그 이상의 윈도우에 포함 될 수도 있다.  

`Kafka Streams` 는 아래와 같은 윈도우 유형을 제공한다. 
각 유형은 아래에서 자세한 설명을 이어가겠지만 추가적으로 [여기](https://www.confluent.io/ko-kr/blog/windowing-in-kafka-streams/)
에서도 확인 할 수 있다.  

윈도우 범위는 시작시간은 포함하고 종료시간은 포함하지 않는다. 
즉 만약 윈도우 범위가 `w1[t10 ~ t20)` 이라면 이는 `t10 <= time range < t20` 인 것이다.  

이름| 동작            |설명
---|---------------|---
Hoping| Time-based    |- 고정된 윈도우 크기<br>- 윈도우간 겹침
Tumbling| Time-based    |- 고정된 윈도우 크기<br>- 윈도우간 겹치지 않음
Sliding| Time-based    |- 고정된 윈도우 크기<br>- 이벤트 시점에 따라 윈도우가 겹치거나 안 겹침<br>
Session| Session-based |- 동적인 윈도우 크기<br>- 윈도우간 겹치지 않음<br>- 이벤트 기반 윈도우


### Tumbling Window
`Tumbling Window` 는 윈도우간 겹침 없이 고정된 윈도우 크기와 연속적인 순서를 갖으며, 
윈도우를 대표하는 특성을 지닌 윈도우이다. 

![kafka-stream-windowing-2.drawio.png](..%2F..%2Fimg%2Fkafka%2Fkafka-stream-windowing-2.drawio.png)

시간 프레임에서 윈도우간 겹침이 없기 때문에, 이벤트는 1개의 윈도우에만 포함된다. 
위 그림은 `Window size` 가 `20t` 일 떄, 
5개의 이벤트가 발생 했을 때 생성되는 윈도우와 각 윈도우가 포함하는 이벤트를 정리하면 아래와 같다.  

- `w1[t0 ~ t20)` - A(t8), B(t12)
- `w3[t20 ~ t40)` - C(t20), D(t25), E(t34)

`Tumbing Window` 는 `t0` 부터 항상 시작하고 시작 포함하지만 종료시간은 포함하지 않는 다는 점을 기억해야 한다. 
`t20` 에 발생한 `C` 이벤트는 시간 범위가 20일 때 `w1` 에 포함되지 않고, 
다음 윈도우인 `w2` 에 포함된 것이 바로 그 이유다. 


### Hoping Window
`Hoping Window` 는 `Tumbling` 처럼 고정된 윈도우 크기를 가지지만, 
이전 윈도우가 닫히기 전에 새로운 윈도우가 생성된다. 
그러므로 하나의 이벤트는 여러 윈도우에 포함 될 수 있다.  

`Tumbling` 과 마찬가지로 시작시간은 포함하지만, 종료시간은 포함하지 않는다. 
그리고 `Hoping Window` 의 경우 추가적으로 `Advance period` 를 설정해야 한다. 
여기서 `Advance period` 란 현재 윈도우 시작 부터 다음 윈도우 생성까지 대기하는 시간을 의미한다. (윈도우 생성 주기)
`Window size` 와 `Advance period` 를 설정할 때 `Window size` 보다 큰 `Advance period`
와 같은 이벤트 누락이 발생 할 수 있는 설정은 `API` 적으로 막아주고 있다.  

![kafka-stream-windowing-3.drawio.png](..%2F..%2Fimg%2Fkafka%2Fkafka-stream-windowing-3.drawio.png)

위 그림은 `Window size` 는 `20t` 이고, `Advance period` 는 `10t` 로 설정된 예시이다. 
`t8` 에 `A` 이벤트, `t12` 에 `B` 이벤트, `t18` 에 `C` 이벤트, `t25` 에 `D` 이벤트, `t34` 에 `E` 이벤트로 총 5개의 이벤트가 발생한다. 
윈도우와 포함되는 이벤트를 나열하면 아래와 같다.  

- `w1[t0 ~ t20)` - A(t8), B(t12), C(t18)
- `w2[t10 ~ t30)` - B(t12), C(t18), D(t25)
- `w3[t20 ~ t40)` - D(t25), E(t34)
- `w4[t30 ~ t50)` - E(t34)

`B`, `C`, `D`, `E` 이벤트가 여러 윈도우에 포함된 것과 같이, 
`Hopping Window` 는 `Tumbling` 과는 다르게 이벤트가 여러 윈도우에 포함될 수 있다.   



### Sliding Window
`Sliding Window` 는 앞서 살펴본 윈도우들과 많이 부분에 차이가 있다. 
고정된 윈도우 크기를 가지지만 정해진 윈도우 간격은 가지지 않고, 
이벤트의 시점에 따라 윈도우가 발생하므로 윈도우가 상황에 따라 겹칠 수도 겹치지 않을 수도 있다. 
또한 윈도우간 시간 간격이 겹칠 수는 있지만, 
윈도우에 포함된 이벤트 구성이 겹치지는 않는다. 
예를들어 `w1` 에 `e1` 이 있고, `w2` 에 `e1, e2`, `w3` 에 `e2` 가 포함되는 상태라고 하자.
각 이벤트는 여러 윈도우에 포함되지만 각 윈도우를 구성하는 이벤트의 집합은 겹치지 않고 유니크한 구성을 포함하고 있다는 의미이다. 
즉 윈도우에 포함되는 이벤트의 구성이 달라지는 경우에만 실제 윈도우 연산이 수행된다.  

그리고 또 한가지 다른 특징은 다른 유형들과 다르게 윈도우 생성이 이벤트 발생 시점을 기준으로 한다는 점이다. 
즉 설정된 윈도우 크기(`Window size`)와 슬라이딩 간격(`Advance`)으로 윈도우가 생성될 수 있는 상태에서, 
이벤트가 발생하면 해당 이벤트가 처음으로 포함되는 윈도우, 처음으로 제외되는 이벤트를 생성하게 된다.  


이러한 `Sliding Window` 는 좀 더 복잡함을 요구하지만, 
이벤트 발생에 따라 윈도우가 생성되기 때문에 적절한 상황에 잘 사용하면 
다른 방식보다 더 효율성을 기대 할 수 있다.  

![kafka-stream-windowing-4.drawio.png](..%2F..%2Fimg%2Fkafka%2Fkafka-stream-windowing-4.drawio.png)

위 그림은 `Window size` 가 `30t` 인 상황에서 예시를 보여주고 있다. 
A 이벤트는 `t29`, B 이벤트는 `t40`, C 이벤트는 `t67` 정도에 발생한다고 해보자. 
그러면 실질적으로 처리되는 윈도우와 각 윈도우에 포함되는 이벤트는 아래와 같다. 

- `w1[t0 ~ t30)` - A(t29) : A 이벤트 발생으로 윈도우 생성
- `w2[t11 ~ t41)` - A(t29), B(t40) : B 이벤트 발생으로 윈도우 생성
- `w3[t31 ~ t61)` - B(t40) : A 이벤트 제거로 윈도우 생성
- `w4[t38 ~ t68)` - B(t40), C(t67) : C 이벤트 발생으로 윈도우 생성
- `w5[t41 ~ t71)` - C(t67) : B 이벤트 제거로 윈도우 생성

`Sliding Window` 를 이해할 때 중요한 점은 윈도우는 크기 `t30`과 슬라이딩 간격 `t10` 으로 계속해서 이동한다. 
하지만 실질적으로 처리되는 윈도우는 이벤트가 처음으로 윈도우에 포함되고 제거되는 시점의 윈도우임을 기억해야 한다.  


### Session Window
`Session Window` 는 앞서 살펴본 `Sliding Window` 보다 좀 더 이벤트을 기반으로 하는 윈도우라고 할 수 있다. 
이벤트 간의 활동 시간과 무활동 시간을 기반으로 `Window` 가 생성되고, 
연속적인 이벤트 활동을 하나의 세션으로 간주하여 그룹화하기 때문에 동적으로 윈도우 크기가 결정된다고 할 수 있다.  

윈도우를 생성할 때 `inactivityGap` 이라는 무활돈 시간을 설정하게 되는데, 
첫 이벤트가 발생된 이후 발생하는 이벤트들이 직전 발생한 이벤트 시간을 기준으로 `inactivityGap` 시간 안에만 발생하면 
윈도우는 계속해서 유지되기 때문에 윈도우 크기는 동적이게 된다. 
그리고 `inactivtyGap` 시간 동안이상 아무런 이벤트가 없으면 이전 윈도우는 닫히게 되고, 
이후 발생하는 이벤트 부터는 새로운 윈도우에 포함되게 된다.  

![kafka-stream-windowing-5.drawio.png](..%2F..%2Fimg%2Fkafka%2Fkafka-stream-windowing-5.drawio.png)

위 그림은 무활동 시간이 `5t` 일때 이벤트 발생에 따른 윈도우 처리르 보여준다. 
A 이벤트는 `t10`, B 이벤트는 `t14`, C 이벤트는 `t21`, D 이벤트는 `t30` 에 발생한다고 하면 
생성되는 윈도우와 포함하는 이벤트는 아래와 같다. 

- `w1[t10 ~ t19)` - A(t10), B(t14) : A이벤트 발생 시점 부터 B이벤트 발생 후 무활동 시간이 포함된 시간까지 윈도우가 생성된다. 
- `w2[t21 ~ t26)` - C(t21) : C이벤트 발생 시점 부터 무활동 시간이 포함된 시간까지 윈도우가 생성된다. 
- `w3[t30 ~ t35)` - D(t30) : D이벤트 발생 시점 부터 무활동 시간이 포함된 시간까지 윈도우가 생성된다. 



---  
## Reference
[Kafka Streams Windowing - Overview](https://www.lydtechconsulting.com/blog-kafka-streams-windows-overview.html)  
[Apache Kafka Beyond the Basics: Windowing](https://www.confluent.io/ko-kr/blog/windowing-in-kafka-streams/)  
[Apache Kafka Sliding Windows](https://kafka.apache.org/30/javadoc/org/apache/kafka/streams/kstream/SlidingWindows.html)  
[Apache Kafka Session Windows](https://kafka.apache.org/21/javadoc/org/apache/kafka/streams/kstream/SessionWindows.html)  



