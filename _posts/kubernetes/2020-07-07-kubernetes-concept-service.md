--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 서비스(Service)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Service
toc: true
use_math: true
---  

## 서비스
서비스(`Service`) 는 여러 파드에 접근할 수 있는 `IP` 를 제공하는 로드밸런서 역할을 수행한다. 
다른 여러기능도 있지만, 주로 로드밸런서로 사용된다.  

쿠버네티스 클러스터는 여러 노드로 구성돼있고, 노드에서 실행되는 파드는 컨트롤러에서 관리한다. 
이는 한 파드가 `A` 노드에서 배포나 특정 동작으로 인해 `B` 라는 다른 노드로 이동할 수 있다는 의미이다. 
그리고 이런 과정에서 파드의 `IP` 가 변경되기도 한다.  

위와 같이 동적인 특성을 띄는 파드를 고정적인 방법으로 접근이 필요할 때 서비스를 사용할 수 있다. 
위에서 언급한 것과 같이 서비스는 쿠버네티스 클러스터에서 파드에 접근할 수 있는 고정 `IP` 를 제공한다. 
또한 클러스터 외부에서 접근할 수도 있다. 
그리고 서비스는 `L4` 영역에서 위와 같은 역할을 수행한다. 

### 서비스의 종류
서비스는 아래와 같은 4가지 종류가 있다. 

- `ClusterIP` : 서비스의 기본 타입으로 쿠버네티스 클러스터 내부 접근시에만 사용할 수 있다. 
클러스터에 구성된 노드나 파드에서 클러스터 `IP` 로 서비스에 연결된 파드에 접근할 수 있다. 
- `NodePort` : 클러스터 외부에서 내부로 접근할 때 가장 간단한 방법으로, 
서비스 하나에 모든 노드의 지정된 포트를 할당한다. 


---
## Reference
