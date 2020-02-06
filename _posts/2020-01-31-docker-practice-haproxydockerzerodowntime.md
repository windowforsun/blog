--- 
layout: single
classes: wide
title: "[Docker 실습] HAProxy 를 사용한 Web Application 무중단 배포"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'HAProxy 의 Load Balancing 기능과 Swarm 의 rolling update 기능으로 무중단 배포를 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Load Balancing
  - HAProxy
  - Docker
  - Spring
  - Zero-Downtime
---  

## Web Application
- 사용할 애플리케이션은 `windowforsun/test-web` 이미지를 사용한다.
- Spring boot 기반이고 제공하는 API 관련 코드는 아래와 같다.

	```java
	@RestController
	public class HelloController {
	    @GetMapping("/")
	    public String root() throws Exception{
	        return InetAddress.getLocalHost().getHostName() + ", current time : " + System.currentTimeMillis();
	    }
	
	    @GetMapping("/{time}")
	    public String root(@PathVariable int time) throws Exception {
	        Thread.sleep(time);
	        return InetAddress.getLocalHost().getHostName() + ", current time : " + System.currentTimeMillis();
	    }
	
	    @GetMapping("/healthcheck")
	    public String healthcheck() throws Exception {
	        return "ok";
	    }
	
	    @GetMapping("/healthcheck/{time}")
	    public String healthcheck(@PathVariable int time) throws Exception{
	        Thread.sleep(time);
	
	        return "ok";
	    }
	}
	```  
	
	- `/` 로 `GET` 요청을 보내면 응답으로 애플리케이션이 구동중인 호스트의 이름과 timestamp 관련 문자열을 응답한다.
	- `/healthcheck` 로 `GET` 요청을 보내면 ok 문자열을 응답한다.
	- 2가지 API 모두 sleep 을 시킬 수 있는 API 가 추가로 구성돼 있다.

## HAProxy 를 이용한 간단한 무중단 배포
- 앞단에서 Load Balancing 역할을 수행하는 `HAProxy` 는 [DockerCloud HAProxy](https://hub.docker.com/r/dockercloud/haproxy/) 이미지를 사용한다.
- `docker-compose` 파일에서 간단한 설정을 통해 `HAProxy` 를 사용할 수 있다는 장점이 있다.
- `/var/run/docker.sock` 을 이미지에 마운트 해서 사용하기 때문에, `HAProxy` 컨테이너에서 네트워크를 참여하는 컨테이너를 감지해 역할을 수행한다.
- 무중단 배포를 위해서 Web Application 은 최소 2개로 구성해야 한다.
- `HAProxy` 컨테이너에서 자동으로 네트워크에 참여하는, 나가는 컨테이너를 감지해 주기 때문에 모든 Web Application 컨테이너가 중지되는 일만 없다면 무중단으로 배포가 가능하다.
- 예제의 방법은 `HAProxy` 의 Load Balancing 기능과 `docker-compose` 에서 `update_config` 를 사용하는 방법 이기 때문에, 서비스 `scale` 의 수와 실제로 WebApplication 이 올라가는 시간이 `update_config` 설정 값에 큰 영향을 줄 수 있다.
	- `update_config` 을 사용하는 것을 `swarm rolling update` 이라고 한다. 관련 설명은 [여기서](https://docs.docker.com/engine/swarm/swarm-tutorial/rolling-update/) 확인 가능하다.

- 구성 그림

### Docker 구성

- 구성된 `docker-compose.yml` 파일은 아래와 같다.

	```yaml
	version: '3.7'

	services:
	  webapp:
	    hostname: "app-{{.Task.Slot}}"
	    image: windowforsun/test-web:latest
	    deploy:
	      replicas: 3
	      # 서비스 업데이트시에 적용되는 설정으로 1개씩 10초 간격으로 업데이트를 수행한다.
	      update_config:
	        parallelism: 1
	        delay: 10s
	    networks:
	      - haproxydocker-net
	    ports:
	      - 8080
	    environment:
	    	# HAProxy 연결할 Web Application 포트
	      - SERVICE_PORTS=8080
	
	  haproxy:
	    image: dockercloud/haproxy
	    environment:
	    	# Load Balancing 에서 사용할 알고리즘
	      - BALANCE=leastconn
	    volumes:
	    	# 호스트의 docker.sock 을 마운트 시켜 네트워크를 감지할 수 있도록 설정
	      - /var/run/docker.sock:/var/run/docker.sock
	    ports:
	      - 80:80
	    depends_on:
	      - webapp
	    networks:
	      - haproxydocker-net
	
	networks:
	  haproxydocker-net:
	    driver: overlay
    	attachable: true
	```  
	
	- `webapp` 서비스에서 `SERVICE_PORTS` 환경 변수는 쉼표를 구분자로 여러개 설정도 가능하다.
	- `webapp` 서비스의 hostname 은 `app-{{.Task.Slot}}` 으로 이름과 클러스터링 번호로 구성한다.
	- `haproxy` 서비스에서 `BALANCE` 환경 변수의 기본 값은 `roundrobin` 이다.
- `docker swarm` 사용을 위해 `docker swarm init` 명령어로 활성화 시켜준다.

	```
	$ docker swarm init
	Swarm initialized: current node (imujahdcy81us6wtrpvy5d80l) is now a manager.
	
	To add a worker to this swarm, run the following command:
	
	    docker swarm join --token SWMTKN-1-0o2onow6a9r346kqauehr2pewf62ud0dm4cc5jxqld55c9og5c-dmf6e36eaj1a8ogvcxpzut1c0 10.0.2.15:2377
	
	To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
	```  
	
- `docker-compose.yml` 파일이 위치한 디렉토리와 동일한 위치에서 아래 명령어로 실행 시킨다.

	```
	$ docker stack deploy -c docker-compose.yml haproxydocker
	Creating network haproxydocker_haproxydocker-net
	Creating service haproxydocker_haproxy
	Creating service haproxydocker_webapp
	```  
	
- 실행 중인 서비스를 확인하면 아래와 같다.

	```
	$ docker service ls
	ID                  NAME                    MODE                REPLICAS            IMAGE                          PORTS
	acevu0dwfbk2        haproxydocker_haproxy   replicated          1/1                 dockercloud/haproxy:latest     *:80->80/tcp
	rbntiezhnwzd        haproxydocker_webapp    replicated          3/3                 windowforsun/test-web:latest   *:30000->8080/tcp
	```  
	
- `curl http://127.0.0.1` 로 요청을 보내면 아래와 같은 결과가 나온다.

	```
	$ curl http://127.0.0.1
	app-2, current time : 1580438651577
	$ curl http://127.0.0.1
	app-3, current time : 1580438653377
	$ curl http://127.0.0.1
	app-1, current time : 1580438653933
	$ curl http://127.0.0.1
	app-2, current time : 1580438654512
	$ curl http://127.0.0.1
	app-3, current time : 1580438655127
	$ curl http://127.0.0.1
	app-1, current time : 1580438655699
	$ curl http://127.0.0.1
	app-2, current time : 1580438656231
	```  
	
- `webapp` 서비스의 이미지를 `windowforsun/test-web:v1` 으로 수정하는 명령어는 `docker service update --image windowforsun/test-web:v1 haproxydocker_webapp` 이다.
- 위 명령어로 `webapp` 서비스의 이미지를 업데이트 시킬 때 요청 테스트를 위해 먼저 이미지를 업데이트 해준다.
- 이미지를 업데이트 하는 동안 계속 요청을 보내면 아래와 같은 결과가 나온다.

	```
	$ curl http://127.0.0.1
	app-1, current time : 1580438960592
	$ curl http://127.0.0.1
	app-2, current time : 1580438961362
	$ curl http://127.0.0.1
	app-3, current time : 1580438962712
	
	# app-2 업데이트 시작
	
	$ curl http://127.0.0.1
	app-1, current time : 1580438963576
	$ curl http://127.0.0.1
	app-1, current time : 1580438964632
	$ curl http://127.0.0.1
	app-3, current time : 1580438965609
	$ curl http://127.0.0.1
	
	# 반복
	# app-2 업데이트 완료
	
	$ curl http://127.0.0.1
	app-2, current time : 1580438985517
	$ curl http://127.0.0.1
	app-3, current time : 1580438986336
	$ curl http://127.0.0.1
	app-1, current time : 1580438987281
	$ curl http://127.0.0.1
	
	# app-1 업데이트 시작
	
	$ curl http://127.0.0.1
	app-2, current time : 1580438988143
	$ curl http://127.0.0.1
	app-3, current time : 1580438989072
	
	# 반복
	# app-1 업데이트 완료
	# app-3 업데이트 시작
	
	$ curl http://127.0.0.1
	app-1, current time : 1580439010019
	$ curl http://127.0.0.1
	app-2, current time : 1580439010881
	
	# 반복
	# app-3 업데이트 완료
	
	$ curl http://127.0.0.1
	app-1, current time : 1580439033560
	$ curl http://127.0.0.1
	app-3, current time : 1580439034693
	$ curl http://127.0.0.1
	app-2, current time : 1580439035747
	$ curl http://127.0.0.1
	app-1, current time : 1580439036736
	$ curl http://127.0.0.1
	app-3, current time : 1580439037704
	$ curl http://127.0.0.1
	```  

---
## Reference

	