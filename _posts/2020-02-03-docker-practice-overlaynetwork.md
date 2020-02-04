--- 
layout: single
classes: wide
title: "[Docker 실습] Overlay Network"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: '서비스와 독립 컨테이너간의 통신이 가능한 Overlay Network 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Overlay Network
---  


## Docker Swarm 의 기본 네트워크
- `Swarm` 을 통해 서비스를 구성하고 배포하면 `ingress`, `docker_gwbridge` 라는 네트워크가 자동으로 생성된다.

### ingress
- overlay 네트워크 드라이버이다.
- `Swarm` 에 구성된 서비스들의 네트워크 트래픽을 통제한다.
- `Swarm` 을 구성하는 기본 네트워크 드라이버이고, 서비스 생성시 별도로 네트워크 정보를 지정하지 않으면 자동으로 `ingress` 네트워크에 연결된다.

### docker_gwbridge
- `bridge` 네트워크 드라이버로 `Swarm` 에 참여한 `Docker Demon` 을 연결해주는 역할을 한다.
	- `Swarm` 에 참여한 호스트(node) 를 연결하는 역할
	

## Overlay Network 란
- Overlay Network 는 도커 데몬 호스트들 간의 분산 네트워크를 구성해 준다.
- 호스트 네트워크의 앞단에 있어, 각 호스트에 있는 컨테이너 간 통신과 안전한 통신이 가능하도록 한다.
- 일반적인 `bridge` 네트워크 처럼 `overlay` 또한 사용자 정의를 통해 네트워크를 생성할 수 있다.
- `overlay` 를 사용하면 서비스, 스택에 구성된 컨테이너 외에 독립형으로 실행된 컨테이너와도 통신이 가능하다.

## Overlay Network 테스트
- 서비스와 독립형 컨테이너를 각각 띄우고, `overlay` 를 통해 네트워크가 가능한지 테스트 해본다.

### 단순 Overlay Network
- 단순 Overlay Network 란 아무런 옵션값을 주지 않고 `driver` 만 `overlay` 로 설정한 네트워크를 뜻한다.
- `docker network create -d overlay test-overlay-net` 명령어를 통해 단순 Overlay Network 를 생성해 준다.

	```
	$ docker network create -d overlay test-overlay-net
	913san8ckpk4trme2g68uscib
	```  
	
- 생성한 단순 Overlay 네트워크를 사용하는 Nginx 서비스를 띄워 준다.

	```
	$ docker service create --name overlay-nginx -p 80:80 --replicas=3 --network test-overlay-net nginx
	jgkjuoorv8oynq5elkzqz0muv
	overall progress: 3 out of 3 tasks
	1/3: running   [==================================================>]
	2/3: running   [==================================================>]
	3/3: running   [==================================================>]
	verify: Service converged
	```  
	
- 독립형 컨테이너에서 사용할 `bridge` 네트워크를 하나 생성하고, 독립형 컨테이너(alpine)를 하나 띄운다.

	```
	$ docker network create -d bridge test-bridge-net
	9ad1c1bc2b77c2c65fcf29c4c7f0d5b8771eb17f7154c20cdb266db56223fe02
	$ docker run -dit --name test-alpine --network test-bridge-net alpine
	d247fcb5177909f20008c3ecb645402210cd4a2fb9bc01a0777a613fa6df378c
	```  
	
- 독립형 컨테이너에서 `overlay-nginx` 와 연결이 가능한지 `ping` 명령어로 테스트 해본다.
	- `bridge` 네트워크의 경우 `Auto DNS Resolution` 기능으로 같은 네트워크 안에서 서비스의 이름으로 요청 및 연결을 수행할 수 있다.
	
	```
	$ docker exec test-alpine /bin/sh -c 'ping overlay-nginx'
	ping: bad address 'overlay-nginx'
	
	# 직접 접속해서 명령어를 사용하는 것도 가능하다.
	$ docker attach test-alpine
	$ ping overlay-nginx
	ping: bad address 'overlay-nginx'
	```  
	
	- 호스트를 찾지 못해 `ping` 명령어가 수행이 되지 않는 것을 확인 할 수 있다.
- 위 테스트는 독립 컨테이너가 `overlay-nginx` 에서 사용하는 `test-oerverlay-net` 네트워크에 참여 하지 않았기 때문에 발생하는 상황이기 때문에, 독립 컨테이너를 `test-overlay-net` 에 참여 시켜본다.

	```
	$ docker network connect test-overlay-net test-alpine
	Error response from daemon: Could not attach to network test-overlay-net: rpc error: code = PermissionDenied desc = network test-overlay-net not manually attachable
	```  
	
	- 네트워크에 참여 시킬수 없다는 에러 메시지가 출력된다.
	
### Attach Overlay Network
- 서비스에서 사용하는 네트워크에서 독립 컨테이너를 연결하고 싶다면 `overlay` 에 `--attachable` 옵션을 부여해야 한다.
- `--attachable` 옵션이 부여된 `overlay` 네트워크를 생성한다.

	```
	$ docker network create -d overlay --attachable test-overlay-attach-net
	ile5smrlua1f48brikjogq0vp
	```  
	
- `overlay-nginx` 서비스에서 사용하는 네트워크를 `test-overlay-net` 에서 새로 생성한 `test-overlay-attach-net` 으로 변경해야 한다.
- 아래 명령어로 변경이 가능하지만 이미 서비스 중이라면, 서비스에 차질이 생길수 있다.

	```
	$ docker service create --name overlay-nginx -p 80:80 --replicas=3 --network test-overlay-attach-net nginx
	```  
	
- 이미 서비스가 올라간 상태에서 네트워크를 추가하기 위해 `docker service update` 명령어로 네트워크를 새로 생성한 네트워크를 추가해 준다.

	```
	$ docker service update --network-add test-overlay-attach-net overlay-nginx
	overlay-nginx
	overall progress: 3 out of 3 tasks
	1/3: running   [==================================================>]
	2/3: running   [==================================================>]
	3/3: running   [==================================================>]
	verify: Service converged
	```  
	
- `overlay-nginx` 에서 기존에 사용하던 네트워크 또한 `docker service update` 명령어로 삭제해 준다.

	```
	$ docker service update --network-rm test-overlay-net overlay-nginx
	overlay-nginx
	overall progress: 3 out of 3 tasks
	1/3: running   [==================================================>]
	2/3: running   [==================================================>]
	3/3: running   [==================================================>]
	verify: Service converged
	```  
	
- `test-overlay-attach-net` 네트워크는 `--attachable` 옵션이 부여된 `overlay` 네트워크이기 때문에 독립 컨테이너(alpine) 을 참여시킬 수 있다.

	```
	$ docker network connect test-overlay-attach-net test-alpine
	```  
	
- 다시 독립 컨테이너에서 `ping` 명령어를 통해 `overlay-nginx` 와 연결 테스트를 하면 연결이 가능 한 것을 확인 할 수 있다.

	```
	$ docker exec test-alpine /bin/sh -c 'ping -c 5 overlay-nginx'
	PING overlay-nginx (10.0.4.2): 56 data bytes
	64 bytes from 10.0.4.2: seq=0 ttl=64 time=0.082 ms
	64 bytes from 10.0.4.2: seq=1 ttl=64 time=0.132 ms
	64 bytes from 10.0.4.2: seq=2 ttl=64 time=0.064 ms
	64 bytes from 10.0.4.2: seq=3 ttl=64 time=0.098 ms
	64 bytes from 10.0.4.2: seq=4 ttl=64 time=0.102 ms
	
	--- overlay-nginx ping statistics ---
	5 packets transmitted, 5 packets received, 0% packet loss
	```  
	
### 암호화 Overlay Network
- 호스트간의 통신이 내부망이 아니라, 외부망을 통한다면 데이터 노출에 대해 대응이 필요하다.
- `overlay` 네트워크에서 `-opt encrypted` 옵션을 사용하면 암호화를 통한 통신을 할수 있다.
	- 모든 호스트(노드)간의 IPSEC 터널링
	- GCM 모드의 AES 알고리즘 사용
	- 12시간 마다 키 로테이션
	
```
$ docker network create --opt encrypted --driver overlay --attachable test-overlay-attach-encrypted-net
```  

---
## Reference

	