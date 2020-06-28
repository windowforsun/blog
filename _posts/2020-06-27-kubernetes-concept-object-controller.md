--- 
layout: single
classes: wide
title: "[Kubernetes 개념] "
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
toc: true
use_math: true
---  

## 오브젝트와 컨트롤러
쿠버네티스의 시스템은 크게 오브젝트(`Object`) 와 오브젝트를 관리하는 컨트롤러(`Controller`) 로 구성된다. 
사용자는 템플릿이나 커맨드라인을 통해 바라는 상태(`desired state`) 를 정의하면,
컨트롤러는 현재 상태가 정의된 바라는 상태가 될 수 있도록 오브젝트를 생성/삭제 한다.  

오브젝트는 아래와 같은 종류가 있다.
- [파드(Pod)](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-overview/)
- [서비스(Service)](https://kubernetes.io/ko/docs/concepts/services-networking/service/)
- [볼륨(Volume)](https://kubernetes.io/ko/docs/concepts/storage/volumes/)
- [네임스페이스(Namespace)](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/namespaces/)

그리고 컨트롤러에는 아래와 같은 종류가 있다. 
- [디플로이먼트(Deployment)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)
- [데몬셋(DaemonSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/daemonset/)
- [스테이트풀셋(StatefulSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/statefulset/)
- [레플리카셋(ReplicaSet)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/replicaset/)
- [잡(Job)](https://kubernetes.io/ko/docs/concepts/workloads/controllers/jobs-run-to-completion/)


---
## Reference
