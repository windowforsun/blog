--- 
layout: single
classes: wide
title: "[Linux 실습] tmux 를 설치하고 사용하자"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'tmux 를 설치, 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - tmux
---  

## 환경
- CentOS 6

## tmux 란
- tmux 는 여러개의 터미널을 실행 할 수 있는 멀티플렛서 이다.
- 화면을 분할하고 구성하는 등의 attach/detach 가 자유롭다.
- tmux 와 Vim 을 사용해서 IDE 를 만들어 사용하기도 할만큼 커스텀이 매우 자유롭다.

## tmux 의 구성
### session
- tmux 의 실행 단위이다.
- 여러개의 window 로 구성된다.

### window
- 한개의 터미널을 지칭하는 화면이다.
- 세션 내에서 탭처럼 사용할 수 있다.

### pane
- 하나의 window 내에서 화면 분할을 할수 있다.

### status bar
- 화면 아래 표시되는 상태 막대이다.



---
## Reference
[CentOS 6 / CentOS 7에서 최신 tmux를 yum으로 넣은 tmux과 공존시킨 상태에서 설치](https://www.terabo.net/blog/install-latest-tmux-on-centos-6-and-7/)  
[Install tmux on Centos release 6.5](https://gist.github.com/cdkamat/fdb136230f67c563c7b1)  
[터미널 멀티플렉서 tmux를 배워보자](http://blog.b1ue.sh/2016/10/10/tmux-tutorial/)  
[TTY 멀티플랙서 tmux](https://blog.outsider.ne.kr/699)  
[tmux 입문자 시리즈 요약](https://edykim.com/ko/post/tmux-introductory-series-summary/)  