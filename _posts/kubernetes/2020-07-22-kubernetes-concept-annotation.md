--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 애너테이션(Annotation)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 필요한 정보를 설정할 수 있는 애너테이션에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Annotation
toc: true
use_math: true
---  

## 애너테이션
애너테이션(`Annotation`) 은 레이블과 비슷한 구조인 `key-value` 구조를 가지고 있다. 
그리고 사용자 필요에 따라 설정하거나 구성할 수 있다.  

하지만 레이블과 차이점이 있는데
레이블은 사용자가 정해진 값 없이 설절하고 클러스터에서 오브젝트를 식별하는데 사용된다면, 
애너테이션은 쿠버네티스 클러스터 시스템에 필요한 정보를 표현하거나, 
쿠버네티스 클라이언트나 라이브러리의 자원 관리에 사용된다.  

이렇게 애너테이션은 쿠버네티스가 인식할 수 있는 정해진 값을 사용한다. 
디플로이먼트에서 앱배포시에 변경 정보를 작성하거나, 
인그레스 컨트롤러인 `ingress-nginx` 를 애너테이션을 사용해서 직접 `nginx` 관련 설정을 수정할 수 있다. 
또한 사용시에 필요할 수 있는 릴리즈 정보, 로깅, 모니터링에 필요한 정보를 메모하는 용도로 사용 할 수도 있다. 
그리고 사용자에게 필요한 담당자, 연락처 등의 정보를 적어 관리용으로 사용도 할 수 있다.

### 예제 
아래 템플릿은 몇가지 애너테이션을 설정한 템플릿의 예시이다. 

```yaml
# annotation-deploy.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: annotation-nginx
  labels:
    app: annotation-nginx
  annotations:
    manager: "admin"
    contact: "010-1234-1234"
    release-version: "v1"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: annotation-nginx
  template:
    metadata:
      labels:
        app: annotation-nginx
    spec:
      containers:
        - name : annotation-nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```  

애너테이션은 `.metadata.annotations` 하위필드에 설정 할 수 있다.
- `.metadata.annotations.manager` : 담당자 정보를 설정한다.
- `.metadata.annotations.contact` : 담당자의 연락처를 설정한다.
- `.metadata.annotations.release-version` : 현재 디플로이먼트의 버전을 설정한다. 

먼저 `kubectl apply -f annotation-deploy.yaml` 명령어로 구성한 템플릿을 클러스터에 적용한다. 
그리고 `kubectl describe deploy annotation-nginx` 으로 디플로이먼트의 상세 정보를 확인하면 아래와 같다. 

```bash
$ kubectl apply -f annotation-deploy.yaml
deployment.apps/annotation-nginx created
$ kubectl describe deploy annotation-nginx
Name:                   annotation-nginx

.. 생략 ..

Labels:                 app=annotation-nginx
Annotations:            contact: 010-1234-1234
                        deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"contact":"010-1234-1234","manager":"admin","release-version":"v1"}...
                        manager: admin
                        release-version: v1
Selector:               app=annotation-nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable

.. 생략 ..
```  

`Annotations` 필드를 확인하면 앞서 템플릿에서 설정한 정보를 확인 할 수 있다. 
그리고 추가로 쿠버네티스에서 관리를 위해 추가한 정보도 확인 가능하다. 

---
## Reference
