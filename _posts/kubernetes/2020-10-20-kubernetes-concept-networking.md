--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 네트워킹"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '파드 및 쿠버네트스 클러스터에서 서비스의 네트워킹과 플러그인에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  
toc: true
use_math: true
---  

## 파드 네트워킹
쿠버네티스 파드의 네트워킹 관련을 이해하기 위해서는 먼저 도커 네트워크에 대한 이해가 필요하다. 
해당 부분은 [여기]({{site.baseurl}}{% link _posts/docker/2020-10-19-docker-practice-networking.md %})
를 참고하도록 한다.  

쿠버네티스는 파드 단위로 컨테이너를 관리하고, 
파드는 쿠버네티스에서 생성한 `pause` 컨테이너와 함께 사용자가 생성한 컨테이너 그룹을 의미한다. 
그리고 하나의 파드에 속한 컨테이너는 모두 동일한 `IP` 를 갖게 된다.  

![그림 1]({{site.baseurl}}/img/kubernetes/concept_networking_pod_1.png)

위 그림과 같이 파드에 속한 컨테이너는 하나의 `veth0` 을 공유하게 된다. 
하나의 파드에 속한 사용자 컨테이너는 `pause` 컨테이너 네트워크 네임스페이스에 해당하므로, 
모두 같은 `IP` 를 사용하게 된다. 
`pause` 컨테이너는 인프라 컨테이너 역할로 쿠버네티스가 생성하고 관리한다.  







---
## Reference
