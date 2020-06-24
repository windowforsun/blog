--- 
layout: single
classes: wide
title: "[Kubernetes 개념] Kubernetes 설치하고 실행해 보기"
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

## Docker 설치
우선 `Kubernetes` 를 사용하기 위해서는 `Docker` 에대한 환경 구성이 필요하다. 
`Docker` 설치는 [여기](https://docs.docker.com/get-docker/)
에서 할 수 있다.  

설치가 완료되었다면 터미널에서 `docker version` 명령어를 통해 설치 여부를 다시한번 확인한다. 

```bash
# for linux
$ sudo docker version
...

$ docker version
Client: Docker Engine - Community
 Version:           19.03.8
 API version:       1.40
 Go version:        go1.12.17
 Git commit:        afacb8b
 Built:             Wed Mar 11 01:23:10 2020
 OS/Arch:           windows/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.8
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.17
  Git commit:       afacb8b
  Built:            Wed Mar 11 01:29:16 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v1.2.13
  GitCommit:        7ad184331fa3e55e52b890ea95e65ba581ae3429
 runc:
  Version:          1.0.0-rc10
  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683

```  

## Docker Container 실행
먼저 `Docker` 컨테이너 이미지를 실행해 본다. 
실행할 이미지는 `Web Server` 로 많이 사용하는 `Nginx` 이미지로 명령어는 아래와 같다.

```bash
$ docker container run -p 9999:80 --name hello nginx:latest
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
8559a31e96f4: Pulling fs layer
8d69e59170f7: Pulling fs layer

.. 생략 ..

d1f5ff4f210d: Pull complete
1e22bfa8652e: Pull complete
Digest: sha256:21f32f6c08406306d822a0e6e8b7dc81f53f336570e852e25fbe1e3e3d0d0133
Status: Downloaded newer image for nginx:latest

```  

웹 브라우저에서 `http://localhost:9999` 으로 접속하면 아래와 같은 문자가 출력되는 페이지를 확인 할 수 있다. 

```
Welcome to nginx!

.. 생략 ..
```  




---
## Reference
[쿠버네티스를 활용한 클라우드 네이티브 데브옵스: 클라우드 환경에서 모던 애플리케이션 빌드, 배포, 스케일링하기](https://play.google.com/store/books/details/%EC%A1%B4_%EC%96%B4%EB%9F%B0%EB%93%A4_%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4%EB%A5%BC_%ED%99%9C%EC%9A%A9%ED%95%9C_%ED%81%B4%EB%9D%BC%EC%9A%B0%EB%93%9C_%EB%84%A4%EC%9D%B4%ED%8B%B0%EB%B8%8C_%EB%8D%B0%EB%B8%8C%EC%98%B5%EC%8A%A4?id=_p_HDwAAQBAJ&hl=ko)
