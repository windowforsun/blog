--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 인그레스(Ingress) 무중단 배포"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스에서 무중단 배포와 인그레스를 사용한 무중단 배포에 대해 살펴보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Ingress
  - HA
toc: true
use_math: true
---  

## 무중단 배포
쿠버네티스 클러스터에서 배포를 할때 상황을 살펴본다. 
클러스터에서 새로운 파드를 배포하고 교체 전의 상황은 아래와 같다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_ingress_nonstop_1_plant.png)

각 노드에는 이전 배포 버전인 `Pod(v1)` 과 새로운 배포 버전인 `Pod(v2)` 가 같이 있는 상황이면서, 
`Nginx` 는 `Pod(v1)` 에게 요청을 전달 하고 있다.  

`Pod(v2)` 의 구성과 실행이 완료되면 아래 그림과 같이 `Nginx` 에서 전달하는 요청은 `Pod(v2)` 가 받게 되고, 
이전 배포 버전인 `Pod(v1)` 은 삭제된다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_ingress_nonstop_2_plant.png)

쿠버네티스 클러스터에서 위와 같이 자연스러운 배포를 수행하기 위해서는 아래와 같은 설정와 확인 사항이 필요하다. 

### maxSurge, maxUnavailable
각 노드의 파드 관리를 `RollingUpdate` 로 설정 했을 경우, `maxSurge`, `maxUnavailable` 필드의 설정이 필요하다.  

`maxSurge` 는 디플로이먼트를 사용해서 배포할 때 템플릿에 설정한 파드의 수보다 몇개의 파드를 추가로 생성 할 수 있는지에 대한 설정이다.  

그리고 `maxUnavailable` 은 디플로이먼트를 통해 업데이트를 수행할 때 최대 몇개의 파드가 비활성화 상태가 될 수 있는지 설정하는 필드이다.  

서비스 중 트래픽 유실 없이 배포를 수행하기 위해서는 현재 구성 환경에 알맞게 두 필드의 값을 설정해 줘야 한다.  


### readinessProbe
[파드]({{site.baseurl}}{% link _posts/kubernetes/2020-06-29-kubernetes-concept-pod.md %})
에서 언급한 것과 같이, 현재 파드의 상태 파악을 위해서는 `livenessProbe` 와 `readinessProbe` 두 가지 프로브 설정이 필요하다. 
`livenessProbe` 는 파드의 컨테이너가 정상 실행 여부를 파악해서 `kubelet` 에서 다시 재시작 설정에 맞게 관리하도록 한다.  

무중단 배포에서는 `readinessProbe` 을 주의 깊게 설정해야 한다. 
`readinessProbe` 는 파드의 컨테이너가 서비스 요청을 처리할 준비가 돼있는지 파악할 수 있는 설정이다. 
`readinessPribe` 가 성공해야 파드와 연결된 서비스에서 트래픽을 전달하게 된다.  

특정 애플리케이션의 컨테이너의 경우 컨테이너의 실행과는 별개로 실제 서비스를 수행 하기까지 걸리는 초기화 과정이 소요된다. 
이런 상황에서 바로 파드쪽으로 트래픽이 전달되면 서비스를 정상적으로 처리할 수 없기 때문에, 
`readinessProbe` 설정을 통해 이를 극복해야 한다.  

만약 `readinessProbe` 의 설정이 어려운 경우 `.spec.minReadySeconds` 필드를 사용해서 파드가 준비 상태가 될 때까지 대기 시간을 설정 할 수 있다. 
대기시간 전까지는 해당 파드로 트래픽이 전달되지 않는다. 
만약 `readinessProbe` 와 `.spec.minReadySeconds` 필드가 모두 설정된 경우라면,
`readinessProbe` 가 성공하게 되면 `.spec.minReadySeconds` 의 대기 시간은 무시된다. 

### Graceful 종료
각 노드에서 컨테이너를 관리하는 `kubelet` 은 배포 과정에서 새로운 파드를 생성하고 이전 파드를 종료 할때, 
이전 파드에게 먼저 `SIGTERM` 신호를 보낸다. 
트래픽 유실 없는 무중단 배포를 위해서는 컨테이너에서는 `SIGTERM` 신호를 받았을 때 현재 처리중인 요청 까지만 수행하고, 
다음 요청은 받지 않는 `Graceful` 한 종료 처리가 필요하다.  

만약 아래 그림과 같이 `Nginx` 에서 아직 `Pod(v1)` 쪽으로  트래픽을 전송하고 해당 파드가 요청을 처리 중에 `SIGTERM` 을 받아 종료가 된다면, 
해당 요청에서는 에러가 발생하게 된다. 
또한 `kubelet` 에서는 `SIGTERM` 을 전송 한 후, 
`.terminationGracePeriodSeconds` 필드에 설정 가능한 기본 대기 시간(30초)이 지나면 `SIGKILL` 신호를 보내 강제 종료 시킨다.  

![그림 1]({{site.baseurl}}/img/kubernetes/concept_ingress_nonstop_3_plant.png)

특정 경우 컨테이너에 `Graceful` 한 종료 처리를 설정하지 못할 수도 있다.
이런 상황에서는 파드 생명 주기의 훅을 설정해 사용할 수 있다. 
흑은 파드 실행 후 실행되는 `Poststart hook` 과 종료 직전에 실행되는 `Prestop hook` 이 있다. 
여기서 `Prestop hook` 은 `SIGTERM` 신호를 보내기 전에 실행되기 때문에, 컨테이너 설정과는 별개로 `Graceful` 효과를 낼 수 있다. 
`Prestop hook` 이 완료 되기 전까지는 컨테이너에 `SIGTERM` 신호를 보내지 않기 때문에, 파드에서 이를 조정 할 수 있다.  

앞서 설명한 `.terminationGracePeriodSeconds` 시간이 지나면 `SIGKILL` 을 통해 강제 종료를 시킨다는 점에 유의 해야 한다.


## Ingress 무중단 배포






---
## Reference
