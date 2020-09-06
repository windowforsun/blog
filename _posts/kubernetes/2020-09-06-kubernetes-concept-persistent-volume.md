--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 퍼시스턴트 볼륨 과 클레임(PV, PVC)"
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
  - Volume
  - PV
  - PVC
toc: true
use_math: true
---  

## PV, PVC
쿠버네티스에서는 `PV` 라고 불리는 퍼시스턴트 볼륨(`PersistentVolume`)` 과 `PVC` 라고 하는 퍼시스턴스 볼륨 클레임(`PersistentVolumeClaim`) 을 사용해서 구성한다.  

`PV` 는 볼륨 그 자체를 의미하고 클러스터에서 자원으로 관리된다. 파드와 별개의 생명주기를 가지고 관리되는 요소이다. 
그리고 `PVC` 는 사용자가 클러스터에 존재하는 `PV` 에게 수행하는 요청을 의미한다. 
사용하려는 용량, 읽기, 쓰기 등을 설정해서 요청을 할 수 있다.  

이렇게 쿠버네티스에서는 볼륨을 사용할때 파드에서 직접 구성하는 것이 아닌, 
`PVC` 라는 매개체를 사용해서 구성한 저장소인 `PV` 를 사용할 수 있도록 했다. 
이런 구조를 통해 파드는 필요에 따라 다양한 스토리지를 사용할 수 있다.  

클라우드 서비스를 사용한다면 서비스에서 제공하는 스토리지를 사용할 수 있고, 
자체적으로 구성한 스토리지가 있다면 해당 스토리지를 사용할 수도 있다. 
이는 `PV` 와 `PVC` 를 통해 파드와 연결되기 때문에 파드에서는 실제 스토리지의 종류에 의존성없이 다양한 구성이 가능하다.  

## PV, PVC 생명주기
`PV` 와 `PVC` 의 생명주기는 아래와 같다.  

![그림 1]({{site.baseurl}}/img/kubernetes/concept_persistent_volume_pvpvc_lifecycle_plant.png)

### 프로비저닝
볼륨을 사용하기 위해서는 먼저 `PV` 를 만들어야 한다. 
`PV` 를 만드는 단계를 프로비저닝(`Provisioning`) 이라고 한다. 
그리고 프로비저닝의 방법으로는 `PV` 를 미리 만들고 사용하는 정적 방법과 요청이 있을 때 `PV` 를 만드는 동적 방법이 있다.  

정적인 방법은 클러스터 관리자가 미리 적정 용량의 `PV` 를 만들어 두고 요청이 있으면 만들어둔 `PV` 를 할당한다. 
이러한 방법은 사용가능한 용량 제한이 있을 경우 유용하다. 
만약 `PV` 용량이 `100GB` 로 만들어져 있을 때, 
`150GB` 에 대한 요청은 모두 실패하게 된다.  

동적인 방법은 `PVC` 를 통해 `PV` 를 요청했을 때 생성하고 볼륨이 제공된다. 










---
## Reference
