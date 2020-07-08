--- 
layout: single
classes: wide
title: "[Docker 실습] Dockerfile 의 ADD 와 COPY "
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Dockerfile 을 작성할때 비슷해 보이는 ADD 와 COPY 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - Dockerfile
---  

## ADD
- `ADD [--chown=<user>:<group>] <src> <dest>`
- `ADD` 명령어는 Host 의 파일(`<scr>`)을 컨테이너로 복사(`<dest>`)하는 명령어지만, 몇가지 기능을 추가로 가지고 있다.
- `<src>` 에 url 을 기제할 경우 파일을 다운로드해준다.
- `<src>` 에 잘 알려진 압출 포맷(ID, gzip, bzip, tar 등) 을 기제할 경우, 자동으로 압축을 해재한다. 
- url 의 경우 압축을 해제만 하고 풀지는 않는다.
- `<src>` 가 파일일 경우 `root:root` 의 소유와 기존권한을 갖는다.
- `<src>` 가 url 일 경우 `root:root` 의 소유와 600 권한을 갖는다.

## COPY
- `COPY [--chown=<user>:<group>] <src> <dest>`
- `COPY` 명령어 또한 Host 의 파일(`<scr>`)을 컨테이너로 복사(`<dest>`)하는 명령어다.
- `ADD` 명령어와 비슷해 보이지만, `ADD` 명령어에서 명시적이지 않게 지원하는 추가 기능으로 인해, 단순 파일을 Host -> 컨테이너로 복사하는 명령어가 추가로 개발되었다.
- `COPY` 는 다른 추가 기능 없이, 파일을 Host -> 컨테이너로 복사만 하는 명령어이다.
- 기존 파일권한 그대로 설정된다.

## ADD / COPY
- `Dockerfile` 로 컨테이너에 대한 명세를 작성할때, 어떤 명령어가 적합한지에 대해 고민이 있을 수 있다.
- 자신의 의도가 `ADD` 의 추가기능을 인지하고 있고, 그 기능을 사용하려고 한다면 `ADD` 명령어를 사용해도 된다.
- 대부분은 Host -> 컨테이너로 파일을 복사하는 경우이기 때문에 `COPY` 명령어를 권장한다.

---
## Reference
	