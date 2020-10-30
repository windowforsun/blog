--- 
layout: single
classes: wide
title: "[Kubernetes 개념] "
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
toc: true
use_math: true
---  

## 로깅
쿠버네티스와 같은 클러스터 환경을 사용해서 서비스를 운영할 때 로깅을 할때는 주의가 필요하다. 
컨테이너 오케스트레이션 환경에서 로그를 수집할 때 주의해야 할 점은 특별한 경우외에는 
로컬 디스크에 로그를 파일로 저장하는 것은 좋지 않다.  

오케스트레이션 환경이 아닌 전통적인 방식에서는 로그를 파일로 지정된 위치에 저장하는 방식을 사용한다. 
그리고 로그로테이트(`logrotate`) 를 사용하거나 애플리케이션에서 제공하는 관리 라이브러리를 사용해서 
로그가 디스크 용량을 넘지 않도록 관리한다.  

위와 같은 방법은 애플리케이션이 지정한 호스트에서 실행된다는 가정을 두고 사용하던 방법이다. 
그래서 애플리케이션에서 문제가 발생하면 지정된 호스트에 접속해서 지정된 경로의 로그를 확인하는 
방법으로 로그를 확인하게 된다.  

하지만 쿠버네티스와 같은 컨테이너 오케스트레이션을 사용하는 환경에서는 위와 같은 방법을 사용한다면, 
로그 확인에 큰 어려움을 격을 수 있다. 
컨테이너 오케스트레이션에서 애플리케이션을 실행하는 컨테이너는 클러스터를 구성하는 노드를 옮겨다니며 
실행되기 때문이다. 
이후 부터는 컨테이너 오케스트레이션 환경에서 효율적으로 로그를 확인하고 관리할 수 있는 방법에 대해 알아본다.  


## 파드 로그 모니터링
쿠버네티스의 `kubectl` 명령을 사용하면 마스터 노드에서 개별 노드에 접근하지 않고, 
파드에서 생성된 로그를 확인해 볼 수 있다. (`kubectl log`)
테스트를 위해 아래 템플릿을 사용해서 `Nginx` 컨테이너를 클러스터에 생성한다. 

```yaml
# pod-nginx.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod-nginx
  labels:
    app: pod-nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```  

```bash
$ kubectl apply -f pod-nginx.yaml
pod/pod-nginx created
$ kubectl get pod
NAME        READY   STATUS    RESTARTS   AGE
pod-nginx   1/1     Running   0          59s
```  

그리고 `kubectl por-forward` 명령을 사용해서 아래와 같이
생성한 `pod-nginx` 가 외부의 요청을 받을 수 있도록 파드의 80포트와 호스트의 8080 포트를 포워딩 한다. 

```bash
$ kubectl port-forward pods/pod-nginx 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```  

다른 `Bash` 를 키고 `curl localhost:8080` 으로 요청을 보내면  
`Nginx` 응답이 오는 것을 확인할 수 있다.  
`kubectl log -f pod-nginx` 명령으로 로그를 확인하면 해당 파드의 로그 뿐만아니라, 
`Nginx` 컨테이너에서 생성한 `accessLog` 또한 확인할 수 있다. 

```bash
$ curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

.. 생략 ..
$ kubectl logs -f pod-nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
127.0.0.1 - - [29/Oct/2020:23:26:52 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.68.0" "-"
```  

이렇게 쿠버네티스에서 파드의 로그는 `kubectl log` 명령어를 사용해서 확인해 볼 수 있고, 
`-f` 옵션을 주게 되면 테일링까지 가능하다.  

## 여러 파드 로그 모니터링
로그를 확인해야 할 파드가 한개인 경우 위 방법을 사용해서 직접 명령어로 로그를 확인할 수 있다. 
하지만 분산 처리를 위해 `replicas` 설정이 돼있다면, 
특정 로그를 확인하기 위해 `replicas` 수 만큼 파드의 이름을 찾아 조회해야 할 것이다. 
또한 컨테이너에서 생성되는 로그는 설정된 도커 로그 관리 정책에 따라 시간이 지나면 지워지게 되고, 
컨테이너가 실행되는 호스트의 물리적인 한계 또한 존재한다.  

이러한 문젬을 개선할 수 있는 방법이 바로, 각 노드에서 생성한 로그를 한 곳에 모아 모니터링 할 수 있는 
로그 모니터링용 서버를 구축하는 것이다. 
이러한 구성을 가능하게 하는 오픈 소스 도구로는 카프카(`Kafka`), 플루언트디(`fluentd`), 로그스태시(`logstash`), 
일래스틱서치(`Elasticsearch`), 키바나(`Kibana`) 등이 있다. 
클라우드 서비스를 사용한다면 각 클라우드에서 제공하는 로그 수집 도구를 사용할 수 있지만, 
이를 위 도구들을 사용해서 직접 구축해서 사용할 수 도 있다. 

### 일래스틱서치
일래스틱서치는 검색 엔진으로 클러스터 형태로 여러 노드에서 실행할 수 있다. 
수집되는 로그들은 일래스틱서치에 저장되고, 특정 로그를 검색 및 정렬도 가능하다. 

아래 템플릿은 디플로이먼트를 사용한 일래스틱서치의 템플릿 예시이다. 

```yaml

```  






---
## Reference
