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
아래는 로그 수집기 역할을 하는 `fluentd` 이미지를 사용한 데몬셋 템플릿 예시이다. 

```yaml
# daemonset-fluentd.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: daemonset-fluentd
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: daemonset-fluentd
    spec:
      containers:
        - name: daemonset-fluentd
          image: fluent/fluentd-kubernetes-daemonset:elasticsearch
          env:
            - name: env1
              value: value1
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
```  

- `.metadata.namespace` : 로그 수집기는 관리용 파드 설정에 해당하기 때문에 `kube-system` 네임스페이스로 설정한다. 
- `.metadata.labels` : 데몬셋 컨트롤러 오브젝트를 식별하는 레이블의 키로는 `k8s-app` 값으로는 `fluentd-logging` 으로 설정한다. 
- `.spec.updateStrategy.type` : 데몬셋 업데이트 방식을 설정하는 필드로 `RollingUpdate` 를 설정했다. 
설정 가능한 다른 값으로는 `OnDelete` 가 있다. 
`1.5` 이하 버전에서는 `Ondelete` 가 기본값이지만, 이후 버전은 `RollingUpdate` 가 기본값이다. 
`RollingUpdate` 는 템플릿 수정시 바로 업데이트를 수행하게 되는데, 
`.spec.updateStrategy.rollingUpdate.maxUnavailable`, `.spec.minReadySeconds` 필드로 한번에 교체하는 파드 개수를 조절 할 수 있다. 
- `.spec.template.spec.containers[].image` : 로그 수집기로 사용하는 `fluentd` 이미지를 설정 한다. 

`kubectl apply -f daemonset-fluentd.yaml` 명령을 통해 데몬셋 템플릿을 적용한다. 
그리고 `kubectl get daemonset -n kube-system` 명령을 통해 조회하면 아래와 같다. 

```bash
$ kubectl apply -f daemonset-fluentd.yaml
daemonset.apps/daemonset-fluentd created
$ kubectl get daemonset -n kube-system
NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset-fluentd   1         1         1       1            1           <none>                        47s
kube-proxy          1         1         1       1            1           beta.kubernetes.io/os=linux   4d22h
```  

현재 쿠버네티스 클러스터에 구성된 노드의 수는 1개이기 때문에 1개의 `daemonset-fluentd` 만 생성 되었다. 
만약 여러개의 노드가 구성된 상태라면 노드의 수만큼 생성된다. 

### 업데이트
















































---
## Reference
