--- 
layout: single
classes: wide
title: "[Kafka 개념] Kafka on Kubernetes using Strimzi"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Strimzi 를 사용해서 Kubernetes 에 Kafka 를 구성하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
  - Kafka
  - Concept
  - Kafka
  - Kubernetes
  - Strimzi
  - Minikube
toc: true
use_math: true
---  

## Kubernetes 와 Kafka
`Kuberntes` 와 `Kafka` 이 두개의 키워드로 검색을 하면 아래 질문에 대한 글이 보인다. 

> `Kafka`, `Kuberntes` 위에 올리는게 좋을까 ?

`Kuberntes` 를 사용해서 `Kafka` 를 어떻게 구성하나요 ?
라는 질문을 하기전에 꼭 `Kubernetes` 를 사용해야 하는지에 대해 먼저 고민이 필요하다.  

먼저 `Kafka` 를 `Kuberntes` 를 사용해서 구성해야 하는 이유와 장점은 아래와 같은 것들이 있다. 

- 표준화된 배포 환경
현재 서비스를 구성하는 모든 혹은 대부분의 애플리케이션들이 `Kubernetes` 환경이라면, 
`Kafka` 만 별도의 환경에서 구성하고 관리하는게 유지보수 측면에서 더 많은 비용이 들어갈 수 있다. 
또한 `Kubernetes` 가 지니고 있는 장점을 그대로 `Kafka` 인프라에도 사용할 수 있다는 점이 있다. (간편한 `scale out`, `scale up` 등..)

- 빠른 서비스 구성
`Kubernetes` 를 사용해서 새로운 인프라를 구성하는 것은 이미 표준화된 작업이 완료된 것을 사용한다는 의미이기도 하다. 
복잡한 설치와 설정 과정이 이미 간편하게 제공되고 있기 때문에 사용하는 입장에서는 쉽고 간편하다. 

위와 같은 장점들이 있지만 다시한번 고민해보라는 것은 아래와 같은 이유와 단점들이 있기 때문이다.  

- Resource
`Kubernetes` 환경은 말그대로 추상화된 레이어를 사용해서 그 위에 애플리케이션을 올리는 것을 의미한다. 
`Kuberntes` 를 사용하지 않는 환경과 비교해서 추상화가 늘어 났기 때문에 추가적인 리소스 사용과 함께 성능에 영향을 줄 수 있다. 
`Kafka` 는 운영체제 위에 바로 설치한 후에 튜닝을 하며 고성능을 끌어올리는 경우가 많은데 `Kuberntes` 위에 올리는 경우 튜닝에 어려움이 있다고 한다.  

- Monitoring
위에서 언급한 추상화 레이어가 늘어는 것은 모니터링에도 영향을 끼쳐 모니터링을 할 요소들이 더 늘어날 수 있다. 
`Kafka` 는 큰 부하가 발생하는 요소들이 실시간으로 변경되는 특성이 있다. 
그리고 일반적인 클러스터 구조를 갖는 애플리케이션들과 달리 클러스터 수를 조절한다고 해서 부하가 해소되는 특성을 가지지 않기 때문에 `Kubernetes` 환경에서 모니터링이 더 어려울 수 있다. 

- Stateful
`Kafka` 는 일반적인 서비스 애플리케이션이 가지는 `Stateless` 가 아닌 `Stateful` 성격을 갖는 애플리케이션이다. 
이러한 이유로 설정이 다른 애플리케이션을 구성하는 것에 비해 까다롭고, 동작을 위해 유지가 필요한 구성에 따른 어려움도 있다.  

- 표준화된 `Kafka` 도커 이미지의 부재
`Kafka` 는 `Kafka Server` 와 `Zookeeper` 의 조합으로 구성되기 때문에 2개 모두 구성을 반드시 해줘야 한다. 
`Zookeeper` 의 경우 표준화된 공식 이미지가 존재해서 이를 사용하면 되지만, 
`Kafka` 는 공식 이미지가 아직 존재하지 않는 상태이다. 
어떤 이미지를 사용할지, 해당 이미지에 대한 정보 및 자료는 적절하게 있는지에 대한 조사부터 필요할 수 있다.  

### Kafka on Kubernetes
위 글을 통해 먼저 `Kafka` 를 꼭 `Kubernetes` 에 구성해야 하는지에 대해 알아보았다. 
우선 현재 대부분의 애플리케이션들이 `Kubernetes` 환경에서 구성되고, 모니터링, 배포 등의 
전반적은 `Devops` 가 `Kubernetes` 를 주축으로 사용중이기 때문에 `Kubernetes` 환경에 `Kafka` 를 구성할 필요가 있다고 느껴졌다.  

그리고 꼭 서비스 뿐만 아니더라도, 테스트 용도로 간단하게 구성해서 사용하고 깔끔하게 제거 하기위해서도 `Kubernetes` 환경에 `Kafka` 를 구성할 필요가 있다고 생각한다.  
`Kubernetes` 에 `Kafka` 를 구성하는 방법은 자료조사를 해보면 아주 다양한 방법들이 보인다. 
그 중에 소개할 방법은 
[Strimzi](https://strimzi.io/) 를 사용하는 방법이다.  

## Strimzi







---
## Reference
[Strimzi Quick Starts](https://strimzi.io/quickstarts/)  
[Apache Kafka on Kubernetes – Could You? Should You?](https://www.confluent.io/blog/apache-kafka-kubernetes-could-you-should-you/)  
