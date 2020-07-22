--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 레이블(Label)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 대상을 구분하는 레이블에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Label
  - Canary
toc: true
use_math: true
---  

## 레이블
레이블(`Label`)은 `key-value` 로 구성하는 값으로, 
클러스터에서 오브젝트를 구성할때 메타데이터(`.metadata`) 를 통해 설정할 수 있다.   

쿠버네티스의 레이블은 클러스터 안에서 대상을 구분하는 역할을 수행한다. 
또한 레이블을 통해 대상을 구분하고 관리 대상을 찾기 때문에, 
레이블 값이 잘못 설정되면 정상적으로 동작하지 못할 수 있다. 
이러한 점은 클러스터를 구성하는 전체적인 구성이 레이블을 통해 느슨하게 연결 되기 때문에 관리측면에서 유연석을 확보 할 수 있다.  

레이블 기반의 이런 유연함은 실제 서비스 중 장애가 발생한 파드를 별도로 분리해서 디버깅용으로 사용하는 등의 활용도 가능하다. 
또한 특정 레이블에만 특정 자원을 할당해서 실행하는 방식도 가능하다.  

이런 레이블을 사용할 때 지켜야할 규칙은 아래와 같다. 
- 최대 63글자
- 시작과 끝은 알파벳 대소문자 또는 숫자
- 중간에 `-`, `_`, `.` 사용 가능
- 레이블 키 앞에 `/` 구분자로 접두어 사용가능(DNS 하위 도메인 형식, 최대 253글자)

대표적으로 쿠버네티스 시스템이 사용하는 레이블의 경우 `kubernetes.io/` 라는 접두어가 붙는다.  

레이블 지정은 레이블 셀렉터(`Label selector`)를 사용한다. 
그리고 레이블 셀렉터의 방법은 아래와 같은 2가지 방법이 있다. 
- 등호 기반(`Equality based`) 셀렉터
- 집합 기반(`Set based`) 셀렉터


### 등호 기반 셀렉터
등호 기반 셀렉터는 레이블이 같은지(`=`, `==`), 다른지(`!=`) 를 구분하는 방식으로 동작한다. 

```
env=dev
release=stable
```  

`env=dev` 는 레이블 키가 `evn` 인 것 중 값이 `dev` 인 것을 설정한다. 
그리고 `release=stable` 은 레이블 키가 `release` 인 것 중 `stable` 인 것을 설정한다. 
만약 두 개를 모두 만족해야 하는 경우 아래와 같이 사용 할 수 있다. 

```
env=dev, release=stable
```  

### 집합 기반 셀렉터
집합 기반 셀렉터는 포함하는지(`in`), 포함하지 않는지(`notin`), 해당 레이블 키가 존재하는지(`exists`) 와 같은 연산을 사용할 수 있다. 

```
env in (dev, qa)
release notin (latest, canary)
storage
!storage
```  

`env in (dev, qa)` 는 레이블 키 `env` 의 값이 `dev` 이거나 `qa` 인 것을 선택한다. 
그리고 `release notin (latest, canary)` 는 레이블 키 `release` 의 값이 `latest` 와 `canary` 가 아닌 것을 모두 선택한다. 
`storage` 는 `storage` 라는 레이블 키가 있는 모든 레이블을 선택하고, 
`!storage` 는 `storage` 레이블 키가 없는 모든 레이블을 선택한다. 
여러 조건을 모두 만족해야 하는 경우 아래와 같이 작성 할 수 있다.

```
env in (dev, qa), release notin (latest, canary), !storage
```  


### 템플릿에서 사용
디플로이먼트와 서비스를 예시로 구성해서 레이블을 사용하고 활용하는 예제를 진행한다. 
여러 디플로이먼트 템플릿을 구성하고 각 다른 레이블을 설정한다. 
그리고 서비스도 종류로 나눠 구성해고 레이블을 사용해서 각 다른 디플로이먼트를 관리하도록 한다. 

디플로이먼트 템플릿 예시는 아래와 같다. 

```yaml
# deploy-nginx-1.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-1
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        env: dev
        release: beta
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```  

- `.spec.template.metadata.labels` : 디플로이먼트에서 레이블을 설정하는 필드이다. 
`app`, `env`, `release` 3개의 레이블 키로 설정을 구성한다. 

위와 같은 파일 구성으로 총 4개 디플로이먼트 템플릿을 만든다. 
다른 내용은 같지만 위에서 언급한 `app`, `env`, `release` 레이블 키에 설정되는 값은 아래와 같이 다르게 설정한다. 

파일 이름|app|env|release
---|---|---|---
deploy-nginx-1.yaml|nginx|dev|beta
deploy-nginx-2.yaml|nginx|prod|beta
deploy-nginx-3.yaml|nginx|dev|stable
deploy-nginx-4.yaml|nginx|prod|stable

서비스 템플릿 예시는 아래와 같다. 

```yaml
# service-dev.yaml

apiVersion: v1
kind: Service
metadata:
  name: dev-service
spec:
  type: ClusterIP
  selector:
    env: dev
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```  

- `.spec.selector` : 해당 서비스에서 관리할 대상을 레이블 셀렉터로 설정하는 필드이다. 
대상이 되는 레이블 키와 값으로 설정한다. 

위와 같은 파일 구성으로 총 2개 서비스 템플릿을 만든다. 
다른 내용은 동일하지만 `.spec.selector` 에서 지정하는 레이블의 키와 값이 달라진다.  

파일 이름|.spec.selector
---|---
service-dev.yaml|.env: dev
service-stable.yaml|.release: stable

디플로이먼트와 서비스 템플릿의 구성으로만 봤을 때 서로 연결되는 관계를 나타내면 아래와 같다. 

서비스|디플로이먼트
---|---
service-dev.yaml|deploy-nginx-1.yaml <br>deploy-nginx-3.yaml
service-stable.yaml|deploy-nginx-3.yaml <br>deploy-nginx-4.yaml

구성한 모든 템플릿을 `kubectl apply -f .` 명령으로 모두 적용한다. 
그리고 `kubectl get` 명령에 `-o wide` 옵션을 줘서 파드와 서비스의 상태를 확인하면 아래와 같다. 

```bash
$ kubectl apply -f .
deployment.apps/nginx-1 created
deployment.apps/nginx-2 created
deployment.apps/nginx-3 created
deployment.apps/nginx-4 created
service/dev-service created
service/stable-service created
$ kubectl get pod,svc -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP           NODE             NOMINATED NODE   READINESS GATES
pod/nginx-1-5b76cc4bf9-69jgs   1/1     Running   0          2m12s   10.1.3.255   docker-desktop   <none>           <none>
pod/nginx-2-7f74c8d4cf-zqb4k   1/1     Running   0          2m12s   10.1.4.0     docker-desktop   <none>           <none>
pod/nginx-3-68788c656d-4fc4t   1/1     Running   0          2m12s   10.1.4.1     docker-desktop   <none>           <none>
pod/nginx-4-58f9d786fb-9nt9g   1/1     Running   0          2m12s   10.1.4.2     docker-desktop   <none>           <none>

NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
service/dev-service      ClusterIP   10.100.98.205    <none>        80/TCP    2m12s   env=dev
service/stable-service   ClusterIP   10.101.100.155   <none>        80/TCP    2m12s   release=stable
```  

구성한 모든 파드가 정상적으로 실행되었고, 서비스의 셀렉터도 값이 정상적으로 설정된 것을 확인 할 수 있다. 
여기서 `kubectl describe` 명령으로 구성된 전체 서비스의 상세 정보를 확인하면 아래와 같다. 

```bash
kubectl describe svc
Name:              dev-service

.. 생략 ..

Endpoints:         10.1.3.255:80,10.1.4.1:80
Session Affinity:  None
Events:            <none>


Name:              stable-service

.. 생략 ..

Endpoints:         10.1.4.1:80,10.1.4.2:80
Session Affinity:  None
Events:            <none>
```  

상세 정보에서 `Endpoints` 필드를 `kubectl get pod -o wide` 에서 조회한 아이피와 매칭 시켜 나열하면 아래와 같이, 
서비스와 파드가 알맞게 연결된 것을 확인 할 수 있다.

서비스|Endpoints
---|---
dev-service|10.1.3.255:80(nginx-1)<br>10.1.4.1:80(nginx-3)
stable-service|10.1.4.1:80(nginx-3)<br>10.1.4.2:80(nginx-4)


### 커멘드라인에서 사용
`kubectl get` 명령에서 오브젝트에 설정된 레이블 값을 통해 조회하는 방법에 대해 살펴 본다. 
`kubectl get` 명령에서 `-l` 옵션을 주면 레이블을 기반으로 조회할 수 있다.  

먼저 파드에서 `app` 이 `nginx` 인 파드를 모두 조회하면 아래와 같다.

```bash
$ kubectl get pod -l app=nginx
NAME                       READY   STATUS    RESTARTS   AGE
nginx-1-5b76cc4bf9-69jgs   1/1     Running   0          11m
nginx-2-7f74c8d4cf-zqb4k   1/1     Running   0          11m
nginx-3-68788c656d-4fc4t   1/1     Running   0          11m
nginx-4-58f9d786fb-9nt9g   1/1     Running   0          11m
```  

다음으로 `env` 가 `dev` 이면서 `release` 가 `stable` 인 파드를 조회하면 아래와 같다. 

```bash
$ kubectl get pod -l env=dev,release=stable
NAME                       READY   STATUS    RESTARTS   AGE
nginx-3-68788c656d-4fc4t   1/1     Running   0          12m
```  

지금 까지는 등호 기반 셀렉터를 사용했다면, 
집합 기반 셀렉터로 `env` 가 `dev` 이거나 `prod` 이면서 `release` 가 `stable` 에 포함되지 않는 파드를 조회하면 아래와 같다.

```bash
$ kubectl get pod -l "env in (dev, prod), release notin (stable)"
NAME                       READY   STATUS    RESTARTS   AGE
nginx-1-5b76cc4bf9-69jgs   1/1     Running   0          15m
nginx-2-7f74c8d4cf-zqb4k   1/1     Running   0          15m
```  

마지막으로 `dev` 가 `prod` 가 아닌 파드를 조회하면 아래와 같다. 

```bash
$ kubectl get pod -l env!=prod
NAME                       READY   STATUS    RESTARTS   AGE
nginx-1-5b76cc4bf9-69jgs   1/1     Running   0          16m
nginx-3-68788c656d-4fc4t   1/1     Running   0          16m
```  


### 카나리 배포
쿠버네티스에서 디플로이먼트를 사용해서 배포를 할경우, 
전체 파드를 업데이트해서 교체하는 방식이기 때문에 카나리 배포 방식과 같은 특정 파드만 업데이트하는 방식은 불가능하다.
하지만 레이블을 사용하면 카나리 배포와 같은 방식으로 배포를 진행할 수 있다.  

테스트를 위해 2개의 디플로이먼트 템플릿과 하나의 서비스 템플릿을 구성한다. 
사용하는 이미지는 [Ingress nonstop]({{site.baseurl}}{% link _posts/kubernetes/2020-07-17-kubernetes-concept-ingress-nonstop.md %})
에서 사용했던 이미지를 그대로 사용한다. 
먼저 안정 버전인 디플로이먼트 템플릿은 아래와 같다. 

```yaml
# canary-deploy-v1.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-v1
  labels:
    app: canary-app
    version: stable
spec:
  replicas: 2
  selector:
    matchLabels:
      app: canary-app
      version: stable
  template:
    metadata:
      labels:
        app: canary-app
        version: stable
    spec:
      containers:
        - name: app
          image: windowforsun/nonstop-spring:v1
          ports:
            - containerPort: 8080
```  

테스트가 필요한 카나리 버전의 템플릿은 아래와 같다. 

```yaml
# canary-deploy-v2.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-v2
  labels:
    app: canary-app
    version: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: canary-app
      version: canary
  template:
    metadata:
      labels:
        app: canary-app
        version: canary
    spec:
      containers:
        - name: app
          image: windowforsun/nonstop-spring:v2
          ports:
            - containerPort: 8080
```  

- `.metadata.labels` : 현재 디플로이먼트에서 사용하는 컨테이너 버전을 레이블로 설정한다. 
`v1` 은 `stable` 이고, `v2` 은 `canary` 이다. 
- `.spec.replicas` : 복제 값은 `v1` 은 2개, `v2` 는 1개로 설정했다. 
- `.spec.template.spec.containers[].image` : `v1` 디플로이먼트는 이미지 태그가 `v1`, 
`v2` 디플로이먼트는 이미지 태그가 `v2` 로 설정했다. 

`kubectl apply -f` 명령으로 두 디플로이먼트를 모두 클러스터에 적용해 준다. 

```bash
$ kubectl apply -f canary-deploy-v1.yaml
deployment.apps/canary-v1 created
$ kubectl apply -f canary-deploy-v2.yaml
deployment.apps/canary-v2 created
$ kubectl get deploy,pod
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/canary-v1   2/2     2            2           22s
deployment.apps/canary-v2   1/1     1            1           19s

NAME                             READY   STATUS    RESTARTS   AGE
pod/canary-v1-689d994d56-bchx8   1/1     Running   0          22s
pod/canary-v1-689d994d56-sszxh   1/1     Running   0          22s
pod/canary-v2-5b899d4b6b-ln974   1/1     Running   0          19s
```  

두 디플로이먼트 파드에 접근하는 서비스 템플릿은 아래와 같다. 

```yaml
# canary-service.yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: canary-app
  name: canary-app-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: canary-app
  ports:
    - nodePort: 32222
      port: 8080
      protocol: TCP
      targetPort: 8080
```  

- `.spec.type` : `NodePort` 타입의 서비스를 설정한다. 
- `.spec.selector` : `app` 레이블 키가 `canary-app` 인 디플로이먼트와 연결하도록 설정한다. 

`kubectl apply -f` 로 서비스를 적용하고 `32222` 포트로 요청을 보내면 아래와 같은 결과를 확인 할 수 있다. 

```bash
$ kubectl apply -f canary-service.yaml
service/canary-app-service created
$ curl localhost:32222/sleep/0
v2
$ curl localhost:32222/sleep/0
v1
$ curl localhost:32222/sleep/0
v2
$ curl localhost:32222/sleep/0
v1
$ curl localhost:32222/sleep/0
v2
$ curl localhost:32222/sleep/0
v1
$ curl localhost:32222/sleep/0
v1
```  

요청에 대한 응답을 보면 `v1` 인 버전과 `v2` 인 버전이 동시에 요청을 처리하는 것을 확인 할 수 있다. 
`v1` 버전은 기존 서비스를 처리했던 안정적인 버전이고, `v2` 버전은 이번에 새로 배포하는 버전으로 생각 할 수 있다. 
이러한 방식으로 특정 파드 몇개만 신규 버전을 올려 테스트를 수행해 볼 수 있다.  

더 자세한 확인을 위해서 `kubectl get` 명령어와 `kubectl describe` 명령을 사용해서 살펴보면 아래와 같다. 

```bash
$ kubectl get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
canary-v1-689d994d56-bchx8   1/1     Running   0          15m   10.1.4.8    docker-desktop   <none>           <none>
canary-v1-689d994d56-sszxh   1/1     Running   0          15m   10.1.4.9    docker-desktop   <none>           <none>
canary-v2-5b899d4b6b-ln974   1/1     Running   0          15m   10.1.4.10   docker-desktop   <none>           <none>
$ kubectl describe svc canary-app-service
Name:                     canary-app-service

.. 생략 ..

Endpoints:                10.1.4.10:8080,10.1.4.8:8080,10.1.4.9:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```  

`canary-app-service` 의 `Endpoints` 필드를 확인하보면 `canary-v1` 과 `canary-v2` 디플로이먼트 파드에 해당하는 아이피가 설정된 것을 확인 할 수 있다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_label_plant_1.png)

`canary` 버전에 문제가 있다면 디플로이먼트를 삭제하거나, 
`.spec.replicas` 필드를 0으로 설정해서 서비스의 대상에서 제외 시킬 수 있다. 
그리고 다시 이슈가 수정되어 테스트를 진행하고 싶다면 디플로이먼트를 다시 올리거나, 복제 값을 올려 진행해 볼 수 있다. 
`canary` 버전이 완전히 안정됐다면, `stable` 버전의 이미지를 업데이트해서 전체 배포를 진행할 수 있다.

---
## Reference
