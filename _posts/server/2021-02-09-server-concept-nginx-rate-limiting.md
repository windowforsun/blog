--- 
layout: single
classes: wide
title: ""
header:
  overlay_image: /img/server-bg.jpg
excerpt: ''
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

## Rate Limiting 설정
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

설명 더~~~

## 이거 테스트코드 및 결과~~~









---
## Reference
[Rate Limiting with NGINX and NGINX Plus](https://www.nginx.com/blog/rate-limiting-nginx/)  
[Limiting Access to Proxied HTTP Resources](https://docs.nginx.com/nginx/admin-guide/security-controls/controlling-access-proxied-http/)  
[Module ngx_http_limit_req_module](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html)  
	