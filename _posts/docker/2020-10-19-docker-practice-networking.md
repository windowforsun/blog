--- 
layout: single
classes: wide
title: "[Docker 실습] Docker Network"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker Container 네트워크와 사용할 수 있는 Network Driver 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Network
  - bridge
  - overlay
  - macvlan
  - none
toc: true
use_math: true
---  

## Docker Network
`Docker` 는 컨테이너 기반으로 애플리케이션을 가볍고 간편하게 
패키징하고 구성할 수 있는 도구이다. 
서비스를 구성하는 다양한 애플리케이션이 동작하기 위해서는 어떠한 방법으로든 `Input` 이 필요하다. 
요즘 대부분 애플리케이션의 경우 로컬에서 `Input` 이 들어오고 
그 `Output` 을 로컬에서만 사용하는 경우는 드물것이다. 
그 말은 대부분의 애플리케이션은 `Network` 라는 자원을 사용해서, 
각 역할을 가지고 분리된 애플리케이션들이 유기적으로 연결되는 구성을 가지게 된다.  

`Docker` 를 사용해서 애플리케이션을 패키징하고 이를 통해 서비스를 구성에 필요한 
`Docker` 의 `Networking` 구성과 사용할 수 있는 드라이버에 대해 알아본다.  

먼저 하나의 `Host` 에 `Docker` 를 설치하면 
아래와 같은 `docker0` 라는 네트워크 인터페이스를 확인할 수 있다. 

```bash
$ ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
      inet6 fe80::42:73ff:fe36:7085  prefixlen 64  scopeid 0x20<link>
      ether 02:42:73:36:70:85  txqueuelen 0  (Ethernet)
      RX packets 1136  bytes 48748 (47.6 KiB)
      RX errors 0  dropped 0  overruns 0  frame 0
      TX packets 1383  bytes 17354723 (16.5 MiB)
      TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
      inet6 fe80::5054:ff:fe4d:77d3  prefixlen 64  scopeid 0x20<link>
      ether 52:54:00:4d:77:d3  txqueuelen 1000  (Ethernet)
      RX packets 142594  bytes 188647767 (179.9 MiB)
      RX errors 0  dropped 0  overruns 0  frame 0
      TX packets 15293  bytes 1125517 (1.0 MiB)
      TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```  

`docker0` 의 상세 정보를 바탕으로 네트워크 대역이 `172.17.0.0/16` 인 것을 확인 할 수 있다. 
`docker0` 는 도커를 실행하게 되면 자동으로 생성되는 가상 네트워크 인터페이스이다. 
그리고 이후에 설명하겠지만 `Docker` 가 제공하는 `Network Driver` 중에서는 `Bridge` 에 해당한다. 
또한 `docker0` 의 역할은 `linux bridge` 로 하나의 `Docker Daemon` 에서 실행 중인 
컨테이너가 통신하기 위한 `L2` 를 담당한다. 
하나의 호스트(`Docker Daemon`)에서 컨테이너 간 통신이나 외부와의 통신시에는 `docker0` 통해 이뤄진다.  

`brctl` 명령으로 호스트에 구성된 `Bridge` 를 조회하면 아래와 같다. 

```bash
$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024273367085       no
```  

이상태에서 아래 명령으로 `Nginx` 컨테이너를 하나 실행한다. 

```bash
$ docker run --rm -d --name test-nginx nginx:latest
90610ce2300092be22a7b898cb226f469828a794ba36b0387983669900c05cff
```  

컨테이너가 생성되면 각 컨테이너는 격리된 네트워크 공간을 할당받게 된다. 
이는 [linux namespace](https://en.wikipedia.org/wiki/Linux_namespaces) 
라는 기술을 통해 구현된 가상화된 기법으로 
컨테이너가 각자의 네임스페이스로 독립된 네트워크를 할당받을 수 있게 해준다.  

컨테이너가 하나 실행 됐을 때 호스트이 네트워크가 어떻게 달라졌는지 살펴본다. 
먼저 `ifconfig` 로 네트워크 인터페이스를 살펴본다. 

```bash
$ ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:73ff:fe36:7085  prefixlen 64  scopeid 0x20<link>
        ether 02:42:73:36:70:85  txqueuelen 0  (Ethernet)
        RX packets 1136  bytes 48748 (47.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1383  bytes 17354723 (16.5 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::5054:ff:fe4d:77d3  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:4d:77:d3  txqueuelen 1000  (Ethernet)
        RX packets 143019  bytes 188711969 (179.9 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 15538  bytes 1154273 (1.1 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vethac20fa0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::b4e0:87ff:feb0:5110  prefixlen 64  scopeid 0x20<link>
        ether b6:e0:87:b0:51:10  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 656 (656.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```  

`vethac20fa0` 라는 네트워크 인터페이스가 새롭게 생성된 것을 확인할 수 있다. 
독립된 네트워크 공간이란 `veth` 를 통해 구현된다. 
컨테이너의 `eth0` 인터페이스와 호스트에 네임스페이스로 구분된 네트워크 인터페이스가 
한 쌍으로 바인딩되어 컨테이너가 외부로 네트워크가 가능하도록 한다. 
그리고 `Brdige` 에 대한 정보를 조회하면 아래와 같다. 

```bash
$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024273367085       no              vethac20fa0
```  

새롭게 생성된 `vethac20fa0` 네트워크 인터페이스는 `docker0` 에 연결된 인터페이스인 것을 확인할 수 있다. 
`ip link` 명령으로 네트워크 인터페이스의 상태를 확인하면 아래와 같다. 

```bash
$ ip link
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:73:36:70:85 brd ff:ff:ff:ff:ff:ff
9: vethac20fa0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether b6:e0:87:b0:51:10 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```  

`vethac20fa0` 부분을 살펴보면 `master docker0` 를 통해 다시한번 `docker0` 브릿지에 연결된 것을 확인할 수 있다. 

이번에는 실제 컨테이너 내부에서 네트워크 인터페이스를 조회해 본다. 

```bash
$ docker exec -it test-nginx /bin/bash
root@90610ce23000:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 794  bytes 8683026 (8.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 652  bytes 36737 (35.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```  

>`ifconfig` 명령을 사용할 수 없다면 `apt install net-tools` 로 설치해 준다. 

실행된 컨테이너 내부에는 `eth0` 네트워크 인터페이스가 있고, 
할당된 아이피는 `172.17.0.2` 인 것을 확인할 수 있다.  

그리고 `route` 명령으로 네트워크의 라우팅 정보를 조회하면 아래와 같다. 

```bash
route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```  

`Gateway` 필드를 확인해보면, 
`172.17.0.1` 로 호스트의 `docker0` 네트워크 인터페이스를 사용하는 것을 확인할 수 있다. 

지금까지 확인한 내용을 도식화하면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/docker/practice_docker_networking_1.png)

지금까지 설명한 내용과 위 그림은 이후에 설명할 `Network Driver` 중 `bridge` 의 내용에 해당한다.  

### Network Namespace
앞서 컨테이너가 생성되면 `veth` 라는 가상 네트워크 인터페이스가 생성되고, 
컨테이너 내부의 `eth0` 인터페이스와 바인딩 된다고 설명했었다. 
이부분에 대해 좀더 자세히 살펴본다.  

컨테이너가 생성되면 `/var/run/docker/netns` 디렉토리에 독립된 네임스페이스가 생성된다. 
이는 아래와 같이 `docker inspect` 명령으로 확인할 수 있다. 

```bash
$ docker inspect -f {{.NetworkSettings.SandboxKey}} test-nginx
/var/run/docker/netns/8b7504f82c82
```  

위 경로를 `/var/run/netns` 에 심볼릭 링크를 걸어 주게 되면 호스트에서 
컨테이너의 네트워크 정보를 조회할 수 있다. 

```bash
$ ln -s /var/run/docker/netns /var/run/netns
$ ip netns exec 8b7504f82c82 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
$ docker exec test-nginx ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```  

### 도커 컨테이너에서 호스트로 요청
도커 컨테이너에서 호스트에 실행중인 서버로 요청을 보내야 하는 상황이 있을 수도 있다. 
이런 경우에는 컨테이너에서 요청 도메인을 `host.docker.internal` 로 사용하면 가능하다.  

## Network Driver
`Docker` 는 컨테이너 네트워크 동작을 위해 몇가지 네트워크 드라이버를 제공한다. 
이를 통해 사용자는 구성과 목적에 맞는 드라이버를 선택해서 컨테이너의 네트워크를 구성할 수 있다. 
네트워크 드라이버로는 아래와 같은 것들이 있다. 
- `bridge`
- `host`
- `overlay`
- `macvlan`
- `none`
- `Network Plugin` 

`Docker` 가 구동되면 기본으로 기사용할 수 있는 네트워크를 생성하게 된다. 
이는 `docker network ls` 명령으로 확인할 수 있다. 

```bash
docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
f4b977194d61        bridge              bridge              local
600a9dc163a0        host                host                local
b3442786b8e2        none                null                local
```  

### bridge
`Docker` 의 기본 네트워크 드라이버로 네트워크를 생성할때, 
별도로 지정하지 않으면 기본으로 설정되는 드라이버이다. 
그리고 컨테이너에 별도의 네트워크를 지정하지 않으면 지정되는 네트워크 이기도 하다. 
각 컨테이너마다 고유한 `Network namespace` 영역이 생성된다. 
`bridge` 타입의 네트워크는 연결된 컨테이너간 혹은 외부와 통신을 담당한다.   

`Docker` 에 기본으로 생성되는 `bridge` 네트워크의 경우, 
앞서 설명한 `docker0` 네트워크 인터페이스를 바라보고 있다. 
`docker network inspect bridge` 명령으로 기본 `bridge` 네트워크의 상세 정보를 확인하면, 
아래와 같이 `docker0` 네트워크의 인터페이스 대역인 것과 
`Options` 의 `com.docker.network.bridge.name` 의 값이 `docker0` 인것을 확인 할 수 있다. 

```bash
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "f4b977194d614d77a391b7bb106bcad2d5306181aa3a34739360d567822ab1a9",
        "Created": "2020-10-16T08:44:21.530707043Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16"
                }
            ]
        },

        .. 생략 ..

        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
    }
]
```  

즉 별도의 네트워크를 지정하지 않고, 
컨테이너를 생성하게 되면 `brdige` 도커 네트워크를 통해 `docker0` 네트워크 인터페이스와 연결된다.  

필요하다면 별도의 `brdige` 네트워크를 생성할 수 있다. 

```bash
$ docker network create test-bridge
6658776917d2bd29c18918ca18f6324c0cb8c1eb5890736c0d4e8e9a205cf5d6
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
f4b977194d61        bridge              bridge              local
600a9dc163a0        host                host                local
b3442786b8e2        none                null                local
6658776917d2        test-bridge         bridge              local
```  

`ifconfig` 로 호스트에 존재하는 네트워크 인터페이스를 조회하면, 
아래와 같이 `br-6658776917d2` 이라는 새로운 네트워크 인터페이스가 생성된 것을 확인 할 수 있다. 

```bash
$ ifconfig
br-6658776917d2: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.19.0.1  netmask 255.255.0.0  broadcast 172.19.255.255
        ether 02:42:69:d6:49:91  txqueuelen 0  (Ethernet)
        RX packets 652  bytes 36737 (35.8 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 794  bytes 8683026 (8.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:73ff:fe36:7085  prefixlen 64  scopeid 0x20<link>
        ether 02:42:73:36:70:85  txqueuelen 0  (Ethernet)
        RX packets 1788  bytes 76357 (74.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2169  bytes 26037093 (24.8 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::5054:ff:fe4d:77d3  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:4d:77:d3  txqueuelen 1000  (Ethernet)
        RX packets 151113  bytes 197819671 (188.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17075  bytes 1298642 (1.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```  

명령 결과로 알수있듯이 `test-bridge` 의 아이피대역은 `172.19.0.0/16` 이다. 
호스트에서 존재하는 `Bridge` 를 확인하면 생성한 `test-bridge` 의 인터페이스를 확인할 수 있다. 

```bash
brctl show
bridge name     bridge id               STP enabled     interfaces
br-6658776917d2         8000.024269d64991       no
docker0         8000.024273367085       no              vethac20fa0
```  

`test-bridge` 를 사용하는 컨테이너를 생성하고 관련 정보를 확인하면 아래와 같다. 

```bash
$ docker run --rm -d --name test-nginx-2 --network test-bridge nginx:latest
1b858951855d807011d6899dde7212447b94deccac8c052d92ce656d2fa043cc
$ ifconfig
br-6658776917d2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.19.0.1  netmask 255.255.0.0  broadcast 172.19.255.255
        inet6 fe80::42:69ff:fed6:4991  prefixlen 64  scopeid 0x20<link>
        ether 02:42:69:d6:49:91  txqueuelen 0  (Ethernet)
        RX packets 652  bytes 36737 (35.8 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 794  bytes 8683026 (8.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth5d9152c: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::a483:58ff:fe16:ed2b  prefixlen 64  scopeid 0x20<link>
        ether a6:83:58:16:ed:2b  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 13  bytes 1102 (1.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br-6658776917d2         8000.024269d64991       no              veth5d9152c
docker0         8000.024273367085       no              vethac20fa0
$ ip link
13: veth5d9152c@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-6658776917d2 state UP mode DEFAULT group default
    link/ether a6:83:58:16:ed:2b brd ff:ff:ff:ff:ff:ff link-netnsid 1
$ docker exec test-nginx-2 ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.19.0.2  netmask 255.255.0.0  broadcast 172.19.255.255
        ether 02:42:ac:13:00:02  txqueuelen 0  (Ethernet)
        RX packets 709  bytes 8678655 (8.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 557  bytes 31607 (30.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

$ docker exec test-nginx-2 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.19.0.1      0.0.0.0         UG    0      0        0 eth0
172.19.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```  

`test-nginx-2` 컨테이너와 바인딩된 `veth5d9152c` 는 `test-bridge` 네트워크 인터페이스와 연결된 것을 확인 할 수 있다. 
그리고 컨테이너의 `eth0` 인터페이스와 `Gateway` 를 확인하면 `veth5d9152c` 를 통해 `test-bridge` 와 연결된 것도 확인 가능하다. 

![그림 1]({{site.baseurl}}/img/docker/practice_docker_networking_2.png)

### host
`host` 드라이버는 컨테이너가 독립된 네트워크 영역을 가지지 않고, 
호스트와 네트워크를 함께 사용한다. 
그러기 때문에 `docker0` 와 바인딩 되지 않는다.  

컨테이너 실행시 네트워크를 `host` 로 지정하고 실행한다. 
그리고 네트워크 관련 정보를 확인하면 아래와 같다. 

```bash
$ docker run --rm -d --name test-nginx-host --network host nginx:latest
b86667fe253e60427f69f2b297c2f6b09e56cf63299287f6193c00c1f80d0dee
$ docker inspect -f {{.NetworkSettings.SandboxKey}} test-nginx-host
/var/run/docker/netns/default
$ docker exec test-nginx-host ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:9aff:fe8e:aebb  prefixlen 64  scopeid 0x20<link>
        ether 02:42:9a:8e:ae:bb  txqueuelen 0  (Ethernet)
        RX packets 706  bytes 30906 (30.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 834  bytes 9901843 (9.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::5054:ff:fe4d:77d3  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:4d:77:d3  txqueuelen 1000  (Ethernet)
        RX packets 34992  bytes 38084976 (36.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6818  bytes 1096305 (1.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
$ docker exec test-nginx-host route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.2.2        0.0.0.0         UG    100    0        0 eth0
10.0.2.0        0.0.0.0         255.255.255.0   U     100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```  

먼저 `Network Namespacd` 의 경우 `/var/run/docker/netns/deafult` 이다. 
그리고 컨테이너 내부에서 `ifconfig` 명령을 수행하면, 
호스트와 동일한 네트워크 인터페이스가 조회되는 것을 확인할 수 있다. 
`route -n` 을 컨테이너 내부에서 조회해도 호스트와 동일한 정보가 출력된다.  


![그림 1]({{site.baseurl}}/img/docker/practice_docker_networking_3.png)

### macvlan
`macvlan` 은 `Bridge` 를 사용하지 않고, 서브 인터페이스(`Linux Subinterface`) 라는 개념을 사용한다. 
호스트의 물리적인 네트워크 인터페이스인 `eth0` 에 하위 여러 개의 하위 인터페이스를 만듬으로써 
동시에 여러개 `MAC` 주소를 가질 수 있도록 구성하는 것을 의미한다. 
이로 인해 하위 인터페이스들에 여러 컨테이너들이 연결될 수 있게 된다. 
간단하게 하나의 네트워크 인터페이스 가상화해 여러 `MAC` 주소를 갖는 것을 의미한다.  

`macvlan` 은 부모 인터페이스와 서브 인터페이스로 나뉜다. 
부모 인터페이스는 가상화의 대상이 되는 `NIC`(`eth0`) 를 의미하고, 
`NIC` 를 가상화해서 생성되는 것을(`mac0`) 서브 인터페이스라고 한다. 
`eth0` 을 가상화해서 생성된 서브 인터페이스인 `mac0` 을 표현할때 `mac0@eth0` 이라고 한다.  

`macvlan` 의 방식중 `Docker` 에서는 `Bridge macvlan` 을 사용한다. 
`Bridge macvlan` 은 아래와 같은 특징이 있다. 
- 호스트 다른 네트워크와는 통신이 불가능하다. 
- 서브 인터페이스간에만 통신이 가능하다. 

`macvlan` 타입을 네트워크를 생성하게 되면 외부 통신은 물론이고, 다른 `NIC` 와도 통신이 불가능하다. 
서브 인터페이스간의 트래픽을 외부로 보내지 않고 바로 전달하는 방식으로 이뤄진다. 
이러한 특징으로 트래픽 전달시 수행해야하는 몇가지 동작이 제외되기 때문에 성능적으로는 이점이 생긴다.  

간단하게 `macvlan` 도커 네트워크를 생성하는 명령어는 아래와 같다. 

```bash
$ docker network create -d macvlan \
>  --subnet=123.11.11.0/24 \
>  --gateway=123.11.11.1 \
>  -o parent=eth0 \
>  eth0-macvlan
fb28abd9dec81f40bff468e1ef4f9c20dc9132e6b988a0aa5b3ed1446f0572bc
```  

`eth0` 을 부모 인터페이스로 사용하는 `eth0-macvlan` 이라는 도커 네트워크를 생성한다. 
`macvlan` 을 생성할때는 `subet` 과 `gateway`, `-o parent` 등의 정보를 명시해주어야 한다.  

`busybox` 이미지를 사용해서 `eth0-macvlan` 을 사용하는 컨테이너를 하나 생성하고, 
`ifconfig` 로 네트워크 인터페이스 정보를 확인하면 아래와 같다. 

```bash
$ docker run \
> --rm -dit \
> --name test-busybox-eth0macvlan-1 \
> --network eth0-macvlan \
> busybox \
> ash
995c10c120093afd02f8485ee36bd9b0fe2c8ae75c4949d4987cd8b7d81c3fa1
$ docker exec test-busybox-eth0macvlan-1 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:7B:0B:0B:02
          inet addr:123.11.11.2  Bcast:123.11.11.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```  

`eth0-macvlan` 에 설정한 서브넷에 맞게 아이피가 할당된 것을 확인할 수 있다. 
`ip` 명령어로 아이피 정보를 확인하면 아래와 같다. 

```bash
$ docker exec test-busybox-eth0macvlan-1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
62: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:7b:0b:0b:02 brd ff:ff:ff:ff:ff:ff
    inet 123.11.11.2/24 brd 123.11.11.255 scope global eth0
       valid_lft forever preferred_lft forever
```  

`eth0@if2` 를 봐서 부모 인터페이스는 `if2` 로 확인된다.  

다음으로 `ping` 명령을 통해 통신여부를 확인해본다. 
먼저 호스트의 `eth0`(`172.20.222.61)` 과 `google.com` 으로 수행하면 아래와 같다. 

```bash
$ docker exec test-busybox-eth0macvlan-1 ping -c 3 172.20.222.61
PING 172.20.222.61 (172.20.222.61): 56 data bytes

--- 172.20.222.61 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
$ docker exec test-busybox-eth0macvlan-1 ping -c 3 google.com
ping: bad address 'google.com'
```  

모두 통신이 불가능한 것을 확인할 수 있고, 
이로써 `macvlan` 은 기본적인 상태에서 서브 인터페이스의 외부와는 통신이 불가능한 것을 확인했다.  

`eth0-macvlan` 네트워크를 사용하는 컨테이너(`test-buxybox-eth0macvlan-2`)를 하나 더 생성한다. 
그리고 `test-busybox-eth0macvlan-1` 의 아이피(`123.11.11.2`)로 `ping` 테스트를 하면 아래와 같다. 

```bash
$ docker run \
> --rm -dit \
> --name test-busybox-eth0macvlan-2 \
> --network eth0-macvlan \
> busybox \
> ash
20fea3744d2fc6fb1f3384876d1bb7fe9dff79e43cdf0060de444a78589b5200
$ docker exec test-busybox-eth0macvlan-2 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:7B:0B:0B:03
          inet addr:123.11.11.3  Bcast:123.11.11.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:9 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:897 (897.0 B)  TX bytes:378 (378.0 B)
$ docker exec test-busybox-eth0macvlan-2 ping -c 3 123.11.11.2
PING 123.11.11.2 (123.11.11.2): 56 data bytes
64 bytes from 123.11.11.2: seq=0 ttl=64 time=0.066 ms
64 bytes from 123.11.11.2: seq=1 ttl=64 time=0.048 ms
64 bytes from 123.11.11.2: seq=2 ttl=64 time=0.071 ms

--- 123.11.11.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.048/0.061/0.071 ms
```  

정상적으로 통신이 이뤄지는 것을 확인할 수 있다. 
이로써 동일 `macvlan` 네트워크안에 있는 서브 인터페이스 간에는 통신이 가능한것을 확인했다.  

지금까지의 네트워크를 도식화 하면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/docker/practice_docker_networking_4.png)

#### VLAN Trunking with MACVLAN
현재 호스트의 `NIC` 인 `eth0` 에는 `eth0-macvlan` 이라는 도커 네트워크가 생성된 상태이다. 
여기서 네트워크를 분리하기위해 `eth0` 을 부모로 새로운 `macvlan` 도커 네트워크를 생성하면 아래와 같이 생성되지 않는다. 

```bash
$ docker network create -d macvlan \
>   --subnet=172.16.86.0/24 \
>   --gateway=172.16.86.1 \
>   -o parent=eth0 \
>   eth0-macvlan-2
Error response from daemon: network dm-fb28abd9dec8 is already using parent interface eth0
```  

부모 인터페이스로 사용중인 `NIC` 의 경우 아래와 같은 방법으로 별도의 `macvlan` 도커 네트워크를 생성할 수 있다. 

```bash
$ docker network create -d macvlan \
>   --subnet=123.12.11.0/24 \
>   --gateway=123.12.11.1 \
>   -o parent=eth0.10 \
>   eth0.10-macvlan
80474f8de27864c73bcd6b245a80f3cd5230b536ad3633b829f66bff2eaa5044
```  

그리고 호스트에서 `ifconfig` 로 `NIC` 를 조회하면 `eth0.10` 이라는 인터페이스가 조회하는 것을 확인할 수 있다. 

```bash
$ ifconfig
eth0.10: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::215:5dff:fe00:224  prefixlen 64  scopeid 0x20<link>
        ether 00:15:5d:00:02:24  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 21  bytes 1370 (1.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

`eth0.10-macvlan` 네트워크를 사용하는 2개의 컨테이너를 생성하고, 
아이피를 확인하면 아래와 같다. 

```bash
$ docker run \
> --rm -dit \
> --name test-busybox-eth0.10macvlan-1 \
> --network eth0.10-macvlan \
> busybox \
> ash
def2f93455954a853aa8c79078b5fbb9af9552f61a55e74e22ecf9dff1e6eef5
$ docker run \
> --rm -dit \
> --name test-busybox-eth0.10macvlan-2 \
> --network eth0.10-macvlan \
> busybox \
> ash
ff02da80d6fff469d65f300e1daada93f52a52359e8cbc7c5618fd68950edd6b
$ docker exec test-busybox-eth0.10macvlan-1 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:7B:0C:0B:02
          inet addr:123.12.11.2  Bcast:123.12.11.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

$ docker exec test-busybox-eth0.10macvlan-2 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:7B:0C:0B:03
          inet addr:123.12.11.3  Bcast:123.12.11.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```  

두 컨테이너 모두 아이피가 지정된 네트워크의 서브넷에 맞게 설정되었다.  

`ping` 명령으로 `eth0-macvlan` 네트워크를 사용하는 `test-busybox-eth0macvlan-1`(`123.11.11.2`)
컨테이너에 통신 가능여부를 테스트하면 아래와 같다. 

```bash
$ docker exec test-busybox-eth0.10macvlan-1 ping -c 3 123.11.11.2
PING 123.11.11.2 (123.11.11.2): 56 data bytes

--- 123.11.11.2 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
root@ubuntu-test:/home/ubuntu# docker exec test-busybox-eth0.10macvlan-2 ping -c 3 123.11.11.2
PING 123.11.11.2 (123.11.11.2): 56 data bytes

--- 123.11.11.2 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```  

통신이 되지 않는다. 
이번에는 `test-busybox-eth0.10macvlan-1`(`123.12.11.2`)와 `test-busybox-eth0.10macvlan-1`(`123.12.11.3`) 간에 
서로 `ping` 테스트를 수행하면 아래와 같다. 

```bash
$ docker exec test-busybox-eth0.10macvlan-1 ping -c 3 123.12.11.3
PING 123.12.11.3 (123.12.11.3): 56 data bytes
64 bytes from 123.12.11.3: seq=0 ttl=64 time=0.057 ms
64 bytes from 123.12.11.3: seq=1 ttl=64 time=0.065 ms
64 bytes from 123.12.11.3: seq=2 ttl=64 time=0.128 ms

--- 123.12.11.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.057/0.083/0.128 ms
$ docker exec test-busybox-eth0.10macvlan-2 ping -c 3 123.12.11.2
PING 123.12.11.2 (123.12.11.2): 56 data bytes
64 bytes from 123.12.11.2: seq=0 ttl=64 time=0.040 ms
64 bytes from 123.12.11.2: seq=1 ttl=64 time=0.053 ms
64 bytes from 123.12.11.2: seq=2 ttl=64 time=0.067 ms

--- 123.12.11.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.040/0.053/0.067 ms
```  

통신이 정상적으로 이뤄지는 것을 확인할 수 있다. 
이로써 `eth0-macvlan` 과 `eth0.10-macvlan` 은 모두 호스트의 `eth0` 를 부모 인터페이스로 만들어진 
서브 인터페이스이지만 서로 독립된 네트워크 공간을 갖는다는 것을 알 수 있다. 
이를 현재까지 네트워크를 도식화하면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/docker/practice_docker_networking_5.png)

### none
`none` 네트워크 드라이버는 각 컨테이너가 격리된 네트워크 영역을 가지지만, 
네트워크 인터페이스(`eth0`) 이 없는 상태로 컨테이너가 생성된다. 
그러므로 `loopback`(`lo`) 네트워크 인터페이스만 가지게 되므로 
컨테이너 내부 통신만 가능하고, 컨테이너 외부와의 통신은 단절된 상태로 컨테이너가 생성된다.  

`test-busybox-none` 이라는 네트워크가 `none` 으로 설정된 컨테이너를 실행하고, 
네트워크 인터페이스를 확인하면 아래와 같다. 

```bash
$ docker run --rm -dit --name test-busybox-none --network none busybox ash
7e727cf8b93533c1419dbbc1f3bf40c2c24734cb83ed793cb79c6a94c26ea9ae
$ docker exec test-busybox-none ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
$ docker exec test-busybox-none ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: sit0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0
```  

`loopback` 인터페이스만 존재하고 `eth0` 과 같은 이더넷 인테페이스는 컨테이너에 존재하지 않는다. 
즉 해당 컨테이너느 추가적으로 네트워클 인터페이스를 설정해 주지 않는 이상 
컨테이너 외부와 통신은 불가능한 상태이다.  


### overlay
[Overlay 네트워크]({{site.baseurl}}{% link _posts/docker/2020-02-04-docker-practice-overlaynetwork.md %})
포스트에서 `overlay` 네트워크에 대해 알아본 적이 있으면 해당 포스트를 참고 하도록 한다. 

<!-- 
중복되는 내용이 포함될 수 있지만 좀더 자세히 알아보기위해 다시 정리한다. 
기본적인 `overlay` 에 대한 설명은 위 링크를 참고하도록 한다. 
`overlay` 드라이버는 클러스터로 구성된 도커 호스트들간의 통신을 제공하고, 
로드밸런싱도 기능도 포함된 도커 네트워크 드라이버이다.  

먼저 테스트를 위해 `docker swarm init` 으로 `Swarm Cluster` 를 초기화를 진행한다. 
`overlay` 네트워크를 생성하기 위해서는 호스트에 `Swarm` 이 설정된 상태여야 한다.  

```bash
docker swarm init --advertise-addr 192.168.100.2
Swarm initialized: current node (v1k7f7fi9ou7s2fc85r0m47sg) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4yq91n4nqfjga7clkswr8ibzobe3y9qi3x8s1mx5gw4xtkd9x4-4eb27ulsouhjq3i2ie6yyq676 192.168.100.2:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```  

`Swarm` 을 설정한 후 `ifconfig` 로 네트워크 인터페이스를 조회하면 아래와 같다. 

```bash
$ ifconfig
docker_gwbridge: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 172.18.255.255
        inet6 fe80::42:90ff:fe93:64a0  prefixlen 64  scopeid 0x20<link>
        ether 02:42:90:93:64:a0  txqueuelen 0  (Ethernet)
        RX packets 32  bytes 2592 (2.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 32  bytes 2592 (2.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth0020689: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::ec33:44ff:feb0:6759  prefixlen 64  scopeid 0x20<link>
        ether ee:33:44:b0:67:59  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12  bytes 1012 (1012.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

.. 기타 생략 ..
```  

`Swarm` 을 설정하기 전과 비교했을 때, 
`docker_bridge` 와 `veth0020689` 이라는 네트워크 인터페이스가 생성됐다. 
`ip a` 명령으로 `veth0020689` 인터페이스는 `docker_bridge` 와 연결된 가상 인터페이스인것을 확인할 수 있다.  

```
$ ip a
9: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:90:93:64:a0 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker_gwbridge
       valid_lft forever preferred_lft forever
    inet6 fe80::42:90ff:fe93:64a0/64 scope link
       valid_lft forever preferred_lft forever
11: veth0020689@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP group default
    link/ether ee:33:44:b0:67:59 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::ec33:44ff:feb0:6759/64 scope link
       valid_lft forever preferred_lft forever

.. 기타 생략 ..
```  

그리고 `docker network ls` 명령으로 도커 네트워크를 조회하면 아래와 같다.  

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
b6481ef7ffdb        bridge              bridge              local
82413e200d1c        docker_gwbridge     bridge              local
78b4a0134731        host                host                local
s3rlc5284a7i        ingress             overlay             swarm
09f31cd303ff        none                null                local
```  

`docker_gwbridge` 와 `ingress` 라는 도커 네트워크가 추가되었다. 
두 도커 네트워크에 대한 자세한 설명은 상단 링크에서 확인 가능하다.  

`test-overlay` 라는 `overlay` 도커 네트워크를 생성한다. 

```bash
$ docker network create --driver overlay test-overlay
h2ncm6gou8wsze9v9t0b8sg69
$ docker network inspect -f '{{json .IPAM.Config }}' test-overlay
[{"Subnet":"10.0.4.0/24","Gateway":"10.0.4.1"}]
```  

`test-overlay` 는 `10.0.4.0/24` 의 아이피 대역을 갖는 것을 확인할 수 있다. 
그리고 `test-overlay` 네트워크를 사용하는 `test-busybox` 서비스도 생성한다.  

```bash
docker service create --name test-nginx --network test-overlay nginx:latest
ofeq9yi8zebg1kepwg8ec5wgr
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
$ docker exec -it `docker ps -q -f name=test-nginx` ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.0.4.27  netmask 255.255.255.0  broadcast 10.0.4.255
        ether 02:42:0a:00:04:1b  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.3  netmask 255.255.0.0  broadcast 172.18.255.255
        ether 02:42:ac:12:00:03  txqueuelen 0  (Ethernet)
        RX packets 903  bytes 8690096 (8.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 711  bytes 39923 (38.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
$ docker exec -it `docker ps -q -f name=test-nginx` route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.18.0.1      0.0.0.0         UG    0      0        0 eth1
10.0.4.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth1
```  

`test-nginx` 서비스에 실행되는 컨테이너에는 `eth0` ,`eth1` 네트워크 인터페이스가 있다. 
그리고 `route -n` 으로 확인하면 

이후 다시 전체적인 아키텍쳐 이해후 작성 필요 .. 



https://success.mirantis.com/article/networking
-->


---
## Reference
[Networking overview](https://docs.docker.com/network/)  
[Docker Swarm Reference Architecture: Exploring Scalable, Portable Docker Container Networks](https://success.mirantis.com/article/networking)  
[Docker container networking](http://docs.docker.oeynet.com/engine/userguide/networking/)  
[Plugins and Services](https://docs.docker.com/engine/extend/plugins_services/)  
[Use overlay networks](https://docs.docker.com/network/overlay/)  
[Understanding Docker Networking Drivers and their use cases](https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/)  
[Networking using a macvlan network](https://docs.docker.com/network/network-tutorial-macvlan/)  
[Bridge vs Macvlan](http://hicu.be/bridge-vs-macvlan)  