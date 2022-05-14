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








---
## Reference
[Varnish HTTP Cache](https://varnish-cache.org/index.html)  
[Varnish 이야기](https://d2.naver.com/helloworld/352076)  
[How To Speed Up Static Web Pages with Varnish Cache Server on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-speed-up-static-web-pages-with-varnish-cache-server-on-ubuntu-20-04)  
    