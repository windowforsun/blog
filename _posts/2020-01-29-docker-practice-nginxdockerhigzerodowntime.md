--- 
layout: single
classes: wide
title: "[Docker 실습] Nginx 를 사용한 Web Application 무중단 배포"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Nginx 의 Load balancing 을 사용해서 Docker 무중단 배포를 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Load Balancing
  - Nginx
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


## Nginx backup 설정을 이용한 무중단 배포
- 아주 간단한 방식으로 Nginx 의 local balancer 설정중 backup 을 이용한 방법이다.
- Docker 배포가 진행되는 동안 서비스가 불가능 한 시점에 backup 으로 설정된 서버를 사용해 지속적으로 서비스를 제공될 수 있도록 하는 방식이다.
- backup 서버를 별도로 두어야 한다는 단점이 있고, 비용적인 문제로 backup 서버를 상용 서버와 비슷하게 두지 못한 상황에 따른 장애 등 문제점들이 발생 할 수 있다.
- 위와 같이 비용이 2배로 들수 있지만, 롤백이나 여타 상황에 대한 장점이 부족한 방식이다.
- 지속적인 서비스 제공을 위해 배포를 상용 서비스, 백업 서비스 각각 해주어야 한다.

- 구성 그림

### Docker 구성

![그림 1]({{site.baseurl}}/img/docker/practice-nginxdockerhighavailability-2.png)

- 예제는 `docker-compose` 파일을 분리해서 구성했지만, 1개의 `docker-compose` 파일에서 구성하는 것도 가능하다.
- `default.conf`

	```
	upstream app-server {
	    least_conn;
	    server webapp-real:8080 max_fails=1 fail_timeout=2s;
	    server webapp-backup:8080 backup;
	}
	
	server {
	    server_name 127.0.0.1;
	    error_log  /var/log/nginx/error.log;
	    access_log /var/log/nginx/access.log;
	
	    location / {
	        proxy_pass http://app-server;
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        proxy_set_header Host $http_host;
	    }
	}
	```  
	
	- `upstream` 블럭은 Nginx load balancing 이고 2개의 서비스를 클러스티링 해서 하나의 서비스로 제공한다.
	- `webapp-real` 은 메인 서비스이고, 배포되거나 서비스가 내려가지 않는 한, 클라이언트의 요청을 받는 역할이다.
	- `webapp-backup` 은 `webapp-real` 서비스가 내려가거나 배포되는 상황 처럼 요청을 받을 수 없을 때만 요청을 받아 처리한다.
	
- `docker-compose-nginx.yml`

	```yaml
	version: '3.7'
	
	services:
	  nginx:
	    image: nginx:stable-alpine
	    ports:
	      - "80:80"
	    volumes:
	      - ./default.conf:/etc/nginx/conf.d/default.conf
	    networks:
	      - nginxdocker-net
	    restart: on-failure
	
	networks:
	  nginxdocker-net:
	    driver: overlay
	    attachable: true
	```  
	
- `docker-compose-app-real.yml`

	```yaml
	version: '3.7'

	services:
	  webapp-real:
	    hostname: "app-real-{{.Task.Slot}}"
	    image: windowforsun/test-web
	    networks:
	      - nginxdocker-net
	    deploy:
	      replicas: 3
	      update_config:
	        parallelism: 1
	        delay: 10s
	
	
	networks:
	  nginxdocker-net:	    
	    driver: overlay
	    attachable: true
	```  
	
	- `hostname` 을 이름 + 클러스터링된 번호로(`app-real-{{.Task.Slot}}`) 설정한다.
	- 서비스 업데이트를 할때 `update_config` 설정으로 컨테이너 1개씩 10초 딜레이를 주고 수행하도록 설정을 했다.
	
- `docker-compose-app-backup.yml`

	```yaml
	version: '3.7'

	services:
	  webapp-backup:
	    hostname: "app-backup-{{.Task.Slot}}"
	    image: windowforsun/test-web
	    networks:
	      - nginxdocker-net
	    deploy:
	      replicas: 2
	
	networks:
	  nginxdocker-net:
	    driver: overlay
	    attachable: true
	```  
	
	- `hostname` 을 이름 + 클러스터링된 번호로(`app-backup-{{.Task.Slot}}`) 설정한다.
	
- 3개의 `docker-compose` 파일에서 `network` 는 모두 동일하게 `nginxdocker-net` 에 `external: true` 를 사용한다면, 따로 `nginxdocker-net` 네트워크를 생성해 주어야 한다.

	```
	$ docker network create --driver=overlay --attachable nginxdocker-net
	gm6pa0yiko1pp8swo9ebmymaz
	```  
	
	- 네트워크를 별도로 `overlay` 로 생성하는 이유는 `docker swarm` 을 사용하기 때문이다.
	- `overlay` 네트워크를 사용하면 호스트간 네트워크를 형성할 수 있다.
	- `docker swarm` 에서 컨테이너간 통신을 위해서 `--attachable` 옵션도 추가해야 한다.
- `docker swarm` 사용을 위해 `docker swarm init` 명령어로 활성화 해준다.

	```
	$ docker swarm init
	Swarm initialized: current node (aqrh1kkfk4svrkbnqyzu2emmq) is now a manager.
	
	To add a worker to this swarm, run the following command:
	
	    docker swarm join --token SWMTKN-1-4qyn8kti7f35n0144ntbcp3rdaab56f63i0a9dkcvda8ylttta-d1tynfqmg3qwl120yat09zdf4 10.0.2.15:2377
	
	To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
	```  

- 실행시키는 명령어는 아래와 같다.

	```
	# 한번에 한 스택으로 실행
	docker stack deploy -c docker-compose-app-real.yml -c docker-compose-app-backup.yml -c docker-compose-nginx.yml nginxdocker

	# 파일단위로 다른 스택으로 실행
	docker stack deploy -c docker-compose-nginx.yml nginxdocker-nginx
	docker stack deploy -c docker-compose-app-real.yml nginxdocker-real
	docker stack deploy -c docker-compose-app-backup.yml nginxdocker-backup
	```  
	
- Nginx 의 설정파일인 `default.conf` 가 구동중 변경될 경우 아래 명령어를 통해 갱신 시킬 수 있다.
	
	```
	docker exec -it <nginx-container-id-or-name> nginx -s reload
	```  

- 구성된 Docker 환경을 실행 시키면 아래와 같이 출력된다.

	```
	$ docker stack deploy -c docker-compose-app-real.yml -c docker-compose-app-backup.yml -c docker-compose-nginx.yml nginxdocker
	Ignoring unsupported options: restart
	
	Creating service nginxdocker_nginx
	Creating service nginxdocker_webapp-backup
	Creating service nginxdocker_webapp-real
	```  

- 웹 브라우저에서 `http://localhost:80` 으로 요청을 하면 `app-real-*` 호스트에만 접속이 되는것을 확인 할 수 있다.
	- `curl http://127.0.0.1` 명령어를 사용해서 CLI 로도 확인 가능하다.

	```
	$ curl http://127.0.0.1
	app-real-1, current time : 1580289821697
	$ curl http://127.0.0.1
	app-real-2, current time : 1580289823212
	$ curl http://127.0.0.1
	app-real-1, current time : 1580289829814
	```  
	<!--![그림 1]({{site.baseurl}}/img/docker/practice-nginxdockerhighavailability-3.png)-->
	<!--![그림 1]({{site.baseurl}}/img/docker/practice-nginxdockerhighavailability-4.png)-->

- `webapp-real` 서비스를 강제로 중지 시키면 `app-backup-*` 호스트만 출력된다.

	```
	$ docker service ls
	ID                  NAME                        MODE                REPLICAS            IMAGE
	  PORTS
	w1j83u8dreq2        nginxdocker_nginx           replicated          1/1                 nginx:stable-alpine
	  *:80->80/tcp
	coo8ey41bhfd        nginxdocker_webapp-backup   replicated          2/2                 windowforsun/test-web:latest
	
	kng8f5aakoyz        nginxdocker_webapp-real     replicated          2/2                 windowforsun/test-web:latest
	
	$ docker service rm nginxdocker_webapp-real
	nginxdocker_webapp-real
	```  

	```
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580290926999
	$ curl http://127.0.0.1
	app-backup-2, current time : 1580290929123
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580290931923
	```  
	<!--![그림 1]({{site.baseurl}}/img/docker/practice-nginxdockerhighavailability-5.png)-->
	<!--![그림 1]({{site.baseurl}}/img/docker/practice-nginxdockerhighavailability-6.png)-->
	
- 다시 `webapp-real` 서비스를 올리고 Nginx 을 reload 시키면 다시 `app-real-*` 호스트만 출력되는 것을 확인 할 수 있다.

	```
	$ docker stack deploy -c docker-compose-app-real.yml nginxdocker
	Creating service nginxdocker_webapp-real
	$ docker ps | grep nginxdocker_nginx
	c12c7a1c8315        nginx:stable-alpine            "nginx -g 'daemon of…"   24 minutes ago      Up 24 minutes       80/tcp                                           nginxdocker_nginx.1.jismk1g12izysihj7uf98fyk4
	$ docker exec -it c12c nginx -s reload
	2020/01/29 11:46:28 [notice] 8#8: signal process started
	```  
	
- 실제 배포 되는 환경과 유사한 테스트를 위해서는 아래와 같은 방법이 있다.
    - Web Application 으로 사용하고 있는 `windowforsun/test-web` 이미지를 새로 빌드 후 `docker deploy -c docker-compose-app-real.yml nginxdocker` 명렁어로 `webapp-real` 서비스를 다시 배포한다.
    - `webapp-real` 서비스의 스케일을 확장하는 명령어(`docker service scale nginxdocker_webapp-real=3`) 를 통해 서비스가 내려갔다 다시 올라오도록 한다.
    
- 간편한 테스트를 위해 후자 방식을 택해서 테스트를 하면 명령어가 수행되고 난후 `webapp-real` 서비스가 내려간 시점에서는 `app-backup-*` 호스트가 출력되다가, 다시 올라온 시점부터 `app-real-*` 호스트가 출력되는 것을 확인 할 수 있다.

	```
	$ docker service scale nginxdocker_webapp-real=3
	nginxdocker_webapp-real scaled to 3
	overall progress: 3 out of 3 tasks
	1/3: running
	2/3: running
	3/3: running
	verify: Service converged
	```  	

	```
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580294327124
	$ curl http://127.0.0.1
	app-backup-2, current time : 1580294328932
	$ curl http://127.0.0.1
	app-real-1, current time : 1580294329910
	$ curl http://127.0.0.1
	app-real-2, current time : 1580294330563
	```  
	<!--![그림 1]({{site.baseurl}}/img/docker/practice-nginxdockerhighavailability-7.png)-->	
	<!--![그림 1]({{site.baseurl}}/img/docker/practice-nginxdockerhighavailability-8.png)-->
	
- `webapp-real` 서비스를 `windowforsun/test-web:v1` 이미지로 업데이트를 할때 상황에 대한 테스트는 아래와 같다.
	- 이미지 업데이트를 위해 `docker service update --image windowforsun/test-web:v1` 명령어를 수행해준다.
	
	```
	$ docker service update --image windowforsun/test-web:v1 nginxdocker_webapp-real
	nginxdocker_webapp-real
	overall progress: 2 out of 2 tasks
	1/2: running
	2/2: running
	```  
	
	- 업데이트가 수행되는 동안 요청을 보내면 아래와 같은 결과가 나온다.
		1. 먼저 업데이트를 수행하는 `app-real-1` 은 요청을 받지않고 `app-real-2` 만 요청을 받아 수행한다.
		1. `app-real-2` 까지 업데이트를 수행하면, `app-backup-*` 에서 요청을 받아 수행한다.
		1. `app-real-*` 업데이트가 모두 완료되면, `app-real-*` 에서 요청을 받아 수행한다.
	
	```
	$ curl http://127.0.0.1
	app-real-2, current time : 1580359841841
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580359842376
	$ curl http://127.0.0.1
	app-backup-2, current time : 1580359842907
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580359843480
	$ curl http://127.0.0.1
	app-backup-2, current time : 1580359843962
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580359844481
	$ curl http://127.0.0.1
	app-real-2, current time : 1580359845009
	$ curl http://127.0.0.1
	app-backup-2, current time : 1580359845558
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580359846110
	$ curl http://127.0.0.1
	app-backup-2, current time : 1580359846678
	$ curl http://127.0.0.1
	app-backup-1, current time : 1580359847212
	$ curl http://127.0.0.1
	app-backup-2, current time : 1580359847741
	$ curl http://127.0.0.1
	app-real-2, current time : 1580359848294
	$ curl http://127.0.0.1
	app-real-1, current time : 1580359848988
	```  
	

---
## Reference

	