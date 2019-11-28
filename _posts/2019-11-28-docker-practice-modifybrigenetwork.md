--- 
layout: single
classes: wide
title: "[Docker 실습] Bridge Network IP 및 대역 변경하기"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 의 Network IP 대역을 변경해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Docker
    - Practice
    - Bridge Network
---  

# 환경
- Docker

## Network 상태 확인
- `ifconfig docker` 명령어를 통해 현재 Docker Bridge Network 에대한 정보를 확인한다.

	```
	[root@localhost vagrant]# ifconfig docker
	docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
	        inet 172.19.0.1  netmask 255.255.0.0  broadcast 172.19.255.255
	        ether 02:42:6c:4f:9e:c8  txqueuelen 0  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	```  
	
- 현재 상태에서 Docker Network 를 추가하면 아래와 같이 생성된다.

	```
	[root@localhost vagrant]# docker network create test
	62cd472f090520863c83a32918197b9ddb65adf925152106b9199f77c5388fdc
	[root@localhost vagrant]# ifconfig
	br-62cd472f0905: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
	        inet 172.18.0.1  netmask 255.255.0.0  broadcast 172.18.255.255
	        ether 02:42:69:91:93:8b  txqueuelen 0  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	[root@localhost vagrant]# docker network inspect test
	[
	    {
	        "Name": "test",
	        "Id": "62cd472f090520863c83a32918197b9ddb65adf925152106b9199f77c5388fdc",
	        "Created": "2019-10-16T03:51:30.906240048Z",
	        "Scope": "local",
	        "Driver": "bridge",
	        "EnableIPv6": false,
	        "IPAM": {
	            "Driver": "default",
	            "Options": {},
	            "Config": [
	                {
	                    "Subnet": "172.18.0.0/16",
	                    "Gateway": "172.18.0.1"
	                }
	            ]
	        },
	        "Internal": false,
	        "Attachable": false,
	        "Ingress": false,
	        "ConfigFrom": {
	            "Network": ""
	        },
	        "ConfigOnly": false,
	        "Containers": {},
	        "Options": {},
	        "Labels": {}
	    }
	]
	```  
	
## Docker Bridge IP 변경하기
- Docker 서비스 재시작 시에도 변경된 IP 를 적용하기 위해서는 `/etc/docker/daemon.json` 파일에 설정을 해야 한다.
- IP 설정전 `docker network prune` 명렁어로 Docker Network 를 정리해 준다.
	- 현재 사용하지 않는 Docker Network 설정이 모두 삭제 되기때문에 추후에 삭제된 네트워크를 사용하는 컨테이너가 있을 경우 컨테이너 삭제후 다시 빌드해야 한다.
- `iptables -t nat -F POSTROUTING` 명렁어로 iptable 에서 POSTROUTING 설정을 삭제한다.
- Docker Bridge 의 IP 를 `15.15.0.1` 로 변경하는 `daemon.json` 설정은 아래와 같다.

	```json
	{
        "bip":"15.15.0.1/24"
	}
	```  
		
- `systemctl restart docker` 명령어를 통해 서비스를 재시작한다.
- `ifconfig docker` 로 확인하면 Docker Bridge 의 IP 가 변경된 것을 확인 할 수 있다.

	```
	[root@localhost vagrant]# ifconfig docker
	docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
	        inet 15.15.0.1  netmask 255.255.255.0  broadcast 15.15.0.255
	        ether 02:42:6c:4f:9e:c8  txqueuelen 0  (Ethernet)
	        RX packets 0  bytes 0 (0.0 B)
	        RX errors 0  dropped 0  overruns 0  frame 0
	        TX packets 0  bytes 0 (0.0 B)
	        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
	```

- docker network create test 로 새로운 Docker Network 를 생성할때, 아이피는 변경되지 않는다.

	```
	[root@localhost vagrant]# docker network create test
	631bcf1d4becaa8db7ea1a6fcdca1a4426d182f40171514c4942eeb85a662d6f
	[root@localhost vagrant]# docker network inspect test
	[
	    {
	        "Name": "test",
	        "Id": "631bcf1d4becaa8db7ea1a6fcdca1a4426d182f40171514c4942eeb85a662d6f",
	        "Created": "2019-10-16T05:00:21.655407387Z",
	        "Scope": "local",
	        "Driver": "bridge",
	        "EnableIPv6": false,
	        "IPAM": {
	            "Driver": "default",
	            "Options": {},
	            "Config": [
	                {
	                    "Subnet": "172.17.0.0/16",
	                    "Gateway": "172.17.0.1"
	                }
	            ]
	        },
	        "Internal": false,
	        "Attachable": false,
	        "Ingress": false,
	        "ConfigFrom": {
	            "Network": ""
	        },
	        "ConfigOnly": false,
	        "Containers": {},
	        "Options": {},
	        "Labels": {}
	    }
	]
	```  
	
## Docker Network IP 대역 변경하기
- Docker Network 에서 생성하는 아이피 대역을 변경하기 위해서도 `/etc/docker/daemon.json` 에 설정을 추가해야 한다.
- `default-address-pools` 설정으로 15.16.x.x 대역의 아이피가 생성될 수 있도록 추가한다.

	```
	{
	        "bip":"15.15.0.1/24",
	        "default-address-pools":[
	                {"base":"15.16.0.0/16", "size":24}
	        ]
	}
	```  

- `systemctl restart docker` 로 서비스를 재시작한다.
- `docker network create test2` 로 새로운 네트워크를 생성하면 아래와 같이 `15.16.x.x` 대역의 아이피가 생성된것을 확인 할 수 있다.

	```
	[root@localhost vagrant]# docker network create test2
	890bcb4f0cf68b3b263f68c8122a0ef0d1fd22ba2b0f7ea5fce1391c505259c7
	[root@localhost vagrant]# docker network inspect test2
	[
	    {
	        "Name": "test2",
	        "Id": "890bcb4f0cf68b3b263f68c8122a0ef0d1fd22ba2b0f7ea5fce1391c505259c7",
	        "Created": "2019-10-16T05:12:43.432941021Z",
	        "Scope": "local",
	        "Driver": "bridge",
	        "EnableIPv6": false,
	        "IPAM": {
	            "Driver": "default",
	            "Options": {},
	            "Config": [
	                {
	                    "Subnet": "15.16.1.0/24",
	                    "Gateway": "15.16.1.1"
	                }
	            ]
	        },
	        "Internal": false,
	        "Attachable": false,
	        "Ingress": false,
	        "ConfigFrom": {
	            "Network": ""
	        },
	        "ConfigOnly": false,
	        "Containers": {},
	        "Options": {},
	        "Labels": {}
	    }
	]
	```  
	
---
## Reference
