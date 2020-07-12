--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 서비스(Service) 와 kube-proxy"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '각 노드에서 서비스를 다루고 네트워크를 관리하는 kube-proxy 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Service
  - kube-proxy
toc: true
use_math: true
---  

## kube-proxy
`kube-proxy` 는 쿠버네티스 클러스터에 구성된 서비스를 통해 각 노드 내부에 접근 할 수 있도록 컨트롤하는 컴포넌트이다. 
각 노드마다 `kube-proxy` 가 실행 되면서 클러스터 내부 `IP` 로 연결 요청을 적절한 파드로 전달한다.  

`kube-proxy` 가 노드에서 네트워크를 관리하는 방법은 아래와 같은 3가지 종류가 있다. 
- `userspace`
- `iptables`
- `IPVS`

`kube-proxy` 의 기본 모드는 초기에는 `userspace` 였지만, 
2019 년도 쯤 `iptables` 로 변경 되었고 또 다시 `IPVS` 로 변경 될 수 있다. 

### userspace

![그림 1]({{site.baseurl}}/img/kubernetes/concept_service_kube_proxy_userspace_plant.png)

`userspace` 는 클라이언트에서 구성된 서비스의 `ClusterIP` 로 요청을 하면, 
`iptables` 를 거쳐 `kube-proxy` 가 이를 전달 받는다. 
전달 받은 `kube-proxy` 는 서비스의 `ClusterIP` 가 전달 되야할 적절한 파드를 찾아 연결 한다. 
이때 다수 개의 파드가 있을 경우 라운드 로빈(`Round robin`) 방식으로 수행한다.  

`userspace` 에서 파드로의 요청이 실행하면 자동으로 다른 파드에 연결을 재시도 한다. 

### iptables

![그림 1]({{site.baseurl}}/img/kubernetes/concept_service_kube_proxy_iptables_plant.png)

앞서 설명한 `userspace` 와 차이점은 `kube-proxy` 가 `iptables` 를 관리하는 역할만 수행하고, 
직접 클라이언트의 요청은 전달 받지 않는다.  

클라이언트의 요청은 `iptables` 를 거쳐 적절한 파드로 바로 전달된다. 
이러한 과정으로 `userspace` 보다 요청 처리에 대한 성능이 뛰어나다. 
하지만 `userspace` 처럼 파드에 요청 실패시, 재연결과 같은 과정은 없고 실패로 끝나게 된다.  

구성된 컨테이너에 `readinessProbe` 가 설정되고 헬스 체크가 성공해야 연결 요청이 해당 파드로 이뤄진다. 

### IPVS

![그림 1]({{site.baseurl}}/img/kubernetes/concept_service_kube_proxy_ipvs_plant.png)

`IPVS`(`IP Virtual SErver`) 는 리눅스 커널에 있는 `L4` 로드밸런싱 기술을 의미한다. 
리눅스 커널 안의 네트워크 프레임워크 `Netfilter` 에 포함돼 있다. 
쿠버네티스에서 `IPVS` 를 사용하기 위해서는 커널 모듈에 `IPVS` 관련 의존성이 설치돼 있어야 한다.  

`IPVS` 는 커널 공간에서 동작하고, 
데이터 구조를 해시 테이블로 관리하기 때문에 `iptables` 보다 성능 적으로 우세하다. 
또한 보다 다양한 로드밸런싱 알고리즘을 이용할 수 있다. 
- `rr(round-robin)` : 어떠한 우선순위 없이 순서, 시간을 단위로 할당한다. 
- `lc(least connection)` : 접속의 수가 가장 적은 순으로 할당한다. 
- `dh(destination hashing)` : 목적지 IP 주소로 해싱한 값을 기준으로 할당한다.
- `sh(source hashing)` : 출발지 IP 주소로 해싱한 값을 기준으로 할당한다. 
- `sed(shortes expected delay)` : 응답 속도가 빠른 순으로 할당 한다. 
- `nq(never queue)` : `sed` 와 비슷한 방식이지만, 활성 접속 수가 0개인 것을 우선해서 할당한다.


---
## Reference
