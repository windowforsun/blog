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
  - Object
  - Controller
  - Namespace
  - Template
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

오브젝트와 클러스터의 다른 종류에 대해서는 추후에 다루고, 
클러스터를 논리적인 단위로 구분할 수 있도록 해주는 네임스페이스와 오브젝트와 컨트롤러의 상태를 정의하는 템플릿에 대해 알아본다. 

## 네임스페이스
쿠버네티스에서 네임스페이스는 하나의 클러스터를 여러 개의 논리적인 단위로 나눠 사용할 수 있도록 해준다. 
이런 네임스페이스를 통해 클러스터 하나에 여러 앱의 구성을 동시에 사용 할 수 있다. 
그리고 클러스터에서 용도에 맞춰 실행해야 하는 앱을 구분 할 수도 있다. 
또한 네임스페이스별로 사용량을 제한 할 수도 있다.  

쿠버네티스를 처음 설치하게 되면 구성되는 기본 네임스페이스는 `kubectl get namespaces` 를 통해 확인 할 수 있다. 

```bash

```  

- `default` : 기본 네임스페이스로 네임스페이스를 지정하지 않게 되면 설정되는 네임스페이스이다. 
- `kube-system` : 쿠버네티스 시스템에서 사용하는 네임스페이스로 관리용 파드나 설정이 있다. 
- `kube-public` : 클러스터를 사용하는 모든 사용자가 접근할 수 있는 네임스페이스로 클러스터 사용량 정보 등을 관리한다. 
- `kube-node-lease` : 클러스터에 구성된 각 노드의 임대 오브젝트(`Lease object`) 를 관리하는 네임스페이스이다. 

`kubectl` 명령을 통해 네임스페이스를 지정할 때는 `--namespace=<네임스페이스>` 와 같이 사용한다.  

앞서 설명한 것처럼 네임스페이스를 지정하지 않으면 기본 네임스페이스(`default`) 가 설정되는데, 
기본 네임스페이스를 변경하는 방법은 우선 `kubectl config current-context` 를 통해 현재 컨텍스트 정보를 확인해야 한다. 

```bash

```  

명령어 출력에서 `NAMESPACE` 부분이 비어 있는데, 이는 설정된 기본 네임스페이스가 `default` 라는 것을 의미한다. 
`kubectl config set-context <컨텍스트이름> --namespace=<기본네임스페이스>` 을 통해 기본 네임스페이스를 변경해 본다. 

```bash

```


---
## Reference
