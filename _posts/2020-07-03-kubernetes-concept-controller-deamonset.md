--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 컨트롤러(DaemonSet)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 파드를 관리하는 컨트롤러 중 DaemonSet 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Controller
  - DaemonSet
toc: true
use_math: true
---  

## 데몬세트
데몬세트(`DaemonSet`) 는 쿠버네티스 클러스터에 구성된 전체 노드에서 특정 파드 실행이 필요할 때 사용하는 컨트롤러이다. 
클러스터에 새로운 노드가 추가되게 되면 데몬세트는 해당 노드에 설정된 파드를 자동으로 실행하고, 
노드가 제거된 경우에는 파드를 다른 노드에 실행하는 동작은 수행하지 않는다.  

데몬세트는 이러한 특징으로 각 노드의 로그 수집기나 모니터링과 같이 클러스터 전체 노드에서 항상 실행되야 할 파드를 구성할 때 자주 사용한다.  

### 데몬세트 템플릿


















































---
## Reference
