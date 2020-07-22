--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 애너테이션(Annotation)"
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
  - Annotation
toc: true
use_math: true
---  

## 애너테이션
애너테이션(`Annotation`) 은 레이블과 비슷한 구조인 `key-value` 구조를 가지고 있다. 
그리고 사용자 필요에 따라 설정하거나 구성할 수 있다.  

하지만 레이블과 차이점이 있는데
레이블은 사용자가 정해진 값 없이 설절하고 클러스터에서 오브젝트를 식별하는데 사용된다면, 
애너테이션은 쿠버네티스 클러스터 시스템에 필요한 정보를 표현하거나, 
쿠버네티스 클라이언트나 라이브러리의 자원 관리에 사용된다.  

이렇게 애너테이션은 쿠버네티스가 인식할 수 있는 정해진 값을 사용한다. 
디플로이먼트에서 앱배포시에 변경 정보를 작성하거나, 
인그레스 컨트롤러인 `ingress-nginx` 를 애너테이션을 사용해서 직접 `nginx` 관련 설정을 수정할 수 있다. 
또한 사용시에 필요할 수 있는 릴리즈 정보, 로깅, 모니터링 메모하는 용도로 사용 할 수도 있다. 
























































---
## Reference
