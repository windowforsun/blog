--- 
layout: single
classes: wide
title: "[Nginx] Rate Limiting"
header:
  overlay_image: /img/server-bg.jpg
excerpt: 'Back-end 서버로 전달되는 요청을 제한하고 관리할 수 있는 Rate Limiting 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Server
tags:
  - Server
  - Rate Limiting
  - Nginx
---  

## Rate Limiting
`Rate Limiting` 이란 말 그대로 요청을 정해진 가용량 만큼만 받고, 
나머지는 에러 처리를 하는 것을 의미한다. 
무한한 자원을 가진 물리 머신은 존재하지 않고, 
그에 따라 항상 서버가 받을 수 있는 요청의 최대치는 존재하게 된다.  
그리고 최대치 이상으로 요청이 오게되면 서버가 죽어버리거나 가용할 수 없는 상태가 된다.  

이렇게 `DDos` 공격과 같이 많은 요청이 한번이 몰릴때 `Rate Limiting` 을 이용한다면, 이를 정해진 가용량 만큼만 받고 
나머지 요청에 대해서는 적절한 에러 응답을 내려 주는 방법으로 보다 안정적인 서비스를 구성할 수 있다.  

## Rate Limiting 의 원리
`Nginx` 의 `Rate Limting` 동작은 대역폭이 제한될 때, 
`Burst`(초과분)를 처리하기 위해 널리 사용되는 [Leaky Bucket Algorithm(https://en.wikipedia.org/wiki/Leaky_bucket) 
을 사용한다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-1.jfif)


위 알고리즘은 주로 구멍이 뚫린 양동이에 물을 쏟아 붓는 비유를 많이 하는데, 
물을 붓는 속도와 양이 너무 크면 구멍으로 물이 빠져 나가기전에 양동이를 넘처 흐르게 될 것이다. 
이를 `Nginx` 에 비유하면 물은 클라이언트의 요청이되고, 
양동이는 `FIFO` 스케쥴링 알고리즘에 따라 요청을 처리를 위해 대기하는 버퍼 공간을 의미한다. 
그리고 구멍으로 빠져나가는 물은 서버에서 처리하기 위해 버퍼에서 나가는 요청이고, 
넘쳐 흐르는 물은 서비스되지 못하고 버려지는 요청을 의미한다.  

## Rate Limiting 설정(Basic)
아래는 간단한 `Rate Limiting` 설정의 예시이다. 

```
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=100r/s;

server {
    location /test {
        limit_req zone=mylimit;

        proxy_pass http://backend;
    }
}
```  

`Rate Limiting` 의 기본적인 설정은 크게 `limit_req_zone` 과 
`limit_req` 로 구성된 것을 확인할 수 있다. 
`limit_req_zone` 은 `Rate Limting` 에 대한 정의를 하는 부분이고, 
`limit_req` 는 특정 경로에 대해 `Rate Limiting` 을 적용하는 역할을 한다.  

`limit_req_zone <key> <zone> <rate>` 와 같이 크게 3가지로 구성되는 그에 대한 설명은 아래와 같다. 
- `<key>` : `Rate Limting` 이 적용되는 기준 값을 설정한다. 
위 예에서는 `Nginx` 변수인 `$binary_remote_addr`(이진 클라이언트 IP주소)를 사용했다. 
즉 클라이언트 IP 를 기준으로 요청 제한을 수행하게 된다. 
추가로 `$remote_addr` 도 설정 할 수 있다.
- `<zone>` : `Rate Limiting` 적용 기준이 되는 `IP`를 저장하는 공유 메모리에 대한 정의와 `URL` 에 따른 처리 빈도를 정의한다. 
공유 메모리라는 것은 `Nginx` 에 구동되는 `worker_processor` 간 공유 되는 메모리를 의미한다. 
그리고 `zone=<name>:<size>` 와 같이 다시 2개로 구성되는데, 
`<name>` 은 공유 메모리의 이름을 정의하고, `<size>` 해당 공유 메모리의 크기를 정의한다. 
여기서 `1m` 은 `1MB` 를 의미하고 1600개의 `IP` 정보를 저장 할 수 있다. 
그러므로 예시 설정에서는 160000개 `IP`를 저장할 수 있는 공간을 할당 한 것이다. 
>모든 공간이 차면 가장 오래된 `IP` 를 삭제하고, 더 이상 가용한 공간이 없는 경우 `503` 에러를 응답한다. 
- `<rate>` : 최대 요청 속도를 설정한다. 
예시 설정에서는 초당 100개의 요청을 초과할 수 없도록 설정된 상태이다. 
`Nginx` 는 요청을 미리초 단위로 추적하므로 현재 설정은 `10ms` 마다 1개의 요청을 받을 수 있다고 해석할 수 있다. 
이후 설정할 `Burst` 에 대한 설정이 없기 때문에 이전 요청보다 `10ms` 이내 도착한 요청은 거부 된다. 

`limit_req_zone` 은 `Rate Limiting` 관련 선언만 할 뿐, 
실제 `Rate Limiting` 동작을 수행하도록 하는 구문은 아니다. 
`Rate Limiting` 동작을 위해서는 `server`, `location` 블록에 `limit_req` 
구문을 통해 공유 메모리 이름과 함께 명시적으로 작성해 주어야 한다.  

`/test` 경로에 대해서 고유한 클라이언트 `IP` 에 대해 초당 100개의 요청만 가능하다. 
다시 말하면, `/test` 경로는 이전 `IP` 주소의 요청으로 부터 `10ms` 이내 요청은 할 수 없다.  


### 테스트
이후 진행하는 테스트를 포함해서 테스트는 `JMeter` 툴을 사용한다. 
그리고 테스트 환경은 `kubernetes` 르 사용해서 구성한다.  

- `basic-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: basic-configmap
  namespace: default
data:
  nginx.conf: |
    user  nginx;
    worker_processes  2;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        limit_req_zone $binary_remote_addr zone=mylimit:10m rate=100r/s;

        server {
            listen 80;
            server_name localhost;

            location /status {
                limit_req zone=mylimit;
                stub_status on;
                allow all;
                deny all;
            }
        }
    }
```  

- `basic-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-rate-limiting
  labels:
    app: basic-rate-limiting
    type: rate-limiting
spec:
  containers:
    - name: basic-rate-limiting
      image: nginx:1.19
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
  volumes:
    - name: nginx-conf
      configMap:
        name: basic-configmap
        items:
          - key: nginx.conf
            path: nginx.conf
```  

- `service.yaml`
    - 이후 테스트에서도 공통으로 사용한다. 
    
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rate-limiting-service
spec:
  type: NodePort
  selector:
    type: rate-limiting
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 32222
```  

작성한 모든 템플릿을 `Kubernetes` 클러스터에 아래 명령으로 적용한다. 

```bash
$ kubectl apply -f basic-configmap.yaml
configmap/basic-configmap created
$ kubectl apply -f basic-pod.yaml
pod/basic-rate-limiting created
$ kubectl apply -f service.yaml
service/rate-limiting-service created
$ kubectl get configmap,pod,svc
NAME                             DATA   AGE
configmap/basic-configmap        1      20s

NAME                      READY   STATUS    RESTARTS   AGE
pod/basic-rate-limiting   1/1     Running   0          14s

NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/rate-limiting-service   NodePort    10.105.130.60   <none>        80:32222/TCP   8s
```  

`localhost:32222/status` 로 요청을 보내면 아래와 같이 `Nginx` 상태 정보가 출력된다. 

```bash
$ curl localhost:32222/status
Active connections: 1
server accepts handled requests
 1 1 1
Reading: 0 Writing: 1 Waiting: 0
```  

`JMeter` 툴의 설정은 아래와 같다. 
- `Number of Thread` : 100
- `Ramp-up period`: 100
- `Loop Count` : 100

테스트는 `Throughput Shaping Timer` 플러그인을 사용해서 아래와 같은 `RPS` 와 시간으로 수행한다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-expected-rps.png)

Start RPS|End RPS|Duration, sec
---|---|---
1|100|5
100|100|5
120|120|4
140|140|4
160|160|4
180|180|4
200|200|4

위 `RPS` 에 따른 테스트는 이후 테스트에서도 계속해서 사용한다.  


테스트 결과에 따른 `TPS` 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-basic-tps.png)

응답시간에 따른 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-basic-rtot.png)

`RPS` 가 100을 넘지않는 5초 이전 그래프에서도 실패가 요청 수에 비례하게 증가함을 알 수 있다. 
그리고 `RPS` 가 200 까지 증가하는 10초 이후 구간에는 급격하게 실패가 증가함을 보여준다. 
현재 `Rate Limiting` 설정은 `10ms` 마다 하나의 요청으로 제한하고 있기 때문에, 
초당 100개의 요청이지만, `10ms` 이내에 1개이상 들어오는 요청은 실패됨을 알수 있다.  

응답시간 그래프는 비교적 일정한 수준의 결과를 보여주고 있다.  

`error.log` 를 확인하면 `Rate Limiting` 의 응답은 아래와 같이 에러 로그가 남게 된다. 

```
[error] 29#29: *100 limiting requests, excess: 1.000 by zone "mylimit", client: 192.168.65.3, server: localhost, request: "GET /status HTTP/1.1", host: "localhost:32222"
```  

그리고 응답 상태코드는 기본 503으로 내려 준다. 

```
192.168.65.3 - - "GET /status HTTP/1.1" 503 197 "-" "Apache-HttpClient/4.5.12 (Java/11.0.9)" "-"
```  

## Burst
만약 `100r/s` 로 `Rate Limiting` 이 설정된다면 1초에 100개의 요청이면서, 
`10ms` 당 1개의 요청 처리가 가능하다고 설명했었다. 
그런데 1초에는 100개의 요청이 들어왔지만, 
순간 `10ms` 에 2개의 요청이 들어온 상황이라면 하나의 요청은 `Rate Limting` 에 걸리게 된다. 
하지만 의도한 바는 최대 초당 100개의 요청을 처리하는 것이지만, 
위 상황에서는 초당 99개의 요청을 처리한 것이 된다.  

위와 같이 `10ms` 당 1개의 요청을 처리하는 설정에서 
버퍼를 둬서 `10ms` 당 1개 이상의 요청이 오더라도, 
정상적인 처리가 가능하도록 하는 것을 `Burst` 라고 한다.  

즉 `Burst` 설정을 통해 `Rate Limiting` 으로 설정한 최소 요청 간격보다 
빈번하게 요청이 오는 경우 초과 분에 대해서 `Burst` 에 넣어 두고 이후에 처리 할 수 있도록 하는 역할을 한다.  

```
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=100r/s;

server {
    location /test {
        limit_req zone=mylimit burst=100;

        proxy_pass http://backend;
    }
}
```  

`Burst` 는 위와 같이 `limit_req` 부분에 `burst=<개수>` 와 같이 설정할 수 있다. 

1. `10ms` 에 100개 요청 도착
    - 1개는 요청처리
    - 나머지 99개 요청은 쿠에 추가, `burst queue = 99'`
1. 다음 `10ms` 에 1개 요청 도착
    - `burst` 큐에 요청 하나 빼서 처리, `burst queue = 98`
    - 도착한 요청 큐에 추가, `burst queue = 99`
1. 다음 `10ms` 에 2개 요청 도착
    - `burst` 큐에 요청 하나 빼서 처리, `burst queue = 98`
    - 도착한 요청 큐에 추가, `burst queue = 100`
1. 다음 `10ms` 에 2개 요청 도착
    - `burst` 큐에 요청 하나 빼서 처리, `burst queue = 99`
    - 도착한 요청 하나만 큐에 추가, `burst queue = 100`
    - 도착한 나머지 요청은 `503` 에러
    
만약 `10ms` 에 101개의 요청이 온다고 하더라도, 
1개는 요청처리하고 나머지 100개는 `Burst` 에 넣어 두기 떄문에 에러는 발생하지 않는다. 
하지만 `10ms` 에 102` 개의 요청이 온다면, 
`Burst` 에 넣어야 하는 수가 101개로 설정된 100개를 초과하기 때문에 한개 요청에 대해서는 에러가 발생한다.  


### 테스트
테스트 전 아래 명령어로 이전에 실행한 `Pod` 를 꼭 삭제해 주어야 한다. 

```bash
$ kubectl delete pod <pod-name>
```  

사용하는 템플릿은 앞서 살펴본 `basic` 템플릿과 거의 비슷하고, 
`burst` 관련 설정과 템플릿 구분을 위해 이름 정도만 차이가 있다.  

- `burst-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: basic-configmap
  namespace: default
data:
  nginx.conf: |
    user  nginx;
    worker_processes  2;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        limit_req_zone $binary_remote_addr zone=mylimit:10m rate=100r/s;

        server {
            listen 80;
            server_name localhost;

            location /status {
                limit_req zone=mylimit burst=200;
                stub_status on;
                allow all;
                deny all;
            }
        }
    }
```  

- `burst-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: basic-rate-limiting
  labels:
    app: basic-rate-limiting
    type: rate-limiting
spec:
  containers:
    - name: basic-rate-limiting
      image: nginx:1.19
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
  volumes:
    - name: nginx-conf
      configMap:
        name: basic-configmap
        items:
          - key: nginx.conf
            path: nginx.conf
```  

```bash
$ kubectl apply -f burst-configmap.yaml
configmap/burst-configmap created
$ kubectl apply -f burst-pod.yaml
pod/burst-rate-limiting created
$ kubectl get configmap,pod,svc
NAME                             DATA   AGE
configmap/burst-configmap        1      17s

NAME                      READY   STATUS    RESTARTS   AGE
pod/burst-rate-limiting   1/1     Running   0          11s

NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/rate-limiting-service   NodePort    10.105.130.60   <none>        80:32222/TCP   67m
```  

테스트 결과에 따른 `TPS` 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-burst-tps.png)

응답시간에 따른 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-burst-rtot.png)

먼저 `TPS` 그래프를 보면 `basic` 테스트와 달리 실패 요청이 뒤늦게 등장한다. 
그리고 `TPS` 가 100을 넘어가는 6초 이후 구간에도 실패하지 않는다. 
이는 `Burst` 설정으로 인해 초과되는 요청은 큐를 통해 관리되고 있음을 확인 할 수 있다. 
즉 현재 `10ms` 당 1개의 요청을 처리하는 과정에서 초과하는 모든 요청을 `Burst` 를 통해 관리하지만 
`TPS` 가 140 이상되는 17, 18초 구간부터는 `Burst` 가 꽉차 에러 응답이 발생한다.  

하지만 응답시간 그래프를 보면 200 `RPS` 까지 증가하는 10초 이후 구간에서 응답시간이 갑자기 증가하는 것을 확인 할 수 있다. 
`10ms` 당 1개 요청처리에서 초과되는 요청들이 `Burst` 큐에 들어가고 다시 큐에서 나와 요청이 처리되기 까지 대기시간이 존재하기 때문이다.  

## Queueing with No Delay
앞서 살펴본 `Burst` 설정은 `Rate Liniting` 을 좀더 유동적으로 사용할수있도록 하는 설정이였다. 
하지만 테스트에서 확인 할 수 있듯이, `Burst` 를 사용하면 응딥시간에 큰 지연을 가져다 준다. 
응답 시간과 `Burst` 의 유동적인 설정을 동시에 사용할 수 있도록 하는 것이 `nodelay` 파라미터 이다. 
간단하게 `nodelay` 는 설정된 `Rate Limiting` 에서 한개 요청을 처리하고 다음 요청을 처리할떄 딜레이를 두지 않고, 
가능한 요청을 모두 처리하는 파라미터라고 할 수 있다.  

`nodelay` 파라미터는 아래와 같이 `limit_req` 부분에 선언해서 사용할 수 있다. 

```
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=100r/s;

server {
    location /test {
        limit_req zone=mylimit burst=100 nodelay;

        proxy_pass http://backend;
    }
}
```  

설정을 보면 1초에 100개의 요청 즉 `10ms` 에 1개 요청처리가 가능하지만, 
`Burst` 설정을 통해 100개의 큐 공간을 두고 `nodelay` 파라미터가 설정된 상태이다. 
위 설정에서 `nodelay` 의 동작에 대한 설정은 아래와 같다.  

1. 첫 `10ms` 이내 101개 요청 도착
    - 1개는 요청처리
    - 현재 큐 크기가 100개 이기 때문에 거용가능한 100개 요청 또한 즉시 요청처리
    - `burst queue = 100`(즉시 처리한 100개 요청 할당), 큐는 `10ms` 마다 가용공간이 1개씩 생겨남
1. 다음 `11ms` 이내 100개 요청 도착
    - 큐에는 1개의 공간만 생겼기 때문에 1개 요청 즉시 처리
    - `burst queue = 100` (즉시 처리한 1개 요청 할당)
    - 나머지 99개 요청은 `503` 에러
1. 다음 `51ms` 이내 100개 요청 도착
    - 큐에는 5개의 공간이 생겼기 때문에 5개 요청 즉시 처리
    - `burst queue = 100` (즉시 처리한 5개 요청 할당)
    - 나머지 95개 요청은 `503` 에러
    
`burst` 설정만 한 상태와, `burst` + `nodelay` 를 한 설정을 비교했을 때 차이에 대한 설명은 아래와 같다. 
설정한 `RPS` 를 넘어가는 요청 중 `burst` 큐의 가용 공간이 있을 때 `burst` 는 가용 공간만큼 요청을 큐에 추가하고, 
`burst` + `nodelay` 는 가용 공간만큼 요청을 즉시 처리하고 큐 가용공간을 즉시 처리한 만큼 줄인다. 
그리고 이후 설정된 `RPS` 의 주기에 따라 `burst` 는 큐에서 요청을 빼서 처리하고, 
`burst` + `nodelay` 는 주기에 따라 큐의 가용공간을 늘린다.  

### 테스트
사용하는 템플릿은 앞서 살펴본 `burst` 템플릿과 거의 비슷하고, 
`nodelay` 관련 설정과 템플릿 구분을 위해 이름 정도만 차이가 있다.  

- `nodelay-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nodelay-configmap
  namespace: default
data:
  nginx.conf: |
    user  nginx;
    worker_processes  2;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        limit_req_zone $binary_remote_addr zone=mylimit:10m rate=200r/s;

        server {
            listen 80;
            server_name localhost;

            location /status {
                limit_req zone=mylimit burst=100 nodelay;
                stub_status on;
                allow all;
                deny all;
            }
        }
    }
```  

- `nodelay-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodelay-rate-limiting
  labels:
    app: nodelay-rate-limiting
    type: rate-limiting
spec:
  containers:
    - name: nodelay-rate-limiting
      image: nginx:1.19
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
  volumes:
    - name: nginx-conf
      configMap:
        name: nodelay-configmap
        items:
          - key: nginx.conf
            path: nginx.conf
```  

```bash
$ kubectl apply -f nodelay-configmap.yaml
configmap/nodelay-configmap created
$ kubectl apply -f nodelay-pod.yaml
pod/nodelay-rate-limiting created
$ kubectl get configmap,pod,svc
NAME                             DATA   AGE
configmap/nodelay-configmap      1      33s

NAME                        READY   STATUS    RESTARTS   AGE
pod/nodelay-rate-limiting   1/1     Running   0          24s

NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/rate-limiting-service   NodePort    10.105.130.60   <none>        80:32222/TCP   1h02m

```  

테스트 결과에 따른 `TPS` 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-nodelay-tps.png)

응답시간에 따른 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-nodelay-rtot.png)

`TPS` 그래프를 보면 `Burst` 테스트와 같이 `RPS` 에 비례하게 증가하다가, 
`RPS` 가 200까지 증가하는 10초 이후 구간 부터 실패 응답이 발생하는 것을 알 수 있다. 
더 정확하게 말하면 `RPS` 가 140 이후 구간인 18초 부터 실패 응답이 발생한다. 
여기서 발생하는 실패응답은 `Burst` 큐의 가용공간이 없을때 들어온 요청에 대한 실패이다.  

응답시간 그래프는 `Burst` 테스트의 응답 그래프와 달리 `Basic` 테스트의 응답 그래프처럼 비교적 일정한 결과를 보여주고 있다. 
`Burst` 큐에 가용한 요청은 즉시 처리되기 때문에, `Burst` 테스트와 달리 `RPS` 를 초과하는 요청이 큐에서 대기하는 시간이 존재하지 않기 때문이다.  


## Two-Stage Rate Limiting
먼저 살펴본 `nodelay` 설정은 `Burst` 에 들어오는 모든 요청에 대해 지연없이 바로 요청을 전달하는 역할이다.  
`nodelay` 가 설정되지 않은 경우 응답시간이 예측 불가능할 정도로 늘어나는 상황이 발생할 수 있다. 
이런 상황을 어느정도 대응할 수 있는 방법이 `delay` 설정이다. 
현재 구성하는 서버에서 초당 요청 개수를 어느정도 예측 가능한 경우, 
보다 서버에 알맞는 `delay` 설정을 구성할 수 있다. 
아래는 평균적으로 초당 요청이 100개가 들어오고 특정 경우 최대 150개의 요청이 들어올 수 있을 때, 
`nodelay` 파라미터의 값에 150값을 주어 `limit_req` 부분에 선언해 사용할 수 있다.  

```
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=5r/s;

        server {
            listen 80;
            server_name localhost;

            location /status {
                limit_req zone=mylimit burst=12 delay=8;
                stub_status on;
                allow all;
                deny all;
            }
        }
```  

`delay` 는 서버에서 가용할 수 있는 최대 요청수설정해서 지연없이 처리 할 수있도록 하는 파라미터이다.  

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-burst-delay.png)

위 설정에 대해 설명하면 아래와 같다.
1. 첫 1초 이내 7개 요청 도착
    - `burst queue = 0`
    - 모든 7개 요청 지연없이 처리 
    - `burst queue = 7`(즉시 처리한 7개 요청 할당)
1. 다음 1초 이내 8개 요청 도착
    - `burst queue = 7`
    - 먼저 들어온 1개 요청은 지연없이 처리
    - 그리고 그 다음으로 들어온 4개의 요청에 대해서는 지연처리
    - `burst queue = 12`(즉시 처리 + 지연처리 요청 할당)
    - 나머지 3개 요청은 `503` 에러
1. 다음 1초 이내 6개 요청 도착
    - `burst queue = 7`
    - 먼저 들어온 5개 요청은 지연처리
    - `burst queue = 12`(지연처리 요청 할당)
    - 나머지 1개 요청은 `503` 에러


### 테스트
사용하는 템플릿은 앞서 살펴본 `nodelay` 템플릿과 거의 비슷하고, 
`delay` 관련 설정과 템플릿 구분을 위해 이름 정도만 차이가 있다.  

- `delay-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: delay-configmap
  namespace: default
data:
  nginx.conf: |
    user  nginx;
    worker_processes  2;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        limit_req_zone $binary_remote_addr zone=mylimit:10m rate=100r/s;

        server {
            listen 80;
            server_name localhost;

            location /status {
                limit_req zone=mylimit burst=200 delay=150;
                stub_status on;
                allow all;
                deny all;
            }
        }
    }
```  

- `delay-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: delay-rate-limiting
  labels:
    app: delay-rate-limiting
    type: rate-limiting
spec:
  containers:
    - name: delay-rate-limiting
      image: nginx:1.19
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
          readOnly: true
  volumes:
    - name: nginx-conf
      configMap:
        name: delay-configmap
        items:
          - key: nginx.conf
            path: nginx.conf
```  

```bash
$ kubectl apply -f delay-configmap.yaml
configmap/delay-configmap created
$ kubectl apply -f delay-pod.yaml
pod/delay-rate-limiting created
$ kubectl get configmap,pod,svc
NAME                             DATA   AGE
configmap/delay-configmap      1      33s

NAME                        READY   STATUS    RESTARTS   AGE
pod/delay-rate-limiting   1/1     Running   0          24s

NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/rate-limiting-service   NodePort    10.105.130.60   <none>        80:32222/TCP   2h02m
```  

테스트 결과에 따른 `TPS` 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-delay-tps.png)  

응답시간에 따른 그래프는 아래와 같다. 

![그림 1]({{site.baseurl}}/img/server/nginx-rate-limiting-delay-rtot.png)  

`TPS` 그래프를 보면 요청수가 150을 넘어가는 16, 17초 부터 에러 응답이 발생하는 것을 확인 할 수 있다. 
그리고 이후 `TPS` 가 200으로 증가하는 30초까지 성공 응답은 100선은 계속 유지하지만, 실패 응답은 계속해서 증가하게 된다.  

응답시간 그래프 또한 `TPS` 가 150을 넘어가는 16, 17초 부터 응답시간이 급격하게 증가하는 것을 확인 할 수 있다.  

`nodelay` 와 비교하면 `delay` 설정은 보다 디테일하게 `delay` 에 대한 설정을 할수 있다는 점에 차이가 있다. 
`nodelay` 는 가용한 `Burst` 사이즈에 대해 모두 `nodelay` 처리하기 때문에 정확하게 몇 `RPS` 까지 `nodelay` 처리가 되는지는 명확하지 않다. 
하지만 `delay` 는 `nodelay` 에 비해 요청 증가에 따라 응답시간, 실패응답이 더욱 큰 폭으로 증가할 수 있다. 
`Burst` 만 설정한 경우 응답시간이 너무 큰 폭으로 증가하고, `nodelay` 를 설정한 경우 한번에 너무 많은 요청이 애플리케이션 서버로 전달 될 수 있다. 
여기서 `delay` 파라미터를 통해 이러한 부분을 보다 명확하게 처리할 수 있다. 
예측되는 최대 요청수를 `delay` 설정해서 과도하게 애플리케이션 서버로 요청이 전달되는 것은 방어하면서, 
`delay` 이내의 모든 요청에 대해서는 빠르게 처리할 수 있다.  





---
## Reference
[Rate Limiting with NGINX and NGINX Plus](https://www.nginx.com/blog/rate-limiting-nginx/)  
[Limiting Access to Proxied HTTP Resources](https://docs.nginx.com/nginx/admin-guide/security-controls/controlling-access-proxied-http/)  
[Module ngx_http_limit_req_module](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html)  
	