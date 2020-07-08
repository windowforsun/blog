--- 
layout: single
classes: wide
title: "[Linux 실습] tmuxinator 를 설치하고 사용하자"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'tmuxinator 를 설치, 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - tmux
  - Tmuxinator
  - Ruby
  - RVM
---  

## 환경
- CentOS 6
- tmux 2.3
- RVM
- Ruby 2.4

## 사전 작업
- tmuxinator 설치를 위해서는 [ruby 2.4]({{site.baseurl}}{% link _posts/linux/2019-05-25-linux-practice-ruby.md %}) 이상 버전이 설치되어야 한다.

## Tmuxinator 란


## Tmuxinator 설치

```
[windowforsun_com@windowforsun ~]$ gem install tmuxinator
Fetching erubis-2.7.0.gem
Fetching xdg-2.2.5.gem
Fetching thor-0.20.3.gem
Fetching tmuxinator-1.1.1.gem
Successfully installed erubis-2.7.0
Successfully installed thor-0.20.3
Development of this project has been transfered here: https://github.com/bkuhlmann/xdg. This will be the last version supported within the 2.2.x series. Be aware major breaking changes will be introduced in the upcoming 3.0.0 release (2019-06-01).
Successfully installed xdg-2.2.5

    __________________________________________________________
    ..........................................................

    Thank you for installing tmuxinator.

    Make sure that you've set these variables in your ENV:

      $EDITOR, $SHELL

    You can run `tmuxinator doctor` to make sure everything is set.
    Happy tmuxing with tmuxinator!

    ..........................................................
    __________________________________________________________

Successfully installed tmuxinator-1.1.1
Parsing documentation for erubis-2.7.0
Installing ri documentation for erubis-2.7.0
Parsing documentation for thor-0.20.3
Installing ri documentation for thor-0.20.3
Parsing documentation for xdg-2.2.5
Installing ri documentation for xdg-2.2.5
Parsing documentation for tmuxinator-1.1.1
Installing ri documentation for tmuxinator-1.1.1
Done installing documentation for erubis, thor, xdg, tmuxinator after 2 seconds
4 gems installed
```  

## Tmuxinator 사용하기

- tmuxinator 를 사용하기 위해서는 쉘의 기본 에디터를 `vim`으로 변경해 주어야 한다.

```
[windowforsun_com@windowforsun ~]$ export EDITOR='vim'
```  

- 새로운 tmuxinator 프로젝트 생성
	- `tmuxinator new <project-name>` 을 통해 새로운 프로젝트를 생성한다.

```
[windowforsun_com@windowforsun ~]$ tmuxinator new test_project
```  

- 기본으로 생성된 tmuxinator 프로젝트 설정 파일
	- tmux 센션의 정보를 저장하고 있는 yaml 파일이다.
	- `./tmuxinator/` 경로 혹은 `~/.config/tmuxinator/` 에 생성된다.

```
# /home/windowforsun_com/.config/tmuxinator/test_project.yml

name: test_project
root: ~/

# Optional tmux socket
# socket_name: foo

# Note that the pre and post options have been deprecated and will be replaced by
# project hooks.

# Project hooks
# Runs on project start, always
# on_project_start: command
# Run on project start, the first time
# on_project_first_start: command
# Run on project start, after the first time
# on_project_restart: command
# Run on project exit ( detaching from tmux session )
# on_project_exit: command
# Run on project stop
# on_project_stop: command

# Runs in each window and pane before window/pane specific commands. Useful for setting up interpreter versions.
# pre_window: rbenv shell 2.0.0-p247

# Pass command line options to tmux. Useful for specifying a different tmux.conf.
# tmux_options: -f ~/.tmux.mac.conf

# Change the command to call tmux.  This can be used by derivatives/wrappers like byobu.
# tmux_command: byobu

# Specifies (by name or index) which window will be selected on project startup. If not set, the first window is used.
# startup_window: editor

# Specifies (by index) which pane of the specified window will be selected on project startup. If not set, the first pane is used.
# startup_pane: 1

# Controls whether the tmux session should be attached to automatically. Defaults to true.
# attach: false

windows:
  - editor:
      layout: main-vertical
      # Synchronize all panes of this window, can be enabled before or after the pane commands run.
      # 'before' represents legacy functionality and will be deprecated in a future release, in favour of 'after'
      # synchronize: after
      panes:
        - vim
        - guard
  - server: bundle exec rails s
  - logs: tail -f log/development.log
```  

- tmuxinator 프로젝트 설정 파일의 기본적은 구조는 아래와 같다.

```
# /home/windowforsun_com/.config/tmuxinator/test_project.yml

# 지정된 프로젝트 이름
name: test_project
# tmux 가 시작 되어야 하는 기본 폴더 위치
root: ~/

# tmux 세션을 시작 할 때 만들 윈도우 및 팬, 레아이웃, 명령어를 지정
 # 3개의 윈도우가 실행 된다.
windows:
  # editor 윈도우에는 2개의 팬이 존재한다.
  - editor:
      layout: main-vertical
      panes:
        - vim
        - guard
  # server 윈도우에서는 bundle 명령어를 실행한다.
  - server: bundle exec rails s
  # logs 윈도우에서는 tail 로그를 출력한다.
  - logs: tail -f log/development.log
```  

## Tmuxinator 레이아웃 설정하기

- tmux 를 켜고 tmux 명렁어를 통해 필요한 레이아웃을 구성한다.

![tmuxinator layout 1]({{site.baseurl}}/img/linux/tmuxinator-layout-1.png)

- `tmux list-windows` 명령어를 통해 현재 레아이웃을 출력한다.

```
[windowforsun_com@windowforsun ~]$ tmux list-windows
1: bash* (4 panes) [117x61] [layout 3c59,117x61,0,0[117x30,0,0{58x30,0,0,11,58x30,59,0,14},117x30,0,31{58x30,0,31,12,58x30,59,31,13}]] @6 (active)
```  

- 출력된 레아이웃으로 만들 tmuxinator 프로젝트를 생성한다.

```
[windowforsun_com@windowforsun ~]$ tmuxinator new layout_project
```  

- 위에서 출력했던 레아이웃 정보를 아래와 같이 복사하고 구성했던 4개의 팬에 각각 1, 2, 3, 4를 출력하도록 명령어를 추가한다.

```
# /home/windowforsun_com/.config/tmuxinator/layout_project.yml

name: layout_project
root: ~/

windows:
        - local:
                layout: 3c59,117x61,0,0[117x30,0,0{58x30,0,0,11,58x30,59,0,14},117x30,0,31{58x30,0,31,12,58x30,59,31,13}]
                panes:
                        - echo 1
                        - echo 2
                        - echo 3
                        - echo 4
```  

- 위에서 생성한 tmuxinator 프로젝트를 시작한다.

```
[windowforsun_com@windowforsun ~]$ tmuxinator start layout_project
```  

![tmuxinator layout 1]({{site.baseurl}}/img/linux/tmuxinator-layout-2.png)

## Tmuxinator 명령어

- `tmuxinator list` : 프로젝트 리스트를 출력한다.

- `tmuxinator new <project-name>` : 프로젝트를 새로 생성한다.

- `tmuxinator start <project-name>` : 프로젝트를 실행 시킨다.

- `tmuxinator stop <project-name>` : 현재 실행 중인 프로젝트를 중지 시킨다.

- `tmuxinator delete <project-name>` : 해당 프로젝트를 삭제 한다.

- `tmuxinator copy <exist-project-name> <new-project-name>` : 해당 프로젝트를 복사한다.

- `tmuxinator` : tmuxinator 관련 명령어 리스트를 출력한다.

---
## Reference
[How to Install Ruby on CentOS/RHEL 7/6](https://tecadmin.net/install-ruby-latest-stable-centos/)  
[CentOS6 ruby-2.3 설치](https://zetawiki.com/wiki/CentOS6_ruby-2.3_%EC%84%A4%EC%B9%98)  
[tmuxinator로 tmux 세션을 관리하자](https://blog.outsider.ne.kr/1167)  
[tmuxinator: Save tmux pane and window layouts](https://fabianfranke.de/use-tmuxinator-to-recreate-tmux-panes-and-windowstmuxinator-save-tmux-pane-and-window-layouts/)  