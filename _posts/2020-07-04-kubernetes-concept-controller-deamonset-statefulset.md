--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 컨트롤러(DaemonSet, StatefulSet)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 파드를 관리하는 컨트롤러 중 DaemonSet, StatefulSet 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Controller
  - DaemonSet
  - StatefulSet
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
실행 중인 데몬셋의 설정을 변경하는 방법은 `kubectl edit daemonset <데몬셋 이름> -n <데몬셋 네임이스페이스>` 
명령을 사용해서 가능하다.  

`kubectl edit daemonset daemonset-fluentd -n kube-system` 명령으로 설정을 열어 `.spec.template.spec.containers[].env[]` 에 `env2` 환경 변수를 추가한다. 
실행 중인 데몬셋의 업데이트 타입이 `RollingUpdate` 이기 때문에 수정 사항은 바로 적용된다. 
업데이트 확인은 `kubectl describe daemonset -n kube-system` 명령으로 가능하고 아래와 같은 결과를 확인할 수 있다. 

```bash
spec:
  template:
    spec:
      containers:
      - env:
        - name: env1
          value: value1
		- name: env2
		  value: value2

$ kubectl edit daemonset daemonset-fluentd -n kube-system
daemonset.extensions/daemonset-fluentd edited
$ kubectl describe daemonset -n kube-system

.. 생략 ..

    Environment:
      env1:  value1
      env2:  value2

.. 생략 ..
```  

이번에는 `.spec.updateStrategy.type` 을 `OnDelete` 로 수정해서 업데이트 상황을 확인해 본다. 
`OneDelete` 로 설정하게 되면 업데이트 사항은 파드를 직접 삭제해야 적용된 파드가 실행된다. 
`.spec.template.spec.containers[].env[]` 에서 `env1` 의 값을 `new-value1` 으로 수정하고, 
`updateStrategy` 도 `OneDelete` 로 수정한다. 

```bash
spec:
  template:
    spec:
      containers:
      - env:
        - name: env1
          value: new-value1
        - name: env2
          value: value2
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: OnDelete

$ kubectl edit daemonset daemonset-fluentd -n kube-system
daemonset.extensions/daemonset-fluentd edited
```  

지금까지는 `RollingUpdate` 로 데몬세트 업데이트가 수행되기 때문에, `OnDelete` 설정과 환경변수 수정은 바로 적용 된다.  

`OnDelete` 업데이트 수행을 위해 다시 `kubectl eidt` 명령으로 `env2` 의 값을 `new-value2` 로 수정한다. 
그리고 `kubectl get daemonset -n kube-system` 명령으로 확인하면 아래와 같다. 

```bash
spec:
  template:
    spec:
      containers:
      - env:
        - name: env1
          value: new-value1
        - name: env2
          value: new-value2

$ kubectl edit daemonset daemonset-fluentd -n kube-system
daemonset.extensions/daemonset-fluentd edited
$ kubectl get daemonset -n kube-system
NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset-fluentd   1         1         0       0            0           <none>                        24m
kube-proxy          1         1         1       1            1           beta.kubernetes.io/os=linux   31m
```  

`UP-TO-DATE` 필드가 0인 상태는 설정이 변경되었지만, 아직 데몬셋 파드에 반영되지 않았음을 뜻한다. 
반영을 위해 `kubectl get pods -n kube-system` 데몬셋 파드를 조회하고, 
`kubectl delete pods <파드 이름> -n kube-system` 명령으로 해당 파드를 삭제한다. 

```bash
$ kubectl get pods -n kube-system
NAME                                     READY   STATUS             RESTARTS   AGE
coredns-5c98db65d4-cqjtg                 1/1     Running            7          33m
coredns-5c98db65d4-njcb6                 1/1     Running            7          33m
daemonset-fluentd-x2jbn                  0/1     CrashLoopBackOff   5          15m  # 데몬셋 파드
etcd-docker-desktop                      1/1     Running            0          32m
kube-apiserver-docker-desktop            1/1     Running            7          33m
kube-controller-manager-docker-desktop   0/1     CrashLoopBackOff   6          33m
kube-proxy-6mxf2                         1/1     Running            0          33m
kube-scheduler-docker-desktop            0/1     CrashLoopBackOff   6          32m
$ kubectl delete pods daemonset-fluentd-x2jbn -n kube-system
pod "daemonset-fluentd-x2jbn" deleted
```  

어느정도 시간이 지나고 데몬셋 조회를 하면 `UP-TO-DATE` 필드가 1로 적용되었고, 
파드 조회를 하면 데몬셋에 해당하는 새로운 파드가 생성된 것을 확인 할 수 있다. 

```bash
$ kubectl get daemonset -n kube-system
NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset-fluentd   1         1         1       1            1           <none>                        32m
kube-proxy          1         1         1       1            1           beta.kubernetes.io/os=linux   38m
$ kubectl get pods -n kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-5c98db65d4-cqjtg                 1/1     Running   7          38m
coredns-5c98db65d4-njcb6                 1/1     Running   7          38m
daemonset-fluentd-mjhz9                  1/1     Running   0          32s
etcd-docker-desktop                      1/1     Running   0          37m
kube-apiserver-docker-desktop            1/1     Running   7          37m
kube-controller-manager-docker-desktop   1/1     Running   7          37m
kube-proxy-6mxf2                         1/1     Running   0          38m
kube-scheduler-docker-desktop            1/1     Running   7          37
```  

## 스테이트풀세트
상태가 없는 파드를 관리하는 레플리케이션 컨트롤러, 레플리카세트, 디플로이먼트와는 달리,
스테이트풀세트(`StatefulSet`)는 상태가 있는 파드를 관리하는 컨트롤러이다. 
스테이트풀세트를 사용하면 볼륨(`Volume`)을 마운트해서 특정 데이터를 노드에 저장해서 유지 할 수 있다. 
그리고 여러 파드사이의 순서를 지정해 실행할 수도 있다. 

### 스테이트풀 템플릿
아래는 `Nginx` 이미지를 사용해서 구성한 스테이트풀세트 템플릿 예시이다. 

```yaml
# statefulset-nginx.yaml

apiVersion: v1
kind: Service
metadata:
  name: statefulset-nginx-service
  labels:
    app: statefulset-nginx-service
spec:
  ports:
    - port: 80
      name: nginx-web
  clusterIP: None
  selector:
    app: statefulset-nginx-service

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-web
spec:
  selector:
    matchLabels:
      app: statefulset-nginx
  serviceName: "statefulset-nginx-service"
  replicas: 3
  template:
    metadata:
      labels:
        app: statefulset-nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: statefulset-nginx
          image: nginx:latest
          ports:
            - containerPort: 80
              name: nginx-web
```  














































---
## Reference
