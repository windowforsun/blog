--- 
layout: single
classes: wide
title: "[Varnish] HTTP Cache, Varnish"
header:
  overlay_image: /img/server-bg.jpg
excerpt: '웹 캐싱을 제공하는 HTTP 가속기 Varnish 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Server
tags:
  - Server
  - Varnish
  - HTTP Accelerator
  - Web Caching
  - Grace Mode
  - Load Balance
  - ESI
  - VCL
  - Health Check
  - Purge
toc: true
use_math: true
---  

## Intro
불특정 다수에게 공개된 웹 서비스를 운영하면 `DDos` 공격이나 갑작스럽게 트래픽이 폭주하는 경우가 있다. 
이런 트래픽 폭주를 대응하는 일반적인 방법은 미리 예측된 트래픽을 가용 할 수 있는 리소스를 할당해 놓는 방법일 것이다. 
하지만 이는 불필요한 리소스가 평상시에도 계속해서 점유하고 있다는 큰 단점을 가지고 있다. 
웹 서비스로 한정 해서 폭주하는 트래픽에 대한 장애 대응 방법으로 `Varnish` 를 사용할 수 있다.  

## Varnish
[Varnish](https://varnish-cache.org/index.html) 
는 `HTTP Accelerator` 의 한 종류로 웹 캐시 `Reverse Proxy` 오픈소스 소프트웨어 이다. 
`Varnish - origin server` 구조에서 웹 캐시는 말그대로 요청에 대한 응답을 캐싱해두고 동일한 요청이 왔을 때 
`origin server` 로 요청을 전달하지 않고 캐싱된 응답을 빠르게 전달하는 방식이다. 
요청한 `URL` 을 기준으로 캐시 데이터를 생성하고, 설정된 `TTL` 동안 캐시를 유지하게 된다. 
또한 필요한 경우 여러대의 `origin server` 를 로드밸런싱 하는 기능도 제공한다.   

### Varnish 특징
#### VCL(Varnish Configuration Language)
`Varnish` 는 설정을 위해 `VCL` 라고 하는 `DSL(Domain-Specific Language)` 를 사용한다. 
사용자는 `VCL` 문법에 맞게 원하는 설정을 다양한 조건으로 작성할 수 있다. 
작성된 설정파일은 최종적으로 `C언어` 를 통해 컴파일 된다. 
또한 여러 설정을 로드한 후 `Varnish` 실행 중에 설정을 변경하는 것도 가능하다.  

#### ESI(Edge-Side Include)
`CSS`, `JS` 파일은 정적인 이기때문에 간단하게 캐싱이 가능하다. 
하지만 `HTML` 의 경우 동적 웹페이지라면 동적인 요소로 캐싱이 어려울 수 있다. 
`Varnish` 는 `ESI` 라는 동적 컨텐츠를 여러 도작으로 분리해서 따로 캐싱하는 기술을 통해 이를 해결 한다. 
동적인 요소인 사용자 프로필은 캐싱하지 않고, 다른 정적인 부분만 캐싱하는 방법이다.  

#### purge
`purge` 는 캐싱된 데이터를 `TTL` 이 만료되지 전에 강제로 삭제하는 것을 의미한다. 
`Varnish` 는 아래와 같은 2가지 방법으로 `purge` 를 제공한다. 

- 특정 `URL` 를 지정해서 데이터 삭제
- 정규 표현식을 사용해서 해당하는 데이터가 사용되지 않도록 수행

첫 번째 방법은 실제로 데이터를 삭제하지만, 두 번째는 실제로 데이터를 삭제하지 않고 이를 `ban` 이라고 한다. 
설정된 정규 표현식에 해당하는 데이터를 사용되지 않도록 `ban` 검사를 수행하게 된다. 
정규 표현식을 추가할 때 데이터가 정규 표현식에 일치하는지 검사하는 것이 아니라, 
데이터를 검색 할때 정규 표현식에 해당하는지 검사해서 만족한다면 데이터를 사용하지 않는다.  

아래는 `ban` 의 예시로 `www.example.com` 의 요청에서 `png` 확장자는 캐시를 사용하지 않는다. 
`ban` 은 설정 전에 캐싱된 데이터에만 적용되고 이후 저장되는 캐시 데이터는 영향받지 않는다.  

```
ban req.http.host == "www.example.com" && req.url ~ "\\.png$"

or

varnishadm ban req.http.host == www.example.com '&&' req.url '~' '\\.png$'
```  

#### grace mode
`TTL` 은 해당 캐시의 유효시간을 의미하고, `TTL` 이 지나면 다시 원본 서버에 다시 요청해서 데이터를 가져온다. 
만약 원본 서버가 모두 장애인 상황이라면 `TTL` 이 지났더라도 캐싱된 데이터로 우선 서빙이 될 수 있다면 전면 장애의 대응 방법이 될 수 있다. 
위와 같은 경우에 사용할 수 있는 것이 바로 `grace mode` 이다.  

`grace mode` 는 동일 데이터에 대해서 하나의 요청만 원본 서버로 보낸다. 
그리고 `TTL` 이 만료된 이후에는 `grace time` 동안 캐시에 저장된 데이터를 사용하게 된다. 
이는 `TTL + grace time` 이 지난 후 캐시에서 데이터가 삭제 된다.  

#### load balancing
원본 서버에 대한 요청을 여러대에 나눠 보낼 수 있는 기능이다. 
`load balancing` 의 방법으로는 `random` ,`client`, `hash`, `round-robin`, `DNS`, `fallback` 등이 있다.  

#### 원본 서버 health check
설정된 주기마다 원본 서버에 `health check` 를 보내 생존 여부를 확인한다. 
만약 특정 원본 서버가 `health check` 를 실패한다면 해당 원본 서버로는 요청을 보내지 않는다. 
그리고 생존한 원본 서버가 없는 경우 `grace mode` 르 동작하게 된다.  

#### saint mode
`saint mode` 는 `TTL` 이 지난 데이터를 사용한다는 부분에선 `grace mode` 와 비슷하다. 
`saint mode` 는 원본 서버의 응답이 `500` 에러와 같이 정상적이지 않거나, 
특정 헤더를 응답하면 원본 서버의 응답을 사용하지 않고 캐싱 된 데이터를 사용하게 된다.  

#### 압축
`Varnish` 는 원본 서버의 응답이 압축되지 않은 경우 데이터를 압축해서 캐싱 할 수 있다. 
사용자 요청이 압축되지 않은 데이터를 요구하는 경우 압축으로 저장된 데이터를 푼 다음 응답한다. 
이런 압축 기능을 통해 동일한 공간에 더 많은 데이터를 저장할 수 있기 때문에 캐시 적중률(cache-hit ratio)을 높일 수 있다.  

#### 저장 공간
`Varnish` 는 설정파일에 저장 공간의 종류를 지정할 수 있다. 
저장 공간 종류로는 아래와 같은 것들이 있다.  

- malloc(메모리)
- file
- persistent

`malloc` 은 `malloc()` 함수를 통해 메모리에 저장하는 것을 의미하고, 
`file` 은 `mmap()` 함수로 메모리를 매핑해서 사용한다. 
지정하는 크기가 메인 메모리를 사용할 수 있는 크기라면 `malloc` 을 사용하고, 
그 외의 경우에만 `file` 를 사용하는게 좋다.  

`Varnish` 재실행 시에도 데이터가 남아 있는 것을 원한다면, `persistent` 를 사용해야 한다. 
`persistent` 는 공간이 부족할 때 기존 `LRU` 가 아닌 `FIFO` 로 `victim` 이 선택 된다.  


### VCL Flow
사용자의 `Varnish` 설정은 웹 요청을 받아 처리하는 중간 과정에 `VCL` 로 작성된 동작을 수행하는 것을 의미한다. 
아래는 `Varnish` 가 웹 요청을 처리하는 흐름을 도식화 한 것이다.  

![그림 1]({{site.baseurl}}/img/server/practice-varnish-intro-1.svg)  

> [Varnish Finite State Machine](https://book.varnish-software.com/4.0/chapters/VCL_Basics.html#varnish-finite-state-machine)

위 그림을 통해 크게 아래 두가지 흐름으로 나눌 수 있다.  

1. 캐시 존재 : ` vcl_recv() -> vcl_hash() -> vcl_hit() -> vcl_deliver()`
1. 캐시 존재 X : `vcl_recv() -> vcl_hash() -> vcl_miss() -> vcl_pass() -> vcl_backend_fetch() -> vcl_backend_response() -> vcl_deliver()`

위 그림에서 원으로 그려진 부분들이 모두 `VCL` 함수이다. 
사용자가 따로 재정의 하지 않으면 모두 기본 동작이 수행된다. 
만약 사용자가 특정 함수를 재정의 하게 되면, 사용자가 재정한 함수가 먼저 실행되고 그 다음 기본 동작이 수행된다. 
사용자가 재정의한 함수에 `return` 이 존재하는 경우 기본 기능은 수행되지 않는다.  

`VCL` 함수는 `deliver`, `fetch,` `pass`, `pipe`, `lookup` 등을 리턴할 수 있다. 
`VCL` 함수에 따른 리턴 가능한 종류는 [여기](https://book.varnish-software.com/4.0/chapters/VCL_Basics.html#legal-return-actions)
에서 더 자세히 확인 가능하다.  

> 리턴 값 설명  
> 함수 설명  

### 기본 TTL 설정 값
[상세](https://docs.varnish-software.com/tutorials/object-lifetime/)

TTL|기본 설정 값
---|---
default_grace|10s
default_keep|0s
default_ttl|120s

### 예제

앞서 언급한 몇가지 `Varnish` 특징들을 어떻게 적용하는 지에 대한 부분을 중점을 예제를 진행한다.  

#### 기본구성
가장 먼저 `Varnish` 와 `원본 서버`를 하나씩 구성해서 `원본 서버` 의 응답을 캐싱하는 간단한 예제를 진행한다.  

예제환경 구성은 `Docker` 와 `docker-compose` 를 사용한다. 
로컬에 미리 두가지 환경 구성이 필요하다.  

`원본 서버` 는 `Python` 을 사용해서 간단한 응답을 주는 `HTTP Server` 를 구성한다. 

```python
# server/httpd.py

from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import os


class JsonServer(BaseHTTPRequestHandler):
	def do_GET(self):
		self.send_response(200)
		self.send_header("Content-type", "application/json")
		self.end_headers()
		res = "hello world(" + os.getenv("HOSTNAME", "") + ")"
		print("res : " + res)
		self.wfile.write(json.dumps({"message": res}).encode("UTF-8"))


def run(server_class=HTTPServer, handler_class=BaseHTTPRequestHandler):
	server_address = ("", 8000)
	httpd = server_class(server_address, handler_class)

	print(f"Starting http on port {server_address[1]}...")
	httpd.serve_forever()


if __name__ == "__main__":
	run(handler_class=JsonServer)
```  

위 파이썬 파일을 사용해서 이미지를 생성하는 `Dockerfile` 내용은 아래와 같다. 

```dockerfile
# server/Dockerfile

FROM python:3.8-slim-buster

COPY httpd.py /usr/src/httpd.py

WORKDIR /usr/src

CMD ["python", "-u", "httpd.py"]
```  

기본구성에 사용할 `Varnish` 설정 파일인 `VCL` 파일은 아래와 같다.  

```
# varnish/basic.vcl

vcl 4.0;

backend default {
  .host = "varnish-be-1";
  .port = "8000";
}

sub vcl_deliver {
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
    return (deliver);
}
```  

캐시에 값이 존재하면 `X-Cache` 해더에 `HIT` 를 설정하고, 
존재하지 않으면 `MISS` 를 설정 한다.  

위 `VCL` 파일을 사용해서 `Varnish` 도커 이미지를 생성하는 `Dockerfile` 내용은 아래와 같다.  

```dockerfile
# varnish/Dockerfile

FROM varnish:6.6

RUN apt -y update
RUN apt -y install curl

COPY basic.vcl /etc/varnish/default.vcl
```  

최종적으로 앞서 설명한 파일들을 사용해서 컨테이너의 구성과 실행에 대한 템플릿인 `docker-compose` 파일은 아래와 같다.  

```yaml
# docker-compose.yaml

version: '3.7'

services:
	varnish-be-1:
		container_name: varnish-be-1
		hostname: varnish-be-1
		build: ./server
		ports:
			- 8001:8000
	varnish:
		container_name: varnish
		hostname: varnish
		build: ./varnish
		ports:
			- 80:80
```  

이제 `docker-compose.yaml` 파일이 위치한 경로에서 아래 명령어를 사용해서 템플릿을 실행해 준다.  

```bash
$ docker-compose up --build
Building varnish-be-1

.. 생략 ..

Successfully built 2599cd83a410
Successfully tagged docker_varnish-be-1:latest
Building varnish

.. 생략 ..

Successfully built 2788a5082441
Successfully tagged docker_varnish:latest
Starting varnish-be-1 ... done
Starting varnish      ... done
Attaching to varnish, varnish-be-1
varnish-be-1    | Starting http on port 8000...
varnish         | Warnings:
varnish         | VCL compiled.
varnish         |
varnish         | Debug: Version: varnish-6.6.2 revision 17c51b08e037fc8533fb3687a042a867235fc72f
varnish         | Debug: Platform: Linux,5.10.102.1-microsoft-standard-WSL2,x86_64,-junix,-smalloc,-sdefault,-hcritbit
varnish         | Debug: Child (21) Started
varnish         | Info: Child (21) said Child starts
```  

아래와 같이 `curl` 명령어로 요청을 한번 모내면 `X-Cache: MISS` 를 확인 할 수 있다. 
이는 `원본 서버` 로 요청이 전달되고 그 응답을 캐싱하고 받은 것이다.  

```bash
$ curl -v 172.26.85.158/test
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 09:48:05 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 2
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

다시 한번 요청을 보내면 `Age: 32` 로 증가해 있고, `X-Cache: HIT` 인 것을 확인 할 수 있다.  
이는 `원본 서버` 에 요청 전달없이 캐시에 존재하는 데이터르 바로 응답해 준것이다.  

```bash
$ curl -v 172.26.85.158/test
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 09:48:05 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 32770 3
< Age: 32
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: HIT
< Accept-Ranges: bytes
< Content-Length: 40
< Connection: keep-alive
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

현재 `Varnish` 에 `TTL` 에 대한 별도의 설정을 하지 않았기 때문에, 
기본 `TTL` 은 `120s` 이다. 
`TTL` 만료 시간 이후에 다시 요청을 보내면 아래와 같이 캐시가 만료되고 다시 `원본 서버로` 요청을 보낸 것을 확인 할 수 있다.  

```bash
$ curl -v 172.26.85.158/test
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 09:54:30 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 5
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

`Varnish` 의 캐시키는 요청 `URL` 과 `HTTP Request` 에 포함된 `Host` 헤더 혹은 `IP` 주소를 조합해서 사용한다. 
그러므로 기존에 테스트에 사용한 `/test` 가 아닌 `/test2` 와 같이 다른 `URL` 은 별도의 키로 캐싱 된다.  


#### purge 및 ban 적용, 사용하기
`purge` 를 사용하기 위해서는 아래와 같이 `VCL` 파일에 설정 추가가 필요하다. 
`purge` 는 `URL` 단위로 `TTL` 이전에 캐시를 삭제하는 동작이다. 

```
# varnish/purge-ban.vcl

vcl 4.0;

acl purge {
    "localhost";
}

backend default {
  .host = "varnish-be-1";
  .port = "8000";
}

sub vcl_recv {
    if (req.method == "PURGE") {
        if (client.ip ~ purge) {
            return (purge);
        } else {
            return(synth(405, "Now Allowed."));
        }

    }
}

sub vcl_deliver {
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
    return (deliver);
}
```  

기존 `basic.vcl` 에서 `purge-ban.vcl` 은 `acl`, `vcl_recv` 블럭이 추가 됐다. 
`acl` 과 `vcl_recv` 설정을 통해 `purge` 동작은 `localhost` 에서 요청한 경우에만 수행이 가능하도록 했다. (일반 사용자의 `purge` 요청이 수행되면 안되기 때문.)   

기존 `varnish/Dockerfile` 을 아래와 같이 수정해 준다.  

```dockerfile
# ..생략.. 

COPY purge-ban.vcl /etc/varnish/default.vcl
```  

`docker-compose.yaml` 을 실행해 준다. 
그리고 기존 처럼 `/test` 로 요청을 보낸후, 
`/test` 경로에 대해서 `purge` 를 수행하면 아래와 같이 `405` 에러가 발생하는 것을 확인 할 수 있다.  

```bash
$ curl -v 172.26.85.158/test
.. 생략 .. 

< Age: 2
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: HIT
{"message": "hello world(varnish-be-1)"}

$ curl -X PURGE -v 172.26.85.158/test

.. 생략 ..

< HTTP/1.1 405 Now Allowed.
<
<!DOCTYPE html>
<html>
  <head>
    <title>405 Now Allowed.</title>
  </head>
  <body>
    <h1>Error 405 Now Allowed.</h1>
    <p>Now Allowed.</p>
    <h3>Guru Meditation:</h3>
    <p>XID: 32773</p>
    <hr>
    <p>Varnish cache server</p>
  </body>
</html>
* Connection #0 to host 172.26.85.158 left intact
```  

아래 명령어로 직접 `varnish` 컨테이너에 접속해 `purge` 요청을 수행하면 정상 동작하는 것을 확인 할 수 있다.  

```bash
$ docker exec -it varnish /bin/bash
root@varnish:/etc/varnish# curl -X PURGE -v localhost/test
*   Trying 127.0.0.1:80...
* Connected to localhost (127.0.0.1) port 80 (#0)
> PURGE /test HTTP/1.1
> Host: localhost
> User-Agent: curl/7.74.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 Purged
< Date: Sun, 15 May 2022 10:08:48 GMT
< Server: Varnish
< X-Varnish: 32775
< Content-Type: text/html; charset=utf-8
< Retry-After: 5
< Content-Length: 240
< Accept-Ranges: bytes
< Connection: keep-alive
<
<!DOCTYPE html>
<html>
  <head>
    <title>200 Purged</title>
  </head>
  <body>
    <h1>Error 200 Purged</h1>
    <p>Purged</p>
    <h3>Guru Meditation:</h3>
    <p>XID: 32775</p>
    <hr>
    <p>Varnish cache server</p>
  </body>
</html>
* Connection #0 to host localhost left intact
```  

다시 컨테이너 외부에서 `/test` 경로를 요청하면 다시 `원본 서버` 로 부터 요청해 캐싱하는 것을 확인 할 수 있다.  

```bash
$ curl -v 172.26.85.158/test
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:09:29 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 7
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

다음으로 `ban` 테스트를 위해 `.png` 로 끝나는 몇개 요청을 수행해 준다. 
`ban` 은 기존 캐싱된 데이터 중에서 정규식과 매핑되는 캐시를 사용하지 않도록 하는 동작이다. 
즉 `ban` 명령이 수행된 이후 캐시 데이터 부터는 영향을 받지 않는다.  

```bash
$ curl -v 172.26.85.158/test1.png
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test1.png HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:13:27 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 32777
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
$ curl -v 172.26.85.158/test2.png
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test2.png HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:13:30 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 10
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
$ curl -v 172.26.85.158/test3.png
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test3.png HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:13:33 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 13
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

그 이후 `varnish` 컨테이너에 접속해 `varnishadm ban req.http.host == 172.26.85.158 '&&' req.url '~' '\\.png$'` 명령을 수행해 준다.  

```bash
$ docker exec -it varnish /bin/bash
root@varnish:/etc/varnish# varnishadm ban req.http.host == 172.26.85.158 '&&' req.url '~' '\\.png$'
200
```  

그리고 이후 다시 동일한 요청을 보내면 기존 캐시를 사용하지 않고, 
`X-Cache: MISS` 로 다시 `원본 서버` 로 부터 데이터를 가져오는 것을 확인 할 수 있다.  

```bash
$ curl -v 172.26.85.158/test1.png
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test1.png HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:16:36 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 65543
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
$ curl -v 172.26.85.158/test2.png
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test2.png HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:16:39 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 32791
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
$ curl -v 172.26.85.158/test3.png
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test3.png HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:16:41 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 32794
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

#### grace mode
`grace mode` 는 `원본 서버` 에 문제가 있었을 때 `TTL` 이 만료된 이후에도 `grace` 에 설정된 
시간동안 캐시를 사용해서 응답을 주는 기능이다.  

`grace mode` 설정을 위해 아래와 같이 설정을 추가해 준다.  

```
# varnish/gracemode.vcl

vcl 4.0;

backend default {
  .host = "varnish-be-1";
  .port = "8000";
}

sub vcl_backend_response {
    # set custom ttl
    set beresp.ttl = 11s;
    set beresp.grace = 30s;
    set beresp.keep = 10m;

    return (deliver);
}
sub vcl_deliver {
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
    return (deliver);
}
```  

기존 `basic.vcl` 설정에서 `TTL`, `grace`, `keep` 설정을 커스텀하게 수정했다. 
캐시는 `ttl` 설정에 따라 `11초` 동안 사용되고, 
`원본 서버` 에 문제가 있는 경우 `ttl` 시간 이후 `grace` 설정 값인 `30s` 동안 기존 캐시를 사용하게 된다. 
그리고 `grace mode` 를 위해 이전 캐시를 유지시키는 시간은 `keep` 설정 값인 `10m` 이다.  

`docker-compose` 로 템플릿을 실행 시킨다. 
그리고 `/test` 경로로 요청을 보내면 `Varnish` 에 캐싱이 수행된다.  

```bash
$ curl -v 172.26.85.158/test
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:44:33 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 2
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

그 후 도커 명령어로 `원본 서버`(`varnish-be-1`) 컨테이너를 중지 시킨 후 다시 요청을 보내면 
`Age` 가 설정된 `TTL` 인 `11` 을 지났지만 정상 응답이 수행 되는 것을 확인 할 수 있다.  

```bash
$ docker stop varnish-be-1
varnish-be-1
$ curl -v 172.26.85.158/test
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:44:33 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 32777 3
< Age: 40
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: HIT
< Accept-Ranges: bytes
< Content-Length: 40
< Connection: keep-alive
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
```  

하지만 `TTL` + `grace`(=41s) 가 넘어간 이후 부터는 `grace mode` 도 사용되지 않기 때문에 
아래와 같이 `503` 에러가 응답 되는 것을 확인 할 수 있다.  

```bash
$ curl -v 172.26.85.158/test
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 503 Backend fetch failed
< Date: Sun, 15 May 2022 10:45:16 GMT
< Server: Varnish
< Content-Type: text/html; charset=utf-8
< Retry-After: 5
< X-Varnish: 98306 32778
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: HIT
< Content-Length: 282
< Connection: keep-alive
<
<!DOCTYPE html>
<html>
  <head>
    <title>503 Backend fetch failed</title>
  </head>
  <body>
    <h1>Error 503 Backend fetch failed</h1>
    <p>Backend fetch failed</p>
    <h3>Guru Meditation:</h3>
    <p>XID: 32778</p>
    <hr>
    <p>Varnish cache server</p>
  </body>
</html>
* Connection #0 to host 172.26.85.158 left intact
```  


#### load balancing, health check
이번에는 `원본 서버` 1대를 더 추가해서 `load balancing` 을 사용해 보고, 
각 `원본 서버` 에 `health check` 도 추가해 본다.  

`VCL` 설정 파일을 아래와 같이 수정해 준다.  

```
# varnish/loadbalancing-healthcheck.vcl
vcl 4.0;

import directors;

backend server1 {
  .host = "varnish-be-1";
  .port = "8000";
  .probe = {
    .url = "/";
    .timeout = 1s;
    .interval = 5s;
    .window = 5;
    .threshold = 3;
  }
}

backend server2 {
   .host = "varnish-be-2";
   .port = "8000";
   .probe = {
     .url = "/";
     .timeout = 1s;
     .interval = 5s;
     .window = 5;
     .threshold = 3;
   }
}

sub vcl_init {
    new balancer = directors.round_robin();
    balancer.add_backend(server1);
    balancer.add_backend(server2);
}

sub vcl_recv {
    set req.backend_hint = balancer.backend();
}

sub vcl_deliver {
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
    return (deliver);
}
```  

기존 `basic.vcl` 에서 `directors` 를 임포트 했고, 
`default` 백엔드서버를 사용하는 것이 아니라 `server1, `server2` 로 이름을 붙여 주었다. 
그리고 각 백엔드서버 설정에는 `.probe` 를 사용해서 `health check` 설정을 추가했다.  

앞서 설정한 `server1`, `server2` 를 `load balancing` 하기 위해서 
`vcl_init` 함수에 `directors.round_robin()` 로 `Round Robin` 방식으로 설정 했다. 
그리고 `add_backend()` 를 사용해서 `server1`, `server2` 를 추가해 주었다. 
마지막으로 `vcl_recv` 함수에 `set req.backend_hint = balancer.backend()` 를 사용해서 
선언한 백엔드서버에 요청이 전달 될 수 있도록 설정했다.  

`docker-compose.yaml` 파일도 아래와 같이 `varnish-be-2` 를 하나 추가한 구성으로 수정한다.  

```yaml
# docker-compose.yaml

version: '3.7'

services:
  varnish-be-1:
    container_name: varnish-be-1
    hostname: varnish-be-1
    build: ./server
    ports:
      - 8001:8000
  varnish-be-2:
    container_name: "varnish-be-2"
    hostname: "varnish-be-2"
    build: ./server
    ports:
      - 8002:8000
  varnish:
    container_name: "varnish"
    hostname: "varnish"
    build: ./varnish
    ports:
      - 80:80
```  

`docker-compose` 를 실행하고 로그를 보면 주기적으로 `health check` 요청이 수행되는 것을 확인 할 수 있다.  

```bash
$ docker-compose up --build

.. 생략 ..

varnish-be-1    | 172.21.0.4 - - [15/May/2022 10:56:29] "GET / HTTP/1.1" 200 -
varnish-be-2    | 172.21.0.4 - - [15/May/2022 10:56:29] "GET / HTTP/1.1" 200 -
varnish-be-1    | res : hello world(varnish-be-1)
varnish-be-2    | res : hello world(varnish-be-2)
varnish-be-1    | 172.21.0.4 - - [15/May/2022 10:56:34] "GET / HTTP/1.1" 200 -
varnish-be-1    | res : hello world(varnish-be-1)
varnish-be-2    | 172.21.0.4 - - [15/May/2022 10:56:34] "GET / HTTP/1.1" 200 -
varnish-be-2    | res : hello world(varnish-be-2)
varnish-be-2    | 172.21.0.4 - - [15/May/2022 10:56:39] "GET / HTTP/1.1" 200 -
varnish-be-1    | 172.21.0.4 - - [15/May/2022 10:56:39] "GET / HTTP/1.1" 200 -
varnish-be-2    | res : hello world(varnish-be-2)
varnish-be-1    | res : hello world(varnish-be-1)
varnish-be-2    | 172.21.0.4 - - [15/May/2022 10:56:44] "GET / HTTP/1.1" 200 -
varnish-be-1    | 172.21.0.4 - - [15/May/2022 10:56:44] "GET / HTTP/1.1" 200 -
varnish-be-1    | res : hello world(varnish-be-1)
varnish-be-2    | res : hello world(varnish-be-2)
varnish-be-2    | 172.21.0.4 - - [15/May/2022 10:56:49] "GET / HTTP/1.1" 200 -
varnish-be-2    | res : hello world(varnish-be-2)
varnish-be-1    | 172.21.0.4 - - [15/May/2022 10:56:49] "GET / HTTP/1.1" 200 -
varnish-be-1    | res : hello world(varnish-be-1)
```  

`/test1`, `/test2` 처럼 여러 요청을 보내면 응답이 `varnish-be-1`, `varnish-be-2` 로 번갈악 가며 응답에 `hostname` 이 찍히는 것을 확인 할 수 있다.  

```bash
$ curl -v 172.26.85.158/test1
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test1 HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:57:48 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 2
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
$ curl -v 172.26.85.158/test2
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test2 HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:57:49 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 32770
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-2)"}
$ curl -v 172.26.85.158/test3
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test3 HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:57:52 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 5
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-1)"}
$ curl -v 172.26.85.158/test4
*   Trying 172.26.85.158:80...
* TCP_NODELAY set
* Connected to 172.26.85.158 (172.26.85.158) port 80 (#0)
> GET /test4 HTTP/1.1
> Host: 172.26.85.158
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: BaseHTTP/0.6 Python/3.8.13
< Date: Sun, 15 May 2022 10:57:53 GMT
< Content-type: application/json
< Vary: Accept-Language
< X-Varnish: 32773
< Age: 0
< Via: 1.1 varnish (Varnish/6.6)
< X-Cache: MISS
< Accept-Ranges: bytes
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 172.26.85.158 left intact
{"message": "hello world(varnish-be-2)"}
```  


---
## Reference
[Varnish HTTP Cache](https://varnish-cache.org/index.html)  
[Varnish 이야기](https://d2.naver.com/helloworld/352076)  
[How To Speed Up Static Web Pages with Varnish Cache Server on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-speed-up-static-web-pages-with-varnish-cache-server-on-ubuntu-20-04)  
[Purging and banning](https://varnish-cache.org/docs/6.6/users-guide/purging.html)  
[Grace mode and keep](https://varnish-cache.org/docs/trunk/users-guide/vcl-grace.html)  
[Backend servers](https://varnish-cache.org/docs/6.6/users-guide/vcl-backends.html?highlight=load%20balance)  
    