--- 
layout: single
classes: wide
title: "[Git] Git Bash 에서 Tmux 설치 및 설정"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'Windows 환경에서 Git Bash 로 Tmux 를 사용해보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - Git Bash
    - Tmux
---  

## 환경
- Windows 10

## Git Bash, Tmux
- Tmux 는 Linux 기반 터미널 멀티플렉서이다.
- 화면을 분할하고 구성에 자유도가 있어, Bash 사용에 있어서 아주 편리하다.
- Windows 에서 Git Bash 를 사용할 때, 불편함이 있어 Git Bahs 에 Tmux 를 설치하고 사용하는 법에 대해 알아본다.

## 설치
- [msys2](https://www.msys2.org/) 를 메뉴얼대로 설치해 준다.
	- `pacman -Syu`
	- `pacman -Su`
- `pacman` 패키지 관리자를 통해 Tmux 를 설치해 준다. `pacman -S tmux`
- `msys2` 설치 경로(`C:\msys64\usr\bin`)로 가서 `tmux.exe` 와 `msys-event-2-x-x.dll` 을 복사한다.
- Git 설치 경로(`C:\Program Files\Git\usr\bin`) 에 붙여넣기 해준다.
- Git Bash 에서 Tmux 로 Bash 창을 유동적으로 사용할 수 있다.
- 만약 `tmux` 명령어에도 동작하지 않는다면 Git 을 업데이트 해준다.
	- 현재 `Git 2.16` 버전 이상이라면 `git update-git-for-windows` 명령어로 업데이트
	- 현재 `Git 2.14 ~ 2.16` 버전이라면 `git update` 명령어로 업데이트 
- 업데이트 시에는 동일한(`msys2` 설치 제외) 과정을 반복한다.

## 설정하기
- 기본 설정외 커스텀한 설정을 원할 경우, 방식은 기존 Linux 에서 설정방법과 동일하다.
- `vi ~/.tmux.conf` 명령어로 설정을 작성한다.
- `tmux source-file ~/.tmux.conf` 명령어로 설정을 적용한다.


---
 
## Reference
[How to run Tmux in GIT Bash on Windows](https://blog.pjsen.eu/?p=440)  
[How to upgrade to the latest version of Git on Windows? Still showing older version](https://stackoverflow.com/questions/13790592/how-to-upgrade-to-the-latest-version-of-git-on-windows-still-showing-older-vers)  


