--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 파드 스케쥴링 (노트셀렉터, 어피니티/안티어피니티)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에 구성된 노드에서 파드 스케쥴링을 설정할 수 있는 노드셀력터와 어피니티/안티어피니티에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Pod Scheduling
  - NodeSelector
  - Affinity
  - Anti Affinity
toc: true
use_math: true
---  

## 파드 스케쥴링
쿠버네티스 클러스터에 구성된 노드에 어떤 파드를 실행 할지를 설정할 수 있는 다양한 옵션이 있다. 
옵션을 적절하게 사용하면 구성하고 싶은 구조로 파드를 배치할 수 있다. 
특정 파드를 특정 노드에만 실행 시키거나, 
특정 IP 대역 노드에만 실행하거나, 
특정 파드가 같은 노드에서 실행되지 못하게 할 수도 있다. 
또한 특정 노드에서 실행중인 파드를 한번에 다른 노드로 옮겨 실행하는 설정도 가능하다.  

## 노드셀렉터
파드 스케쥴링에서 기본적인 옵션인 노드셀력터(`NodeSelector`) 는 노드를 선택해서 파드를 실행하는 기능이다. 
파드가 클러스터에 구성된 노드들 중 어떤 노드에 실행 될지를 `key-value` 구조로 설정 할 수 있다.  

노드셀렉터는 노드의 레이블에 설정된 값으로 노드를 선택한다. 
`kubectl get nodes --show-labels` 명령으로 현재 노드에 어떤 레이블 값이 설정 됐는지 확인하면 아래와 같다. 

```bash
$ kubectl get nodes --show-labels
NAME             STATUS   ROLES    AGE   VERSION   LABELS
docker-desktop   Ready    master   35d   v1.15.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=docker-desktop,kubernetes.io/os=linux,node-role.kubernetes.io/master=
```  

주의깊게 봐야할  필드는 `LABELS` 필드이다. 
`beta.kubernetes.io/arch` 는 노드의 아키텍쳐가 `amd64` 임을 의미 하고, 
`beta.kubernetes.io/os` 는 노드의 운영체제가 `linux` 임을 의미 하고,  
`kubernetes.io/hostname` 는 노드의 호스트 이름이  `docker-desktop` 임을 의미 하고, 
`node-role.kubernetes.io/master=` 는 노드가 현재 마스터 노드임을 의미한다.  

노드에 레이블을 추가하는 명령어는 `kubectl label nodess docker-desktop key=value` 로 가능 하다. 
해당 명령어를 사용해서 `disktype=ssd` 를 추가하고 다시 확인하면 아래와 같다. 

```bash
$ kubectl label nodes docker-desktop disktype=ssd
node/docker-desktop labeled
$ kubectl get nodes --show-labels
NAME             STATUS   ROLES    AGE   VERSION   LABELS
docker-desktop   Ready    master   35d   v1.15.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=docker-desktop,kubernetes.io/os=linux,node-role.kubernetes.io/master=
```  

새로 추가한 `disktype=ssd` 가 정상적으로 출력되는 것을 확인 할 수 있다.  

아래 템플릿은 노드 셀렉터 옵션을 추가한 파드 템플릿의 예시이다. 

```yaml
# pod-nodeselector.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
spec:
  containers:
    - name: pod-nodeselector
      image: nginx:latest
      ports:
        - containerPort: 80
  nodeSelector:
    disktype: hdd
```  

- `.sepc.nodeSelector` : 파드 템플릿에서 노드셀렉터를 설정하는 필드로, `disktype=hadd` 을 설정 했으므로 현재 파드가 실행 가능한 노드는 없다. 

`kubectl apply -f pod-nodeselector.yaml` 명령으로 클러스터에 적용하고 실행 중인 파드를 조회하면 아래와 같다. 

```bash
$ kubectl apply -f pod-nodeselector.yaml
pod/pod-nodeselector created
$ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
pod-nodeselector   0/1     Pending   0          7s
$ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
pod-nodeselector   0/1     Pending   0          32s
```  

적용한 파드가 실행되지 못하고 계속 `Pending` 상태인 것을 확인 할 수 있다. 
`diskType=hdd` 에 해당하는 노드가 없어, 해당 파드는 실행되지 못하는 상황이다.  

해당 파드를 삭제하고, `.spec.nodeSelector.disktype` 값을 `ssd` 로 수정하고 다시 적용해서 파드를 조회하면 아래와 같다. 

```bash
$ kubectl delete pod pod-nodeselector
pod "pod-nodeselector" deleted
$ vi pod-nodeselector

.. 생략 ..
spec:
  .. 생략 ..
  nodeSelector:
    disktype: ssd  

$ kubectl apply -f pod-nodeselector.yaml
pod/pod-nodeselector created
ckdtj@CS-SurfacePro6 MINGW64 ~/Documents/kubernetes-example/pod-scheduling/nodeselector (master)
$ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
pod-nodeselector   1/1     Running   0          10s
```  

현재 노드 레이블 중 `disktype=ssd` 값이 있기 때문에 파드가 정상적으로 실행 되는 것을 확인 할 수 있다. 

## 어피니티, 안티 어피티니
어니피티(`Affinity`) 는 파드를 하나의 노드에 실행되도록 하고, 
안티 어피니티(`Anti Affinity`)` 는 파드가 같은 노드에서 실행 되지 못하도록 하는 설정이다. 

### 노드 어피니티
노드 어피니티(`Node Affinity`) 는 앞서 살펴본 노드셀렉터와 비슷한 방식인 레이블 기반으로 파드를 스케쥴링한다. 
노드 어피니티는 노드셀렉터와 같이 설정 할 수 있고, 함께 사용할 경우 두 조건을 모두 만족한 경우에만 파드가 실행된다.  

노드 어피니티의 설정 필드는 아래 2가지가 있다. 
- `requireDuringScheduingIgnoredDuringExecution` : 파드 스케쥴링 할때 필수로 필요한 조건을 설정하는 필드이다. 
- `preferredDuringSchedulingIgnoredDuringExecution` : 파드 스케쥴링 할때 선택적인 의미로 만족하면 좋은 조건을 설정하는 필드이다. 

노드 어피니티는 파드 스케쥴링이 실행 중인 상황에서는 해당 필드의 값이 변경되더라도, 
클러스터에 적용시점에 설정된 값으로 계속해서 스케쥴링이 수행된다.  

아래는 노드 어피니티를 적용한 파드 템플릿의 예시이다. 

```yaml
# pod-node-affinity.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-node-affinity
spec:
  containers:
    - name: pod-node-affinity
      image: nginx:latest
      ports:
        - containerPort: 80
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: beta.kubernetes.io/os
                operator: In
                values:
                  - linux
                  - windows
              - key: disktype
                operator: Exists
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 10
          preference:
            matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                  - worker-node-1
```  

`.spec.affinity.nodeAffinity` 가 노드 어피니티에 대한 조건을 설정하는 필드이다. 
`.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution` 에서는 
노드 어피니티와 노드셀렉터 설정을 연결하는 `.nodeSelectorTerms[].matchExpressions[]` 필드를 사용해서 `.key`, `.operator`, `values[]` 를 사용해 필수 조건에 대한 내용이 있다. 
여기서 `operator` 로 사용할 수 있는 종류는 아래와 같다. 
- `In` : `.values[]` 필드의 값 중 레이블과 일치하는 값이 하나이상 존재하는지 판별한다.
- `NotIn` : `In` 과 반대로 `.values[]` 필드의 모든 값이 레이블과 일치하지 않는지 판별 한다. 
- `Exists` : `.values[]` 필드는 사용하지 않고, 레이블의 `key` 가 있는지 판별 한다. 
- `DoesNotExists` : 레이블의 `key` 가 존재하지 않는지 판별한다. 
- `Gt` : `Greater than` 의 의미로 `.values[]` 의 값이 하나이면서 숫자이여 하고, 
값보다 레이블의 값이 큰지 판별 한다. 
- `Lt` : `Lower than`의 의미로 값 보다 레이블의 값이 작은지 판별한다. 

작성한 템플릿을 `kubectl apply -f pod-node-affinity.yaml` 명령으로 클러스터에 적용하고, 
파드를 조회하면 아래와 같이 정상적으로 실행되는 것을 확인 할 수 있다. 

```bash
$ kubectl apply -f pod-node-affinity.yaml
pod/pod-node-affinity created
$ kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
pod-node-affinity   1/1     Running   0          5s
```  

`Gt` 테스트를 위해 먼저 노드 레이블에 `core=8` 을 추가한다. 

```bash
$ kubectl label nodes docker-desktop core=8
node/docker-desktop labeled
$ kubectl get nodes --show-labels
NAME             STATUS   ROLES    AGE   VERSION   LABELS
docker-desktop   Ready    master   35d   v1.15.5   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,core=8,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=docker-desktop,kubernetes.io/os=linux,node-role.kubernetes.io/master=
```  

그리고 템플릿의 노트 어피니티에 `core` 가 4보다 큰 조건을 추가하고 파드를 삭제 했다가 다시 적용한다. 

```bash
vi pod-node-affinity.yaml

.. 생략 ..
spec:
  .. 생략 ..
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
             .. 생략 ..
              - key: core
                operator: Gt
                values:
                  - "4"

$ kubectl delete pod pod-node-affinity
pod "pod-node-affinity" deleted
$ kubectl apply -f pod-node-affinity.yaml
pod/pod-node-affinity created
$ kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
pod-node-affinity   1/1     Running   0          6s
```  

노드에 설정된 레이블은 `core=8` 이기 때문에 정상적으로 실행된다. 
템플릿에서 `core` 의 `.values[]` 값을 10으로 수정하고 다시 적용하면 아래와 같이 실행 되지 않는 것을 확인 할 수 있다. 

```bash
vi pod-node-affinity.yaml

.. 생략 ..
spec:
  .. 생략 ..
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
             .. 생략 ..
              - key: core
                operator: Gt
                values:
                  - "10"

$ kubectl delete pod pod-node-affinity
pod "pod-node-affinity" deleted
$ kubectl apply -f pod-node-affinity.yaml
pod/pod-node-affinity created
$ kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
pod-node-affinity   0/1     Pending   0          15s
```  

`.preferredDuringSchedulingIgnoredDuringExecution` 필드는 `.weight` 필드를 사용한다는 점과 
`.nodeSelectorTerms` 를 사용하지 않고 `.perference` 필드를 사용한다는 점을 제외하곤 먼저 살펴 봤던 `.requiredDuringSchedulingIgnoredDuringExcecution` 과 비슷하다. 
말그대로 제시되는 조건을 선호한다는 정도의 의미로 사용된다. 
조건을 만족하지 않더라도 노드에 적절하게 파드를 스케쥴링 하게 된다.  

`.weight` 필드는 `1~100` 값을 사용할 수 있다. 
`.matchExpressions[]` 에 나열된 조건이 만족할 때마다 해당하는 `.weight` 를 합산해, 
합산한 값이 가장 큰 노드에 파드를 스케쥴링 하게 된다.  

### 파드의 어피니티와 안티 어피니티
파드에서 어피니티(`Affinity`)와 안티 어피니티(`Anti Affinity`) 는 디플로이먼트, 스테이트풀세트로 파드를 배포 했을 때, 
파드와 파드 사이(`inter-pod`) 관계를 정의할 수 있다.  

실제로 여러 노드에서 서비스를 구성하다보면 특정 서비스들 간에 내부 통신이 빈번하게 발생해서 같은 노드에 두고 싶은 경우가 있을 수 있다. 
이런 상황에서 어피니티를 사용해서 같은 노드에서 파드를 실행 시키고, 
쿠버네티스 클러스터를 구성할 때 리눅스의 `BPF(Berkeley Packet Filter)`, `XD(PeXpress Data Path)` 를 이용하는 실리엄(`cilium`) 
네트워크 플러그인을 이용해 같은 노드에서 내부 통신의 성능을 더욱 끌어 올릴 수 있다.  

안티 어피니티는 특정 파드가 노드의 자원을 많이 사용해야 하는 경우, 이런 파드를 여러 노드로 분산 할 때 사용 할 수 있다. 
이렇게 안티 어피니티를 사용하지 않은 경우, 
노드의 자원이 부족해 노드를 추가했지만 자원을 많이 사용하는 파드가 같은 노드에 있어서 문제는 해결되지 않을 수 있다.  

아래는 안티 어피니티를 적용한 `Redis` 파드를 배포하는 디플로이먼트 템플릿의 예시이다. 

```yaml
# pod-anti-affinity.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-redis
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - store
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: redis-server
          image: redis:alpine
```  

파드의 어피니티와 안티 어피니티는 `.spec.template.spec.affinity` 하위에 `.podAffinity`, `.podAntiAffinity` 로 설정 할 수 있다. 
이후 조건 작성은 앞서 살펴본 노드 어피니티와 대부분 동일하다. 
`.operator` 필드에서 사용 가능한 값은 아래와 같다.
- `In`
- `NotIn`
- `Exists`
- `NotExists`

`.topologyKey` 필드에 대해서는 추후에 어피니티 설정 소개후 다루도록 한다.  

- `.spec.template.metadata.labels` : `app` 필드에 `store` 값을 설정했다. 
- `.spec.template.spec.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution.labelSelector[].matchExpressions[]` : `.key` 로 `app` 을 설정 하고, 
`.values[]` 에는 `store` 를 설정해서, 
`app` 필드의 값이 `store` 인 파드는 같은 노드에서 실행되지 않도록 한다. 

`kubectl apply -f pod-anti-affinity.yaml` 명령으로 클러스터에 적용하고, 
실행 중인 파드를 조회하면 아래와 같이 2개의 파드 중 1개만 실행 중인 것을 확인 할 수 있다. 

```bash
$ kubectl apply -f pod-anti-affinity.yaml
deployment.apps/deploy-redis created
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
deploy-redis-84c5bc778-qjdw4   0/1     Pending   0          15s   <none>      <none>           <none>           <none>
deploy-redis-84c5bc778-s8q4w   1/1     Running   0          15s   10.1.0.27   docker-desktop   <none>           <none>
```  

만약 클러스터에 구성된 노드가 2개 이상이였다면, 모든 파드가 실행될 수 있다.  

이어서 아래는 파드 어피니티를 적용한 디플로이먼트 템플릿의 예시이다. 

```yaml
# pod-affinity.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - web-store
              topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - store
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: web-app
          image: nginx:alpine
```  

- `.spec.template.spec.affinity.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[].labelSelector.matchExpressions[]` : 안티 어피니티 설정은 파드의 레이블 중 `app` 필드가 `webs-store` 인 조건으로 설정했다. 
해당 조건으로 인해 `Nginx` 파드는 같은 노드에서 실행 될 수 없다. 
- `.spec.template.spec.affinity.podAffinity.requiredDuringSchedulingIgnoredDuringExecution[].labelSelector.matchExpressions[]` : 어피니티 설정은 파드의 레이블 중 `app` 필드가 `store` 인 조건으로 설정했다. 
해당 조건으로 `Redis` 파드와 `Nginx` 파드는 같은 노드에서 실행 된다. 
- `.spec.template.spec.pod*Affinity.requiredDuringSchedulingIgnoredDuringExecution.topologyKey` : 노드의 레이블을 사용해서 파드의 어피니티와 안티 어피니티를 설정 할 수 있는 기준으로 사용되는 필드이다. 
쿠버네티스는 파드를 스케쥴링할 때 파드의 레이블을 기준으로 대상 노드를 찾고, 
`.topologyKey` 필드의 값을 기준으로 해당 노드가 실제 조건에 부합하는 노드인지 확인 한다. 
현재 `.topologyKey` 에 설정된 `kubnerntes.io/hostname` 은 `hostname` 을 기준으로 어피니티 설정을 만족하면 같은 노드에 실행 하고, 
안티 어피니티 설정을 만족하면 `hostname` 을 기준으로 다른 노드에 파드를 실행 하게 된다. 

실제 서비스를 구성하게 되면 스케쥴링 기준에는 노드 뿐만아니라, 
`IDC` 에서 같은 랙인지, `IDC` 의 종류도 포함될 수 있다. 
`.topologyKey` 필드 설정에 있어서 성능이나 보안상 이유로 아래와 같은 제약 사항들이 있다. 
- `.podAffinity` 의 하위 필드와 `.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[]` 하위 필드에는 `.topologyKey` 는 필수로 설정해야 한다. 
- `.podAntiAffinity.requiredDuringSchedulingIgnoredDuringExecution[]` 하위와 어드미션 컨트롤러의 `.LimitPodHardAntiAffinityTopology` 하위에 
설정하는 `.topologyKey` 필드에는 `kubernetes.io/hostname` 만 설정 가능하다. 
사용자 정의 값을 사용하기 위해서는 어드미션 컨트롤러 설정을 변경해야 가능하다. 
- `.podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution` 하위 필드에는 `.topologyKey` 필드가 필수가 아니다. 
설정되지 않은 경우 전체 토폴로지를 대상으로 안티 어피니티 설정이 만족하는지 확인한다. 
여기서 전체 토폴로지는 `kubernetes.io/hostname`, `failure-domain.beta.kubernetes.io.zone`, `failure-domain.beta.kubernetes.io/region` 이 있다. 

작성한 템플릿을 `kubectl apply -f pod-affinity.yaml` 명령으로 클러스터에 적용하고, 
실행 중인 파드를 조회하면 아래와 같다. 

```bash
$ kubectl apply -f pod-affinity.yaml
deployment.apps/web-server created
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
deploy-redis-84c5bc778-qjdw4   0/1     Pending   0          36m   <none>      <none>           <none>           <none>
deploy-redis-84c5bc778-s8q4w   1/1     Running   0          36m   10.1.0.27   docker-desktop   <none>           <none>
web-server-7c88794f9c-clvg8    1/1     Running   0          9s    10.1.0.29   docker-desktop   <none>           <none>
web-server-7c88794f9c-fphxj    0/1     Pending   0          9s    <none>      <none>           <none>           <none>
```  

현재 노드에서는 `Nginx`, `Redis` 파드가 각각 1개씩만 실행 된 것을 확인 할 수 있다. 
그외 다른파드는 `Pending` 상태이다. 

---
## Reference
