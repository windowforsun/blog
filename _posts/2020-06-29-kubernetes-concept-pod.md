--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 파드(Pod)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 컨테이너를 관리하는 기본 단위인 파드에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Pod
toc: true
use_math: true
---  

## 파트
쿠버네티스에서 파드(`Pod`)는 컨테이너 관리의 기본 단위이다. 
파드라는 단위로 컨테이너를 묶어 관리하기 때문에, 
하나의 컨테이너가 포함되는 것이아니라 여러 컨테이너로 구성된다. 
물론 하나의 파드에 하나의 컨테이너만 구성하는 방식도 가능하다.  

파드에 여러 컨테이너를 구성해서 컨테이너 마다 각 역할을 부여할 수 있다. 
그리고 파드에 혹한 컨테이어는 동일한 노드에서 실행된다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_cluster_pod_1.png)

위 그림은 파드 구성의 예이다. 
파드 하나에 있는 컨테이너들은 동일한 `IP` 를 공유해서 사용하기 때문에, 
파드나 파드내의 컨테이너에 접근할 때는 하나의 `IP` 사용하고 포트를 달리해서 가능하다. 

```yaml
# simple-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  labels:
    app: simple-pod
spec:
  containers:
    - name: simple-pod
      image: nginx:latest
      ports:
      - containerPort: 8080
```  

위 `YAML` 템플릿은 파드를 구성하는 템플릿의 예시이다.
- `.metadata.name` : 파드의 이름을 설정한다. 
- `.metadata.labels.app` : 파드 오브젝트를 식별하는 레이블을 설정한다. 
- `.spec.continaers[].name` : 파드에 구성되는 컨테이너 이름을 설정한다. 하나 이상의 컨테이너를 구성 할 수 있다. 
- `.spec.containers[].image` : 파드에 구성되는 컨테이너가 사용하는 이미지를 설정한다. 
- `.spec.containers[].ports[].containerPort` : 해당하는 컨테이너에서 사용하는 포트를 설정한다. 

`kubectl apply -f simple-pod.yaml` 명령을 통해 파드 템플릿을 클러스터에 적용 할 수 있다.

```bash
$ kubectl apply -f simple-pod.yaml
pod/simple-pod created
```  

`kubectl get pods` 명령 수행 결과에서 `STATUS` 필드가 `Running` 일 경우 파드가 정상적으로 실행 된 것이다. 

```bash
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
simple-pod   1/1     Running   0          7s
```  

## 파드의 생명주기
파드는 생성을 시작으로 삭제까지 생명주기(`Lifecycle`) 이 있다. 
- `Pending` : 파드 생성중을 의미한다. 
컨테이너 이미지를 다운로드하고 파드에 구성된 컨테이너를 실행하는 중을 의미한다. 
- `Running` : 파드에 구성된 컨테이너가 실행 중인 상태를 의미한다. 
1개 이상의 컨테이너가 실행 중 혹은 재시작 상태일 수 있다. 
- `Succeeded` : 파드에 구성된 컨테이너가 정상적으로 실행 종료된 상태로 재시작되지 않는다. 
- `Failed` : 파드에 구성된 컨테이너 중 정상 종료가 되지 않은 컨테이너가 있는 상태이다. 
종료 코드가 0이 아니거나, 시스템에서 직접 종료한 상황이다. 

현재 실행 중인 파드의 생명주기는 `kubectl describe pods` 명령을 통해 확인 가능하다. 

```bash
$ kubectl describe pods simple-pod
Name:         simple-pod
Namespace:    default
Priority:     0
Node:         docker-desktop/192.168.65.3
Start Time:   Wed, 01 Jul 2020 02:17:30 +0900
Labels:       app=simple-pod
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"app":"simple-pod"},"name":"simple-pod","namespace":"default"},"spe...
# 파드 상태
Status:       Running   
IP:           10.1.0.8

.. 생략 ..

# 파드 상태 정보
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True

.. 생략 ..
```  

`Status` 필드는 현재 파드의 상태를 의미하고, 
`Conditions` 는 현재 파드의 상태 정보를 의미하는데 `Type` 필드는 아래와 같은 종류가 있다. 
- `Initialized` : 모든 초기화 컨테이너의 실행이 성공적으로 완료 되었는지
- `Ready` : 파드가 요청을 실행할 수 있고, 연결된 모든 서비스의 로드밸런싱 풀에 추가되었는지
- `ContainersReady` : 파드에 구성된 컨테이너가 준비 상태인지
- `PodScheduled` : 파드가 하나의 노드로 스케쥴 완료 되었는지
- `Unschedulable` : 스케쥴러에서 자원 부족 혹은 특정 상황으로 인해 파드를 스케쥴 할수 없는지

`Status` 필드는 아래와 같은 종류가 있다.
- `True` 활성화
- `False` 비활성화
- `Unknown` 상태를 알 수 없음

실행된 `simple-pod` 는 `kubectl delete pod simple-pod` 명령을 통해 종료 가능하다. 

```bash
$ kubectl delete pod simple-pod
pod "simple-pod" deleted
```  

## kubelet 을 통한 컨테이너 진단
파드에 컨테이너가 실행되면 `kubelet` 이 컨테이너를 주기적으로 진단한다. 
이는 프로브(`Probe`)를 기준으로 수행되는데 그 종류는 아래와 같다. 
- `livenessProbe` : 컨테이너가 실행 됐는지 확인한다. 
진단이 실패하면 `kubelet` 은 컨테이너를 종료하고 재시작 정책에 따라 컨테이너를 재시작 한다. 
기본 값은 `Success` 이다. 
- `readinessProbe` : 컨테이너 실행 후 서비스 요청을 수행할 수 있는지 진단한다. 
진단 실패하면 엔드포인트 컨트롤러(`endpoint controller`)는 해당 파드에 연결된 모든 서비스를 엔드포인트에서 제거한다. 
해당 프로브가 설정되었다면 기본 값은 `Failure` 이고, 설정되지 않았다면 기본 값은 `Success` 이다. 

쿠버네티스 클러스터에서는 프로브를 통해 해당 파드를 관리하고 서비스 가능 여부를 판별한다. 
`readinessProbe` 를 통해 트래픽 처리 여부를 판별하기 때문에,
 지원하는 컨테이너에서 실패하게 되면 트래픽을 받지 않게 된다. 
 
 컨테이너에서는 핸들러(`handler`) 를 구현 및 설정하게 되면 `kubelet` 은 이를 호출해 실행하게 된다. 
 - `ExecAction` : 컨테이너에 지정된 명령을 실행하고 종료 코드가 0 일때, `Success` 라고 진단한다. 
 - `TCPSocketAction` : 컨테이너에 지정된 `IP`, `PORT` 로 TCP 상태를 확인하고 포트가 열려 있으면 `Success` 로 진단한다. 
 - `HTTPGetAction` : 컨테이너에 지정된 `IP`, `PORT` 로 `HTTP GET` 요청을 보내 응답 코드가 200 ~ 400 사이일 경우 `Success`로 진단한다. 
 
 진단 결과에는 아래와 같은 종류가 있다. 
 - `Success` : 컨테이너 진단 성공
 - `Failure` : 컨테이너 진단 실패
 - `Unknown` : 진단 수행 자체가 실패해서 컨테이너 상태 알 수 없음

## 초기화 컨테이너
파드에서 초기화 컨테이너(`init container`) 는 앱 컨테이너(`app container`) 가 실행되기 전 파드를 초기화 하는 역할을 수행한다. 
초기화 컨테이너의 특징은 아래와 같다. 
- 초기화 컨테이너는 여러 개로 구성이 가능하다. 파드 템플릿에서 작성된 순서대로 실행된다.
- 초기화 컨테이너는 실행 실패할 경우 성공 할때까지 재시작한다. 이를 통해 명령을 순서대로 실행하는 등의 `선언적` 에서 벗어난 구성이 가능하다.
- 초기화 컨테이너가 순서대로 모두 실행 된 후, 앱 컨테이너를 실행한다. 

초기화 컨테이너를 활용하면 피드의 실행을 특정 조건이 만족 할때까지 지연 시킬 수 있다. 
그리고 초기화 컨테이너는 `readinessProbe` 를 지원하지 않는다. 

```yaml
# simple-init-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  labels:
    app: simple-pod
spec:
  initContainers:
    - name: init-a-service
      image: nginx:latest
      command: ['sh', '-c', 'sleep 2; echo init a service;']
    - name: init-db
      image: nginx:latest
      command: ['sh', '-c', 'sleep 2; echo init db;']
  containers:
    - name: simple-pod
      image: nginx:latest
      command: ['sh', '-c', 'echo init app running && sleep 3600']
```  

위는 초기화 컨테이너의 예시 파드 템플릿이다. 
초기화 컨테이너는 `.spec.initContainers[]` 의 하위에 설정 가능하다. 
- `.spec.initContainers[0]` : 2초 슬립 후, `init a service` 라는 메시지를 출력한다. 
- `.spec.initContainers[1]` : 2초 슬립 후, `init db` 를 출력한다. 
- `.spec.containers[0]` : 앱 컨테이너의 설정으로 `init app running` 이라는 메시지를 출력하고 3600초 간 슬립한다. 

## 파드 인프라 컨테이너
쿠버네티스 클러스터에 구성되는 모든 파드에는 파드 인프라 컨테이너(`Pod infrastructure container`) 라는 `pause` 컨테이너가 있다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_cluster_pod_2.png)

`pause` 는 파드의 기본 네트워크로 실행된다. 
그리고 프로세스의 `PID` 가 1로 설정되는 파드내 모든 컨테이너의 부모 컨테이너 역할을 한다. 
파드에 구성되는 모든 컨테이너는 `pause` 가 제공하는 네트워크를 공유해서 사용한다. 
이러한 특성은 파드내 컨테이너를 재시작 할 경우 `IP` 는 계속해서 유지 가능하지만, 
파드를 시작하게 되면 파드안 모든 컨테이너가 재시작 되기 때문에 `IP` 가 변경 될 수 있다. 

`pause` 컨테이너가 아닌 다른 컨테이너를 파드 인프라 컨테이너로 지정할 때는 `kubelet` 명령 옵션인 `--pod-infra-container-image` 를 사용한다. 

## 스태틱 파드
쿠버네티스 클러스터에서 파드는 `kube-apiserver` 를 통해 실행하는데, 
스새틱 파드(`static pod`)는 `kube-apiserver` 를 통하지 않고 `kubelet` 이 직접 실행하는 파드이다.
`kubelet` 옵션에서 `--pod-manifest-path` 옵션으로 지정된 디렉토리에 스태틱 파드의 템플릿을 추가하면,
`kubelet` 에서는 스태틱 하드로 실행하게 된다.  

스태틱 파드는 `kubelet` 에서 직접 관리하면서 이상이 생길 경우 재시작을 수행한다. 
그러므로 `kubelet` 이 실행 중인 노드에서만 실행이 된다. 
`kube-apiserver` 로 조회는 가능하지만, 명령실행은 불가능하다.  

기본적으로 스태틱 파드는 `kube-apiserver`, `etcd` 와 같은 시스템 파드를 실행하는 용도로 사용된다.  

스태틱 파드의 경로는 `/etc/kubernetes/manifest` 이다. 
경로 하위에는 쿠버네티스 시스템용 파드 템플릿이 위치해 있다. 
해당 템플릿을 수정해서 저장하게 되면 `kubelet` 이 감지하고 이를 재시작하게 된다. 

## 파드 자원 할당
쿠버네티스 클러스터는 마이크로서비스 아키텍쳐 기반으로 여러 노드에 걸쳐 여러 파드들이 실행 된다. 
이런 상황에서 하나의 노드에 자원을 많이 소모하는 파드가 몰려 있거나 한다면 전체적인 클러스터의 효율이 낮아지게 된다.  

쿠버네티스에서는 이런 상황의 대비책으로 `CPU`, `Memory` 의 사용할지를 명시해서 관리한다. 
파드 템플릿에서 이를 관리하는 필드는 아래와 같다. 
- `.spec.containers[].reousrces.limits.cpu` : 최대 `CPU` 사용량
- `.spec.containers[].reousrces.limits.memory` : 최대 `Memory` 사용량
- `.spec.containers[].reousrces.requests.cpu` : 최소 `CPU` 사용량
- `.spec.containers[].reousrces.requests.memory` : 최소 `Memory` 사용량

`CPU` 사용량 관련 설정은 아래와 같은 특징을 갖는다. 
- 설정하는 사용량의 단위는 `코어` 이다. 
- 1, 2 와 같은 정수도 가능하고, 0.1, 0.2 와 같은 소수로 설정도 가능하다. 
- 1은 1개의 코어를 모두 사용한다는 의미이고, 0.1 과 같은 소수점은 1개 코어를 100으로 했을 때의 10만큼 사용한다는 의미이다. 

`Memory` 사용량 관련 설정은 아래와 같은 특징을 갖는다. 
- 단위를 명시하지 않으면 바이트 단위를 사용한다. 
- 사용 가능 한 십진법 접두어는 `E`, `P`, `T`, `G`, `M`, `K` 가 있다.
- 사용 가능 한 이진법 접두어는 `Ei`, `Pi`, `Ti`, `Gi`, `Mi`, `Ki` 가 있다. 

```yaml
# simple-resource-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  labels:
    app: simple-pod
spec:
  containers:
    - name: simple-pod
      image: nginx:latest
      ports:
      - containerPort: 8080
      resources:
        requests:
          cpu: 0.1
          memory: 100M
        limits:
          cpu: 0.5
          memory: 500M
```  

위 파드 템플릿은 `.spec.containers[].resources` 필드를 사용해서 자원을 할당한 예시이다. 
`.spec.containers[].resources.requests` 는 최소 요구하는 자원을 의미하기 때문에 만족하는 노드에 파드를 스케쥴링 한다. 
만약 만족하는 노드가 없다면 파드는 계속해서 `Pending` 상태에 빠지게 되고, 요구하는 자원만큼 여유 자원이 생길 때까지 대기한다.  

`.spec.containers[].resources.limits` 는 최대 사용가능한 자원을 의미한다. 
이를 통해 파드의 한 컨테이너가 노드의 전체 자원을 사용해서 다른 파드나 컨테이너에 영향을 끼치지 않도록 제한하는 역할을 수행한다.  


## 파드 환경 변수
환경 변수는 컨테이너의 활용성을 더욱 높여주는 역할을 수행한다. 
개발환경의 구성을 환경 변수의 수정만으로 실환경에 적용하는 방식도 가능하다. 

파드 템플릿에서 환경 변수 설정관련 필드는 아래와 같은 의미를 갖는다. 
- `name` : 환경 변수 이름
- `value`: 환경 변수의 값으로 문자열이나 숫자
- `valueFrom` : 참조 방식으로 환경 변수의 값 설정
- `fieldRef` : 파드의 현재 설정 내용의 값을 참조해서 설정
- `fieldPath` : `fieldRef` 에서 어느 항목의 위치를 참조할지 설정
- `resourceFieldRef` : 컨테이너에 설정 된 `CPU`, `Memory` 사용량의 정보를 참조해서 설정
- `containerName`: 환경 변수 설정을 가져올 컨테이너 이름
- `resource`: 어떤 자원 정보를 가져올지 설정

```yaml
# simple-env-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  labels:
    app: simple-pod
spec:
  containers:
    - name: simple-pod
      image: nginx:latest
      ports:
        - containerPort: 8080
      env:
        - name: E1
          value: "v1"
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: simple-pod
              resource: requests.cpu
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: simple-pod
              resource: limits.cpu
```  

위 파드 템플릿은 환경 변수 필드를 사용해서 환경 변수를 추가한 예시 템플릿이다. 
- `E1` : `v1` 이라는 문자열 설정
- `HOSTNAME` : `spec.nodeName` 을 참조해서 호스트 이름 설정
- `POD_NAME` : `metadata.name` 을 참조해서 파드 이름 설정
- `POD_IP` : `status.podIP` 을 참조해서 파드 아이피 설정
- `CPU_REQUEST` : `.containerName` 은 `.spec.containers[0].name` 값을 사용해서 컨테이너를 지정하고, 
`.resource` 는 `.spec.containers[].resources.requests.cpu` 를 참조해서 `CPU` 최소 요구량 설정
- `CPU_LIMIT` : 
- `CPU_REQUEST` : `.containerName` 은 `.spec.containers[0].name` 값을 사용해서 컨테이너를 지정하고, 
`.resource` 는 `.spec.containers[].resources.limits.cpu` 를 참조해서 `CPU` 최대 요구량 설정

환경 변수 테스트를 위해 지금까지 구성했던 파드 템플릿을 모두 합쳐 `simple-all-pod.yaml` 이라는 템플릿을 구성한다. 
```yaml
# simple-all-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
  labels:
    app: simple-pod
spec:
  initContainers:
    - name: init-a-service
      image: nginx:latest
      command: ['sh', '-c', 'sleep 2; echo init a service;']
    - name: init-db
      image: nginx:latest
      command: ['sh', '-c', 'sleep 2; echo init db;']
  containers:
    - name: simple-pod
      image: nginx:latest
      command: ['sh', '-c', 'echo init app running && sleep 3600']
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: 0.1
          memory: 100M
        limits:
          cpu: 0.5
          memory: 500M
      env:
        - name: E1
          value: "v1"
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: simple-pod
              resource: requests.cpu
        - name: CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: simple-pod
              resource: limits.cpu
```  

`kubectl apply -f simple-all-pod.yaml` 명령을 통해 파드를 실행하고 환경 변수를 확인 해본다. 
환경 변수 확인을 위해서 `kubectl exec -it simple-pod sh` 명령을 통해 컨테이너에 접속해서 수행한다. 

```bash
$ kubectl apply -f simple-env-pod.yaml
pod/simple-pod created
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
simple-pod   1/1     Running   0          20s
$ kubectl exec -it simple-pod sh
# env
POD_IP=10.1.0.17
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
CPU_REQUEST=1
HOSTNAME=docker-desktop
HOME=/root
PKG_RELEASE=1~buster
E1=v1
TERM=xterm
POD_NAME=simple-pod
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_VERSION=1.19.0
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
NJS_VERSION=0.4.1
KUBERNETES_PORT_443_TCP_PROTO=tcp
CPU_LIMIT=1
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```  

`KUBERNETES` 접두사가 있는 환경 변수는 쿠버네티스 안에서 사용 중인 자원관련 정보로, 파드를 생성할 때 기본으로 설정한다.  

파드 환경 설정을 변경하고 다시 적용 하기위해서는 `kubectl delete pod` 를 통해 삭제하고나서,
다시 `kubectl apply -f` 를 통해 실행해 주어야 한다. 
`CPU`, `Memory` 와 같은 자원관련 설정은 `kubectl describe pods` 를 통해 확인 가능하다. 


## 파드 구성 패턴
파드에는 여러 컨테이너가 구성되기 때문에, 이에 따른 몇가지 디자인 패턴이 존재한다. 
구글에서 오랜 기간동안 운영하며 정리한 [컨테이너 기반의 분산 시스템 디자인 패턴](https://static.googleusercontent.com/media/research.google.com/ko//pubs/archive/45406.pdf)
을 통해 보다 자세한 내용을 확인 할 수 있다. 

---
## Reference
