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

아래 템플릿은 디플로이먼트와 서비스를 사용한 일래스틱서치의 템플릿 예시이다. 

```yaml
# elasticsearch.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  labels:
    app: elasticsearch
spec:
  # 단일 노드로 일래스틱서치를 구성한다.
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
        # 일래스틱서치 컨테이너 이미지와 설정을 기술한다. 
        - name: elasticsearch
          image: elastic/elasticsearch:7.9.3
          # discovery.type 환경 변수를 single-node 값으로 설정해서 단일 노드로 실행 하도록 한다.  
          env:
            - name: discovery.type
              value: "single-node"
          # 일래스틱서치에서 필요한 9200, 9300 포트를 설정한다. 
          ports:
            - containerPort: 9200
            - containerPort: 9300

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: elasticsearch
  name: elasticsearch-svc
  namespace: default
spec:
  type: NodePort
  selector:
    app: elasticsearch
  # 디플로이먼트에서 설정한 9200, 9300 포트를 30920, 30930 포트를 사용해서 외부에서 접근하 수 있도록 한다. 
  ports:
    - name: elasticsearch-rest
      nodePort: 30920
      port: 9200
      protocol: TCP
      targetPort: 9200
    - name: elasticsearch-nodecom
      nodePort: 30930
      port: 9300
      protocol: TCP
      targetPort: 9300
```  

`kubectl apply -f` 명령으로 위 템플릿을 클러스터에 적용하고, 
`localhost:30920` 으로 웹 브라우저 혹은 `curl` 로 요청하면 아래 결과를 확인 할 수 있다. 

```bash
$ kubectl apply -f elastaicsearch.yaml
deployment.apps/elasticsearch created
service/elasticsearch-svc created
$ curl http://localhost:30920
{
  "name" : "elasticsearch-7bc6b99f4-ssdfn",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "3cU-YM_STNC03RMtDwQwpA",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```  

일래스틱서치를 사용하면 `REST API` 를 사용해서 로그 데이터를 저장하고 검색할 수 있다. 
이는 `REST API` 를 사용하는 것은 시각적으로 불편하기 때문에 
일래스틱서치 전용 대시보드 역할을 하는 키바나를 함께 사용하면 활용도를 더욱 높일 수 있다. 


### 키바나
아래 디플로이먼트와 서비스로 구성된 키바나 템플릿의 예시이다.  

```yaml
# kibana.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
        - name: kibana
          image: elastic/kibana:7.9.3
          env:
            - name: SERVER_NAME
              value: "kibana.k8s.test.com"
            # ELASTICSEARCH_HOSTS 환경 변수에 일래스틱서치 서비스 도메인과 포트를 설정한다. 
            - name: ELASTICSEARCH_HOSTS
              value: "http://elasticsearch-svc.default.svc.cluster.local:9200"
          ports:
            - containerPort: 5601

---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kibana
  name: kibana-svc
  namespace: default
spec:
  type: NodePort
  selector:
    app: kibana
  ports:
    - name: kibana-http
      port: 5601
      protocol: TCP
      targetPort: 5601
      nodePort: 30561
```  

`ELASTICSEARCH_HOST` 환경 변수에 설정하는 값은 쿠버네티스 내부에서 사용할 수 있는 도메인이다. 
포트를 제외한 뒤쪽 부터 차례대로 살펴보면 `local` 이라는 `cluster` 에 있는 `svc`(서비스) 중 `default` 네임스페이스의 
`elasticsearch-svc` 서비스를 가리키는 도메인이다. 
해당 도메인을 사용하면 쿠버네티스 클러스터 내부에서 도메인에 해당하는 컨테이너의 포트에 접근 할 수 있다. 
직접 `elasticsearch-svc` 의 `IP` 를 설정하는 것도 가능하지만, 
서비스의 아이피는 재실행 될때마다 변경될 수 있기 때문에 도메인으로 사용하는 것이 더욱 효율적일 수 있다.  

`kubectl apply -f` 명령으로 키바나 템플릿을 클러스터에 적용하고, 
웹 브라우저에서 `localhost:30561` 로 접속하면 키바나 대시보드를 확인할 수 있다. 

```bash
$ kubectl apply -f kibana.yaml
deployment.apps/kibana created
service/kibana-svc created
```  

![그림 1]({{site.baseurl}}/img/kubernetes/concept-logging-1.png)

우선 일래스틱서치로 로그를 보내면 이를 키바나에서 모니터링 가능한 구조를 구성해 보았다.


## 클러스터 레벨 로깅
클러스터 환경에서는 컨테이너가 비정상 종료되거나 노드에 장애가 있더라도 컨테이너의 로그 확인은 가능해야 한다. 
그러기 때문에 컨테이너, 파드, 노드의 생명 주기와 분리된 스토리지 구축이 필요하다.  

이렇게 쿠버네티스에서 생명주기와 분리된 스토리지를 구축하는 아키텍처를 클러스터 레벨 로깅이라고 한다. 
쿠버네티스 자체적으로 클러스터 레벨 로깅관련 도구를 제공하지 않기 때문에 외부 도구를 사용해서 이를 구축해야 한다.  

### 컨테이너 로그 
일반적으로 컨테이너의 로그 수집은 컨테이너의 런타임인 도커가 담당한다. 
도커 컨테이너에서 `stdout` 혹은 `stderr` 출력이 있으면 이를 도커에서 설정된 로그 드라이버로 리다이렉트 하는 방식을 사용한다. 
도커 로그관련 자세한 설명은 [여기]({{site.baseurl}}{% link _posts/docker/2020-08-06-docker-practice-docker-jsonfile-logdriver.md %})
에서 확인 가능하다.  

`docker ps` 명령으로 현재 실행 중인 컨테이너를 확인하고, 
그중 아무 컨테이너 아이디를 사용해서 `docker inspect <컨테이너ID>` 명령을 수행하면 아래와 같이 로그관련 정보를 확인 할 수 있다. 

```bash
docker inspect 4408ddb2c2bb
[
    {
        .. 생략 ..

        "LogPath": "/var/lib/docker/containers/<컨테이너ID>/<컨테이너ID>-json.log",
        "HostConfig": {
            "LogConfig": {
                "Type": "json-file",
                "Config": {
                    "max-file": "3",
                    "max-size": "10m"
                }
            },

            .. 생략 ..
        },

    .. 생략 ..
]
```  

`docker inspect` 결과에서 `LogPath` 필드는 컨테이너 관련 메타 데이터를 저장한 심볼릭 링크이다. 
그리고 `LogConfig` 필드는 로그 설정 정보를 담고 있다. 
`Type` 은 현재 컨테이너에서 사용하는 도커 로그 드라이버이고, 
`Config` 의 `max-file` 은 컨테이너에서 유지하는 로그 파일의 최대 개수이고, 
`max-size` 는 파일당 최대 크기를 의미한다.  

클러스터 각 노드에서 실행되는 `kubelet` 은 `/var/lib/docker/<컨테이너ID>/<컨테이너ID>-json.log` 파일에 대해 아래와 같은 심볼릭 링크를 생성해서 로그를 관리한다. 
- `/var/log/container/<파드이름>_<파드네임스페이스>_<컨테이너이름><컨테이너ID>.log`
- `/var/log/pods/<파드UID>/<컨테이너이름>/0.log`

플루언트디 같은 로그 수집기는 첫 번째 심볼릭 링크를 테일링해서 로그 내용으 수집하고, 
파일 경로를 바탕으로 파드, 컨테이너, 네임스페이스 등에 대한 정보를 얻는다. 

### 시스템 컴포넌트 로그
쿠버네티스 클러스터 구성요소 중 `kubelet`, `docker` 등은 컨테이너 기반으로 동작하지 않는다. 



---
## Reference
