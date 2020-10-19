--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 네트워킹"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '파드 및 쿠버네트스 클러스터에서 서비스의 네트워킹과 플러그인에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Pod Network
  - Service Network
toc: true
use_math: true
---  

## 파드 네트워킹
쿠버네티스 파드의 네트워킹 관련을 이해하기 위해서는 먼저 도커 네트워크에 대한 이해가 필요하다. 
해당 부분은 [여기]({{site.baseurl}}{% link _posts/docker/2020-10-19-docker-practice-networking.md %})
를 참고하도록 한다.  

쿠버네티스는 파드 단위로 컨테이너를 관리하고, 
파드는 쿠버네티스에서 생성한 `pause` 컨테이너와 함께 사용자가 생성한 컨테이너 그룹을 의미한다. 
그리고 하나의 파드에 속한 컨테이너는 모두 동일한 `IP` 를 갖게 된다.  

![그림 1]({{site.baseurl}}/img/kubernetes/concept_networking_pod_1.png)

위 그림과 같이 파드에 속한 컨테이너는 하나의 `veth0` 을 공유하게 된다. 
하나의 파드에 속한 사용자 컨테이너는 `pause` 컨테이너 네트워크 네임스페이스에 해당하므로, 
모두 같은 `IP` 를 사용하게 된다. 
`pause` 컨테이너는 인프라 컨테이너 역할로 쿠버네티스가 생성하고 관리한다.  

`veth0` 은 `pause` 컨테이너 네임스페이스에 속한 네트워크 인터페이스이다. 
그리고 해당 네트워크 인터페이스를 파드에 실행 되는 다른 컨테이너들이 공유해서 함께 사용하기 때문에, 
`pause` 컨테이너가 재시작 등으로 변하지 않는 다면 `veth0` 에 할당된 동일한 `IP` 를 사용할 수 있다. 
반대로 `pause` 컨테이너에 문제가 발생하면 다른 컨테이너는 정상이더라도 네트워크 통신에 장애가 발생한다.  

이런 특징으로 하나의 파드에서 실행되는 컨테이너 들은 `localhost` 혹은 `127.0.0.1` 주소를 사용해서 
서로간 통신이 가능하다.  

파드에서 컨테이너간 통신 테스트를 위해 아래 템플릿으로 `pod-net-test` 라는 파드 를 구성한다. 

```yaml
# pod-network-test.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-net-test
spec:
  containers:
    - name: nginx
      image: nginx:latest
    - name: ubuntu
      image: ubuntu:latest
      command: ["/bin/sh", "-c", "while : ;do sleep 5; done"]
```  

`pod-net-test` 파드에는 `nginx`, `ubuntu` 2개의 컨테이너가 실행된다. 
테스트는 `ubuntu` 컨테이너에서 `nginx` 컨테이너 쪽으로 `curl` 요청을 보내 정상적으로 응답이 오는지로 진행한다. 
`kubectl apply -f pod-network-test.yaml` 명령으로 템플릿을 클러스터에 적용한다. 
그리고 `docker ps -f name=pod-net-test` 명령으로 `pod-net-test` 파드에 해당하는 도커 컨테이너를 조회하면 아래와 같다. 

```bash
$ kubectl apply -f pod-network-test.yaml
pod/pod-net-test created
$ docker ps -f name=pod-net-test
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
959f78e968a1        ubuntu                 "/bin/sh -c 'while :…"   13 minutes ago      Up 13 minutes                           k8s_ubuntu_pod-net-test_default_5c29fa39-3315-4860-b528-2440ecee410e_0
de45d6a669a6        nginx                  "/docker-entrypoint.…"   13 minutes ago      Up 13 minutes                           k8s_nginx_pod-net-test_default_5c29fa39-3315-4860-b528-2440ecee410e_0
e2bd0cf93319        k8s.gcr.io/pause:3.1   "/pause"                 13 minutes ago      Up 13 minutes                           k8s_POD_pod-net-test_default_5c29fa39-3315-4860-b528-2440ecee410e_0
```  

앞서 설명한 것처럼 템플릿에서 정의한 사용자 컨테이너인 `ubuntu`, `nginx` 와 함께 `pause` 컨테이너가 실행된 것을 확인할 수 있다.  

테스트전 먼저 `nginx`, `ubuntu` 에 아래와 같이 필요한 패키지를 설치해 준다. 

```bash
$ kubectl exec pod-net-test -c nginx apt-get update
$ kubectl exec pod-net-test -c nginx -- apt-get install -y iproute2
$ kubectl exec pod-net-test -c ubuntu apt-get update
$ kubectl exec pod-net-test -c ubuntu -- apt-get install -y iproute2 curl
```  

그리고 `'/sbin/ip' 'a'` 명령으로 `nginx` 와 `ubuntu` 컨테이너에 할당된 아이피를 확인하면 아래와 같다. 

```bash
$ kubectl exec pod-net-test -c nginx '/sbin/ip' 'a'
4: eth0@if44: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 2e:50:68:05:66:44 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.4.170/16 scope global eth0
       valid_lft forever preferred_lft forever
$ kubectl exec pod-net-test -c ubuntu '/sbin/ip' 'a'
4: eth0@if44: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 2e:50:68:05:66:44 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.4.170/16 scope global eth0
       valid_lft forever preferred_lft forever
```  

두 컨테이너 모두 아이피가 `10.1.4.170` 으로 동일한 것을 확인 할 수 있다. 
파드의 `pause` 컨테이너의 네트워크를 사용한다는 것을 알 수 있다.  

`ubuntu` 파드에서 먼저 `nginx` 가 프로세스로 실행 중인지 확인 후, 
`curl http://localhost` 로 요청을 보내본다. 

```bash
$ kubectl exec pod-net-test -c ubuntu ps
  PID TTY          TIME CMD
    1 ?        00:00:00 sh
 3014 ?        00:00:00 sleep
 3015 ?        00:00:00 ps
$ kubectl exec pod-net-test -c ubuntu curl http://localhost
.. 생략 ..

<html>
<head>
<title>Welcome to nginx!</title>

.. 생략 ..
```  

`ubuntu` 컨테이너에는 `nginx` 가 실행 중이지 않지만, 
`http://localhost` 로 요청을 보내면 `nginx` 페이지 응답이 오는 것을 확인할 수 있다. 
`nginx` 컨테이너 로그를 확인하면 아래와 같이 `nginx` 컨테이너 쪽으로 요청이 들어와 처리되고 응답된 것을 확인할 수 있다. 

```bash
$ kubectl logs pod-net-test -c nginx
127.0.0.1 - - [19/Oct/2020:18:18:43 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
```  

동일 파드내 컨테이너는 네트워크 네임스페이스 영역만 공유할 뿐 나머지 네임스페이스인 프로세스, 파일, 시스템 등은 공유하지 않는다.  

`pod-net-test` 파드에서 실행중인 컨테이너를 다시한번 확인하고, 
`ubuntu`, `nginx` 컨테이너의 `NetworkMode` 를 확인하면 아래와 같다. 

```bash
$ docker ps -f name=pod-net-test
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
959f78e968a1        ubuntu                 "/bin/sh -c 'while :…"   21 minutes ago      Up 21 minutes                           k8s_ubuntu_pod-net-test_default_5c29fa39-3315-4860-b528-2440ecee410e_0
de45d6a669a6        nginx                  "/docker-entrypoint.…"   21 minutes ago      Up 21 minutes                           k8s_nginx_pod-net-test_default_5c29fa39-3315-4860-b528-2440ecee410e_0
e2bd0cf93319        k8s.gcr.io/pause:3.1   "/pause"                 21 minutes ago      Up 21 minutes                           k8s_POD_pod-net-test_default_5c29fa39-3315-4860-b528-2440ecee410e_0
$ docker inspect -f '{{ .HostConfig.NetworkMode }}' 959
container:e2bd0cf933199bf7512595bdfcbec4d0ccea4a67e6a060a114f9942c8f3b1040
$ docker inspect -f '{{ .HostConfig.NetworkMode }}' de45
container:e2bd0cf933199bf7512595bdfcbec4d0ccea4a67e6a060a114f9942c8f3b1040
```  

`ubuntu`, `nginx` 의 `NetworkMode` 의 설정 값이 
도커 네트워크 드라이버의 값이 아닌 `pause` 컨테이너의 아이디 인것을 확인할 수 있다. 
이로써 다시 한번 파드에 정의된 사용자 컨테이너는 `pause` 네트워크 네임스페이스를 공유해서 사용한다는 것을 알 수 있다. 
또한 파드에 구성되는 컨테이너는 동일한 네트워크이기 때문에 포트로 구분되어야 한다는 점도 기억해야 한다.  

지금까지는 단일 호스트에 대한 파드 네트워크에 대해 살펴봤다. 
다중 호스트로 구성된다면 아래와 같은 구조로 구성될 수 있다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_networking_multihost_1.png)

그림의 상황은 각 호스트의 `eth0` 의 아이피는 각각 다르지만, 
호스트에 구성된 `docker0` 와 `veth0` 의 아이피는 모두 같다. 
쿠버네티스는 기본적으로 클러스터에 실행 중이 파드들이 고유한 아이피를 갖도록 설정한다. 
하지만 위와 같은 상황이라면 호스트내 파드는 고유한 아이피를 가지겠지만, 
클러스터 전체로 본다면 아이피가 중복되는 파드가 생길 여지가 분명하다. 
이로 인해 만약 파드 아이피인 `172.17.0.2` 로 요청이 온다면 쿠버네티스는 `Host_1` 과 `Host_2` 중 
어느 곳으로 트래픽을 전달해야 할지 결정하지 못하는 상황이 발생할 수 있다.  

이러한 문제로 쿠버네티스에서 사용하는 다중 호스트의 네트워크 구조는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_networking_multihost_2.png)










---
## Reference
