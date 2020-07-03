--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 컨트롤러(Replication Controller, Replica Set)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 파드를 관리하는 컨트롤러 중 Replication controller 와 Replica Set 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Controller
  - Replication Controller
  - Replica Set
toc: true
use_math: true
---  

## 컨트롤러
컨트롤러는 쿠버네티스 클러스터에서 파드들을 관리하는 역할을 수행한다. 
컨트롤러에는 파드들을 다양한 목적에 맞게 관리할 수 있도록 아래와 같은 종류가 있다. 
- 레플리케이션 컨트롤러(`Replication Controller`)
- 레플리카세트(`Replica Set`)
- 디플로이먼트(`Deployment`)
- 데몬세트(`Daemon Set`)
- 스테이트풀세트(`Stateful Set`)
- 잡(`Job`)
- 크론잡(`CronJob`)

## 레플리케이션 컨트롤러
레플리케이션 컨트롤러(`Replication Controller`)는 가장 기본적인 컨트롤러로, 
지정한 숫자만큼 파드가 클러스터에 실행 될 수 있도록 관리한다. 
지정된 숫자만큼 파드를 실행하고 실제로 클러스터에 실행 중인 파드가 개수와 맞는지 검사하면서 개수를 조정한다. 
레플리케이션 컨트롤러를 사용하게 되면 특정 노드의 장애로인해 파드가 종료되면, 다른 노드에 다시 파드를 실행하는 식으로 관리한다. 
레플리케이션 컨트롤러는 등호 기반(`equality-based`) 이기 때문에 레이블을 선택할 때 같은지 다른지만 판별한다. 

초기에 사용됐던 컨트롤러이지만 현재는 비슷한 역할을 수행하는 컨트롤러의 등장으로 배포에는 디플로이먼트를 사용한다. 

## 레플리카세트
레플리카세트(`Replicaset`) 는 레플리케이션 컨트롤러의 발전형태로 같은 동작을 수행하면서 집합 기반(`set-based`) 셀렉터(`selector`) 를 지원한다.  
집합 기반 셀렉터는 레블을 선택할 때 `in`, `notin`, `exists` 등과 같은 연산자를 지원한다.  

그리고 레플리케이션 컨트롤러에서는 `rolling-update` 를 사용할 수 있지만, 
레플리카세트에서는 불가하다. 
추가로 `rolling-update` 를 사용하기 위해서는 디플로이먼트를 사용해야 한다. 

### 레플리카세트 템플릿
아래는 `Nginx` 이미지를 사용해서 구성한 레플리카서트의 템플릿 예시이다. 

```yaml
# replicaset-nginx.yaml

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  template:
    metadata:
      name: replicaset-nginx
      labels:
        app: replicaset-nginx
    spec:
      containers:
        - name: repliacaset-nginx
          image: nginx:latest
          ports:
            - containerPort: 80
  replicas: 3
  selector:
    matchLabels:
      app: replicaset-nginx
```  

- `.spec.template.metadata` : `.spec.template` 는 구성하는 레플리카세트가 어떤 파드를 실행할지에 대해서 설정한다. 
이를 위해 `.spec.template` 하위에 `.metadata` 를 사용해서 실행할 파드의 이름과 식별을 위한 레이블 을 설정한다. 
- `.spec.template.spec.controllers[]` : `.spec.template.spec` 는 살행하는 파드에 대한 정보를 설정한다. 
하위 `.controllers[]` 에서는 `.name`, `.image`, `.ports[]`, `containerPort` 를 사용해서 실행하는 컨테이너의 구체적인 명세를 정의한다. 
- `.spec.replicas` : 구성하는 레플리카세트에서 몇개의 파드를 유지할지 설정한다. 기본값은 1이다.
- `.spec.selector` : 구성하는 레플리카세트에서 어떤 레이블의 파드를 선택해서 관리할지 설정한다. 
레블을 기준으로 파드를 관리하기 때문에, 레플리카세트 실행 중 설정을 변경해서 관리하는 파드를 수정할 수 있다. 
초기 구성을 위해서는 `.spec.template.metadata.labels` 와 `.spec.selector.matchLabels` 가 같아야 한다. 
만약 `.spec.selector` 에 별도의 설정이 없을 경우 `.spec.template.metadata.labels.app` 의 내용을 기본값으로 가용한다. 

구성한 레플리카세트 템플릿은 `kubectl apply -f replicaset-nginx.yaml` 명령을 통해 실행 할 수 있다. 

```bash
$ kubectl apply -f replicaset-nginx.yaml
replicaset.apps/replicaset-nginx created
$ kubectl get pods
NAME                     READY   STATUS              RESTARTS   AGE
replicaset-nginx-8dtrt   0/1     ContainerCreating   0          6s
replicaset-nginx-stgj2   0/1     ContainerCreating   0          6s
replicaset-nginx-t8t46   1/1     Running             0          6s
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-8dtrt   1/1     Running   0          29s
replicaset-nginx-stgj2   1/1     Running   0          29s
replicaset-nginx-t8t46   1/1     Running   0          29s
```  

`replicaset-nginx-` 라는 프리픽스로 3개의 파드가 생성된 것을 확인 할 수 있다. 
`kubectl delete pod <파드이름>` 을 통해 파드를 삭제하면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ kubectl delete p
od replicaset-nginx-t8t46
pod "replicaset-nginx-t8t46" deleted
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-8dtrt   1/1     Running   0          2m15s
replicaset-nginx-r2b64   1/1     Running   0          7s
replicaset-nginx-stgj2   1/1     Running   0          2m15s
```  

파드 1개를 삭제 했을 때, 레플리카세트에서 새로운 파드 `replicaset-nginx-r2b64` 를 실행하는 것을 확인 할 수 있다.  

템플릿에서 `.sepc.replicas` 값을 4로 수정하고 나서 다시 적용하면 아래와 같은 결과를 확인 할 수 있다. 

```bash
# 수정
spec:
  replicas: 4


$ kubectl apply -f
 replicaset-nginx.yaml
replicaset.apps/replicaset-nginx configured
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-8dtrt   1/1     Running   0          5m45s
replicaset-nginx-fk2gd   1/1     Running   0          6s
replicaset-nginx-r2b64   1/1     Running   0          3m37s
replicaset-nginx-stgj2   1/1     Running   0          5m45s
```  

### 레플리카세트와 파드의 느슨한 관계
앞서 언급한 것처럼 레플리카세트는 파드를 레이블 기준으로 관리하므로, 
이는 느슨한 관계이다. 
레플리카세트와 파드를 함께 삭제하기 위해서는 `kubectl delete replicaset <레플리카세트 이름>` 명령어로 가능하다. 
명령어에서 `--cascade=false` 옵션을 사용하면 파드에는 영향을 주지 않고 레플리카세트만 삭제하게 된다.  

`kubectl delete replicaset <레플리카세트 이름> --cascade=false` 를 사용하고 조회를 수행하면,
아래와 같이 레플리카세트는 삭제되지만 관리되면 파드들은 그대로인 것을 확인 할 수 있다. 

```bash
$ kubectl delete replicaset replicaset-nginx --cascade=false
replicaset.apps "replicaset-nginx" deleted
$ kubectl get replicaset,pods
NAME                         READY   STATUS    RESTARTS   AGE
pod/replicaset-nginx-8dtrt   1/1     Running   0          13m
pod/replicaset-nginx-fk2gd   1/1     Running   0          7m27s
pod/replicaset-nginx-r2b64   1/1     Running   0          10m
pod/replicaset-nginx-stgj2   1/1     Running   0          13m
```  

`kubectl apply -f replicaset-nginx.yaml` 을 통해 다시 레플리카세트를 생성하면, 
아래와 같이 생성된 레플라카세트에서 기존의 파드를 관리하는 것을 확인 할 수 있다. 
레플리카세트에 대한 조회는 `kubectl get replicaset` 혹은 `kubectl get rs` 로 가능하다. 

```bash
$ kubectl apply -f
 replicaset-nginx.yaml
replicaset.apps/replicaset-nginx created
$ kubectl get replicaset,pods
NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/replicaset-nginx   4         4         4       5s

NAME                         READY   STATUS    RESTARTS   AGE
pod/replicaset-nginx-8dtrt   1/1     Running   0          15m
pod/replicaset-nginx-fk2gd   1/1     Running   0          9m23s
pod/replicaset-nginx-r2b64   1/1     Running   0          12m
pod/replicaset-nginx-stgj2   1/1     Running   0          15m
```  

레플리카세트 조회 필드에서 `DESIRED` 와 `CURRENT` 의 의미는 아래와 같다.
- `DESIRED` : 레플리카세트 설정에 지정한 파드 개수
- `CURRENT` : 레플리카세트에서 관리하는 클러스터에 실행된 실제 파드 수

레플리카세트의 정상동작 여부확인을 위해 다시 파드 하나를 삭제하게 되면, 
`replicas` 개수를 유지하기 위해 새로운 파드를 실행하는 것을 확인 할 수 있다. 

```bash
$ kubectl delete pods replicaset-nginx-stgj2
pod "replicaset-nginx-stgj2" deleted
$ kubectl get replicaset,pods
NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/replicaset-nginx   4         4         4       4m38s

NAME                         READY   STATUS    RESTARTS   AGE
pod/replicaset-nginx-8dtrt   1/1     Running   0          19m
pod/replicaset-nginx-fk2gd   1/1     Running   0          13m
pod/replicaset-nginx-r2b64   1/1     Running   0          17m
pod/replicaset-nginx-z7kmp   1/1     Running   0          8s
```  

실행 중인 파드의 `.metadata.labels.app` 필드를 레플리카세트와 다른 이름으로 변경했을 때의 상황을 살펴본다. 
변경을 위해 실행 중인 파드 목록 중, `kubectl edit pod <파드이름>` 명령을 실행하면 편집기가 실행되고 설정을 수정할 수 있다. 
기존 `replicasset-nginx` 에서 `replicaset-nginx-2` 로 변경한다. 

```bash
$ kubectl edit pod replicaset-nginx-z7kmp

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-07-02T18:42:54Z"
  generateName: replicaset-nginx-
  labels:
    # 변경
    app: replicaset-nginx-2
  name: replicaset-nginx-z7kmp
  namespace: default

.. 생략 ..


pod/replicaset-nginx-z7kmp edited
```  

`kubectl get pods` 명령을 통해 실행 중인 파드를 확인하면 아래와 같이 새로운 파드가 실행 된 것을 확인 할 수 있다. 

```
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-6bhv8   1/1     Running   0          52s # 새로운 파드 추가
replicaset-nginx-8dtrt   1/1     Running   0          24m
replicaset-nginx-fk2gd   1/1     Running   0          18m
replicaset-nginx-r2b64   1/1     Running   0          22m
replicaset-nginx-z7kmp   1/1     Running   0          4m44s
```  

실행 중인 파드의 `.metadata.labels.app` 필드 설정을 확인하기 위해,
`kubectl get pods -o=jsonpath="{range   .items[*]}{.metadata.name}{'\t'}{.metadata.labels}{'\n'}{end}"` 실행하면 아래와 같다. 

```bash
$ kubectl get pods -o=jsonpath="{range   .items[*]}{.metadata.name}{'\t'}{.metadata.labels}{'\n'}{end}"
replicaset-nginx-6bhv8  map[app:replicaset-nginx]
replicaset-nginx-8dtrt  map[app:replicaset-nginx]
replicaset-nginx-fk2gd  map[app:replicaset-nginx]
replicaset-nginx-r2b64  map[app:replicaset-nginx]
replicaset-nginx-z7kmp  map[app:replicaset-nginx-2]
```  

`.metadata.labels.app` 을 변경한 `replicaset-nginx-6bhv8` 파드만 `replicaset-nginx-2` 인 것을 확인 할 수 있다. 
기존 `replicaset-nginx` 에서는 파드 한개가 자신이 관리하는 레이블에서 빠졌기 때문에 `replicas` 유지를 위해 하나를 더 실행 한 것이다. 

>추가로 `kubectl get pod -o=jsonpath` 를 사용하면 실행 중인 파드의 전체 내용중 원하는 내용만 확인 할 수 있다. 


---
## Reference
