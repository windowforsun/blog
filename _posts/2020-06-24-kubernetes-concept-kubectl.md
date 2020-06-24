--- 
layout: single
classes: wide
title: "[Kubernetes 개념] kubectl"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스의 대표적인 명령어인 kubectl 에 대해 알아보자'
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

`kutectl` 명령어를 통해 쿠버네티스의 클러스터를 관리하는 대부분의 동작을 수행할 수 있다. 
명령어가 수행하는 동작을 구분하면 아래와 같다. 
- 쿠버네티스 자원 생성, 업데이트 삭제(`create`, `update`, `delete`)
- 디버그, 모니터링, 트러블슈팅(`log`, `exec`, `cp`, `top`, `attach` ...)
- 클러스터 관리(`cordon`, `top`, `drain`, `taint` ...)

언급한 기능외에도 더욱 많은 기능이 있는데 더 자세한 내용은 [여기](https://kubernetes.io/ko/docs/reference/kubectl/cheatsheet/)
에서 확인 가능하다.  

## 설치
`kutectl` 은 마스터 노드에 설치되어 있고, 마스터 노드에서 직접 클러스터 관리자 권한으로 명령어를 수행할 수 있다. 
설치는 [여기](https://kubernetes.io/ko/docs/tasks/tools/install-kubectl/)
를 참고해서 진행할 수 있다. 
만약 Docker Desktop 을 사용중이라면 `kutectl` 은 이미 설치돼 있다. 

## 사용하기
명령어는 크게 아래와 같이 구성돼 있다. 

```
kutectl <command> <TYPE> <NAME> <flags>
```  

- `command` : 자원에 대해 실행하는 동작을 의미한다. `create`, `get`, `delete` 등이 있다. 
- `TYPE` : 자원 타입을 의미한다. `pod`, `service`, `ingress` 등이 있다.
- `NAME` : 지원의 이름을 의미한다.
- `flags` : 부가적으로 설정할 옵션을 의미한다.

에코서버 동작을 시키기 위해 `echoserver` 이름으로 파드(`pod`) 를 생성한다.

```bash
$ kubectl run echoserver --generator=run-pod/v1 --image="k8s.gcr.io/echoserver:1.10" --port=8080
pod/echoserver created
```  

쿠버네티스 파드들에 접근할 때 필요한 `echoserver` 라는 이름의 서비스를 생성한다. 

```bash
$ kubectl expose po echoserver --type=NodePort
service/echoserver exposed
```  

`kubectl get` 은 쿠버네티스에 있는 자원 상태를 확인할 때 사용하는 명령어이다. 
`get` 뒤에는 확인하려는 자원 이름과 옵션을 설정할 수 있다. 
생성한 파드의 상태를 확인해야 하기 때문에 `pods` 를 통해 확인한다. 

```bash
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
echoserver   1/1     Running   0          30s
```  

각 항목은 아래의 의미를 갖는다. 
- `NAME` : 파드의 이름을 의미한다.
- `READY` : 파드의 준비 상태를 의미한다. 0/1 이면 헌재 파드는 생성은 되었지만, 사용할 준비가 되지 않은 것이다. 1/1 은 생성되었고 사용준비까지 완료 된 것이다. 
- `STATUS` : 파드의 현재상태를 의미한다. `Running` 은 실행 중, `Terminating` 은 파드 생성 중, `ContainerCreating` 은 컨테이너 생성 중을 의미한다. 
- `RESTARTS` : 파드의 재시작 횟수를 의미한다. 
- `AGE`: 파드 생성 후 지난 시간을 의미한다. 

생성한 `echoserver` 서비스가 정상적으로 생성되었는지 확인하면 아래와 같다.

```bash
$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
echoserver   NodePort    10.110.177.61   <none>        8080:30322/TCP   4m26s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          3d8h
```  

각 항몽은 아래의 의미를 갖는다.
- `NAME` : 서비스의 이름을 의미한다. 
- `TYPE` : 서비스의 타입을 의미한다.
- `CLUSTER-IP` : 구성된 클러스터 안에서 사용하는 IP를 의미한다. 
- `EXTERNAL-IP` : 구성된 클러스터 외부에서 접속할 떄 사용하는 IP 를 의미한다. 현재는 설정되지 않았다.
- `PORT(S)` : 서비스에 접속하는 포트를 의미한다.
- `AGE` : 서비스 생성 후 지난 시간을 의미한다.

> `kubernetes` 서비스는 `kube-apiserver` 관련 파드를 가리킨다.

에코 서버 접속을 위해 포드포워딩을 하면 아래와 같다. 

```bash
kubectl port-forward svc/echoserver 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
Handling connection for 8080
```  

실행 후 웹 브라우저를 통해 `http://localhost:8080` 으로 접속하면 아래와 같은 결과를 확인할 수 있다. 

```


Hostname: echoserver

Pod Information:
	-no pod information available-

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=127.0.0.1
	method=GET
	real path=/
	query=
	request_version=1.1
	request_scheme=http
	request_uri=http://localhost:8080/

Request Headers:
	accept=text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
	accept-encoding=gzip, deflate, br
	accept-language=ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7,ja;q=0.6
	connection=keep-alive
	cookie=csrftoken=uAThpStvanbqacQjmXWVz1b1WJjN0Gex4AaidAFLUeUwUU7l3ImZapTMl40h4z55; Pastease.passive.activated.kfzz9br7dC3Qwzl=0; Pastease.passive.chance.kfzz9br7dC3Qwzl=1; Idea-eb07ea1c=8b5cd59b-4cbd-4fbf-be8c-308d0c39c2ad
	host=localhost:8080
	sec-fetch-dest=document
	sec-fetch-mode=navigate
	sec-fetch-site=none
	sec-fetch-user=?1
	upgrade-insecure-requests=1
	user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36

Request Body:
	-no body in request-
```  

실행한 파드의 로그를 수집하고 싶다면 `kubectl logs -f <파드이름>` 을 통해 가능하다. 
다른 쉘에서 실행하도록 한다. 

```bash
$ kubectl logs -f echoserver
Generating self-signed cert
Generating a 2048 bit RSA private key
..................................................................................+++
..................................................+++
writing new private key to '/certs/privateKey.key'
-----
Starting nginx
127.0.0.1 - - [24/Jun/2020:15:41:36 +0000] "GET / HTTP/1.1" 200 1104 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36"
127.0.0.1 - - [24/Jun/2020:15:41:37 +0000] "GET /favicon.ico HTTP/1.1" 200 1028 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36"
127.0.0.1 - - [24/Jun/2020:15:43:01 +0000] "GET / HTTP/1.1" 200 496 "-" "Mozilla/5.0 (Windows NT; Windows NT 10.0; ko-KR) WindowsPowerShell/5.1.18362.752"
127.0.0.1 - - [24/Jun/2020:15:45:02 +0000] "GET / HTTP/1.1" 200 1129 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36"
```  

생성된 파드와 서비스의 삭제는 `kubectl delete pod <파드이름>` ,`kubectl delete service <서비스이름>` 을 통해 가능하다.

```bash
$ kubectl delete pod echoserver
pod "echoserver" deleted
$ kubectl delete service echoserver
service "echoserver" deleted
```  

다시 파드와 서비스를 확인하면 아래와 같다.

```bash
$ kubectl get pods
No resources found.
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3d8h
```  

## POSIX/GNU 스타일 명령



---
## Reference
