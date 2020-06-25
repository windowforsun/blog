--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 컨테이너 실행하기"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '디플로이먼트를 사용해서 컨테이너를 실행해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - kubectl
toc: true
use_math: true
---  

## 컨테이너 실행
쿠버네티스에서 컨테이너를 실행하는 방법은 아래 2가지 방법이 있다.
- `kubectl run` 명령을 통해 직접 컨테이너 실행하기
- `YAML` 파일에 컨테이너 실행관련 세부 내용을 작성해 템플릿으로 실행

`YAML` 을 통해 실행할 경우 버전관리에도 용의하고 자원이나, 설정 변경을 파악에서 용의하다.

## kubectl run
쿠버네티스는 파드를 실행하는 여러 가지 컨트롤러를 제공하는데, 
`kubectl run` 을 통해 파드를 실행하면 기본 컨트롤러는 `deployment`(디플로이먼트) 이다. 

아래 명령어를 통해 `Nginx` 컨테이너를 실행 시킨다. 

```bash
$ kubectl run nginx-app --image nginx --port=80
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx-app created
```  

- `--image` 에는 사용할 컨테이너 이미지 이름
- `--port` 에는 사용할 포트 번호

![그림 1]({{site.baseurl}}/img/kubernetes/concept_run_container_plant_1.png)

위 그림과 같이 클러스터에 컨테이너를 실행 명령을 내리면,
지정된 컨테이너 이미지를 가져와서 클러스터에 실행 시킨다. 
`kubectl get pods` 를 통해 실행 여부를 확인하면 아래와 같다.

```bash
$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
nginx-app-cc84d4cc6-nbhcx   1/1     Running   0          9m27s
```  

`kubectl get deployments` 명령을 통해 디플로이먼트 상태를 확인하면 아래와 같다.

```bash
$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   1/1     1            1           10m
```  

각 항목은 아래와 같은 의미를 갖는다. 
- `NAME` : 클러스터에 배포한 디플로이먼트 이름
- `READY` : `사용자가 최종 배포한 파드 수 / 현재 클러스터에 실제로 동작된 파드 수` 의 표현이다. 
디플로이먼트를 새로 생성하거나 디플로이먼트 설정을 변경할 경우 새로운 버전의 파드를 배포한다. 
새로운 버전의 디플로이먼트가 배포된 경우 이전, 신규 버전의 파드 개수 합을 표시한다. 
그러므로 `사용자가 최종 배포한 파드 수` 보다 `현재 클러스터에 실제로 동작된 파드 수` 가 더 클 수도 있다.
- `UP-TO-DATE` : 디플로이먼트 설정에 정의한 대로 동작 중인 신규 파드 수
- `AVAILABLE` : 서비스 가능한 파드 수, 파드 실행 후 헬스 체크를 통해 서비스 가능 상태인지 판별한다.
- `AGE` : 생성후 지금까지의 시간

`kubectl scale deploy <디플로이먼트이름> --replicas=<개수>` 를 통해 실행 중인 파드 수를 늘리면 아래와 같다.

```bash
$ kubectl scale deploy nginx-app --replicas=2
deployment.extensions/nginx-app scaled

$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
nginx-app-cc84d4cc6-nbhcx   1/1     Running   0          17m
nginx-app-cc84d4cc6-rlj98   1/1     Running   0          21s

$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   2/2     2            2           18m
```  

`kubectl delete deployment <디플로이먼트이름>` 을 통해 생성된 디플로이먼트를 삭제 할 수 있다. 

```bash
$ kubectl delete deployment nginx-app
deployment.extensions "nginx-app" deleted

$ kubectl get pods
No resources found.

$ kubectl get deployments
No resources found.
```  

## YAML 템플릿
`YAML` 템플릿을 통해 컨테이너를 실행할 때 사용할 파일 내용은 아래와 같다. 
이미지는 동일한 `Nginx` 를 사용한다. 

```yaml
# nginx-app.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
  app: nginx-app
spec:
  replicas: 1
  selector:
	matchLabels:
      app: nginx-app
  template:
	metadata:
      labels:
	    app: nginx-app
	spec:
	  container:
	  - name: nginx-app
	    image: nginx
	    ports:
	    - containerPort: 80			
```  

`kubectl apply -f nginx-app.yaml` 명령을 통해 실행하면 아래와 같다.

```bash
$ kubectl apply -f nginx-app.yaml
deployment.apps/nginx-app created

$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
nginx-app-6dd64896c8-2g4fx   1/1     Running   0          23s

$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   1/1     1            1           33s
```  

쿠버네티스 자원은 `YAML` 템플릿과 `kubectl apply` 명령을 통해 선언적으로 관리할 것을 권장한다. 
자원 생성시에도 템플릿 파일을 사용해서 소스코드와 함께 버전관리(Git)을 통해 관리하는 것이 좋다.


---
## Reference
