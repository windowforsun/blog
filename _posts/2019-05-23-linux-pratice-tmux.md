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
- tmux 2.3

## tmux 란
- tmux 는 여러개의 터미널을 실행 할 수 있는 멀티플렉서 이다.
- 화면을 분할하고 구성하는 등의 attach/detach 가 자유롭다.
- tmux 와 Vim 을 사용해서 IDE 를 만들어 사용하기도 할만큼 커스텀이 매우 자유롭다.

## tmux 의 구성 및 용어
### session
- tmux 의 실행 단위이다.
- attach 와 detach 가 이루어 진다.
- 세션은 detach 하더라도 백그라운드에서 계속해서 실행 된다.
- 여러개의 window 로 구성된다.

### window
- 세션안에 존재하는 탭 같은 기능이다.
- 하나의 세션에는 여러개의 윈도우를 가질 수 있다.
- 세션 안에서 윈도우를 만들고 전환할 수 있다.

### pane
- 하나의 윈도우 안에 존재하는 화면 단위이다.
- 윈도우 안에서 분할하여 여러개의 팬을 가질 수 있다.

### status bar
- 화면 아래 표시되는 상태 막대이다.

### prefix
- 단축키를 입력하기 전에 먼저 입력해야 하는 키 조합이다.
- tmux 의 기본 프리픽스는 `ctrl + b` 이다.
- 단축키가 `y` 라면 'ctrl + b` 를 입력 후, `y` 를 눌러야 한다.


## tmux 설치

- Centos 6 버전에서 `yum` 을 통해 받은 tmux 는 1.8 버전 이기때문에 2.3 버전에 대한방법을 기술한다.

- tmux 설치를 위한 의존성을 설치해 준다.

```
[root@windowforsun ~]# yum install gcc kernel-devel make ncurses-devel
Loaded plugins: fastestmirror, security
Setting up Install Process
Determining fastest mirrors
// 생략
Package 1:make-3.81-23.el6.x86_64 already installed and latest version
Resolving Dependencies
// 생략 
Dependencies Resolved

=====================================================================================================================
 Package                       Arch                  Version                            Repository              Size
=====================================================================================================================
Installing:
 gcc                           x86_64                4.4.7-23.el6                       base                    10 M
 kernel-devel                  x86_64                2.6.32-754.14.2.el6                updates                 11 M
 ncurses-devel                 x86_64                5.7-4.20090207.el6                 base                   641 k
Installing for dependencies:
 cloog-ppl                     x86_64                0.15.7-1.2.el6                     base                    93 k
 cpp                           x86_64                4.4.7-23.el6                       base                   3.7 M
 glibc-devel                   x86_64                2.12-1.212.el6_10.3                updates                991 k
 glibc-headers                 x86_64                2.12-1.212.el6_10.3                updates                620 k
 kernel-headers                x86_64                2.6.32-754.14.2.el6                updates                4.6 M
 libgomp                       x86_64                4.4.7-23.el6                       base                   135 k
 mpfr                          x86_64                2.4.1-6.el6                        base                   157 k
 ppl                           x86_64                0.10.2-11.el6                      base                   1.3 M

Transaction Summary
=====================================================================================================================
Install      11 Package(s)

Total download size: 33 M
Installed size: 66 M
Is this ok [y/N]: y
// 생략
Installed:
  gcc.x86_64 0:4.4.7-23.el6   kernel-devel.x86_64 0:2.6.32-754.14.2.el6   ncurses-devel.x86_64 0:5.7-4.20090207.el6

Dependency Installed:
  cloog-ppl.x86_64 0:0.15.7-1.2.el6                         cpp.x86_64 0:4.4.7-23.el6
  glibc-devel.x86_64 0:2.12-1.212.el6_10.3                  glibc-headers.x86_64 0:2.12-1.212.el6_10.3
  kernel-headers.x86_64 0:2.6.32-754.14.2.el6               libgomp.x86_64 0:4.4.7-23.el6
  mpfr.x86_64 0:2.4.1-6.el6                                 ppl.x86_64 0:0.10.2-11.el6

Complete!
```  

- LIBEVENT 파일 다운

```
[root@windowforsun ~]# curl -OL https://github.com/libevent/libevent/releases/download/release-2.0.22-stable/libevent-2.0.22-stable.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  834k  100  834k    0     0   936k      0 --:--:-- --:--:-- --:--:-- 2378k
```  

- LIBEVENT 압축 해제

```
[root@windowforsun ~]# tar -xf libevent-2.0.22-stable.tar.gz
``` 

- LIBEVENT 폴더로 이동

```
[root@windowforsun ~]# cd libevent-2.0.22-stable
```  

- LIBEVENT 설치

```
[root@windowforsun libevent-2.0.22-stable]# ./configure --prefix=/usr/local
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
// 생략
checking that generated files are newer than configure... done
configure: creating ./config.status
config.status: creating libevent.pc
config.status: creating libevent_openssl.pc
config.status: creating libevent_pthreads.pc
config.status: creating Makefile
config.status: creating include/Makefile
config.status: creating test/Makefile
config.status: creating sample/Makefile
config.status: creating config.h
config.status: executing depfiles commands
config.status: executing libtool commands


[root@windowforsun libevent-2.0.22-stable]# make
test -d include/event2 || /bin/mkdir -p include/event2
/bin/sed -f ./make-event-config.sed < config.h > include/event2/event-config.hT
mv -f include/event2/event-config.hT include/event2/event-config.h
make  all-recursive
make[1]: Entering directory `/root/libevent-2.0.22-stable'
Making all in .
// 생략
/bin/sh ../libtool --tag=CC   --mode=link gcc  -g -O2 -Wall -fno-strict-aliasing    -o regress regress-regress.o regress-regress_buffer.o regress-regress_http.o regress-regress_dns.o regress-regress_testutils.o regress-regress_rpc.o regress-regress.gen.o regress-regress_et.o regress-regress_bufferevent.o regress-regress_listener.o regress-regress_util.o regress-tinytest.o regress-regress_main.o regress-regress_minheap.o regress-regress_thread.o     ../libevent.la ../libevent_pthreads.la   -lrt
libtool: link: gcc -g -O2 -Wall -fno-strict-aliasing -o .libs/regress regress-regress.o regress-regress_buffer.o regress-regress_http.o regress-regress_dns.o regress-regress_testutils.o regress-regress_rpc.o regress-regress.gen.o regress-regress_et.o regress-regress_bufferevent.o regress-regress_listener.o regress-regress_util.o regress-tinytest.o regress-regress_main.o regress-regress_minheap.o regress-regress_thread.o  ../.libs/libevent.so ../.libs/libevent_pthreads.so -lrt -Wl,-rpath -Wl,/usr/local/lib
make[3]: Leaving directory `/root/libevent-2.0.22-stable/test'
make[2]: Leaving directory `/root/libevent-2.0.22-stable/test'
make[1]: Leaving directory `/root/libevent-2.0.22-stable'


[root@windowforsun libevent-2.0.22-stable]# make install
// 생략
----------------------------------------------------------------------
Libraries have been installed in:
   /usr/local/lib

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the `-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the `LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the `LD_RUN_PATH' environment variable
     during linking
   - use the `-Wl,-rpath -Wl,LIBDIR' linker flag
   - have your system administrator add LIBDIR to `/etc/ld.so.conf'

See any operating system documentation about shared libraries for
more information, such as the ld(1) and ld.so(8) manual pages.
----------------------------------------------------------------------
 // 생략 
make[4]: Leaving directory `/root/libevent-2.0.22-stable/test'
make[3]: Leaving directory `/root/libevent-2.0.22-stable/test'
make[2]: Leaving directory `/root/libevent-2.0.22-stable/test'
make[1]: Leaving directory `/root/libevent-2.0.22-stable'

```  

- tmux 파일 다운

```
[root@windowforsun ~]# curl -OL https://github.com/tmux/tmux/releases/download/2.3/tmux-2.3.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  462k  100  462k    0     0   618k      0 --:--:-- --:--:-- --:--:--  618k
```  

- tmux 파일 압축 해제

```
[root@windowforsun ~]# tar -xf tmux-2.3.tar.gz
```  

- tmux 디렉토리로 이동

```
[root@windowforsun ~]# cd tmux-2.3
```  

- tmux 설치

```
[root@windowforsun tmux-2.3]# LDFLAGS="-L/usr/local/lib -Wl,-rpath=/usr/local/lib" ./configure --prefix=/usr/local
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... /bin/mkdir -p
// 생략
checking that generated files are newer than configure... done
configure: creating ./config.status
config.status: creating Makefile
config.status: executing depfiles commands


[root@windowforsun tmux-2.3]# make                                                                                  
depbase=`echo alerts.o | sed 's|[^/]*$|.deps/&|;s|\.o$||'`;\
        gcc -DPACKAGE_NAME=\"tmux\" -DPACKAGE_TARNAME=\"tmux\" -DPACKAGE_VERSION=\"2.3\" -DPACKAGE_STRING=\"tmux\ 2.3\" -DPACKAGE_BUGREPORT=\"\" -DPACKAGE_URL=\"\" -DPACKAGE=\"tmux\" -DVERSION=\"2.3\" -DSTDC_HEADERS=1 -DHAVE_SYS_TYPES_H=1 -DHAVE_SYS_STAT_H=1 -DHAVE_STDLIB_H=1 -DHAVE_STRING_H=1 -DHAVE_MEMORY_H=1 -DHAVE_STRINGS_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_STDINT_H=1 -DHAVE_UNISTD_H=1 -DHAVE_DIRENT_H=1 -DHAVE_FCNTL_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_PATHS_H=1 -DHAVE_PTY_H=1 -DHAVE_STDINT_H=1 -DHAVE_SYS_DIR_H=1 -DHAVE_TERM_H=1 -DHAVE_DIRFD=1 -DHAVE_FLOCK=1 -DHAVE_PRCTL=1 -DHAVE_SYSCONF=1 -DHAVE_CFMAKERAW=1 -DHAVE_NCURSES_H=1 -DHAVE_B64_NTOP=1 -DHAVE_FORKPTY=1 -DHAVE_DAEMON=1 -DHAVE_SETENV=1 -DHAVE_ASPRINTF=1 -DHAVE_STRCASESTR=1 -DHAVE_STRSEP=1 -DHAVE_CFMAKERAW=1 -DHAVE_OPENAT=1 -DHAVE_DECL_OPTARG=1 -DHAVE_DECL_OPTIND=1 -DHAVE_DECL_OPTRESET=0 -DHAVE_BSD_TYPES=1 -DHAVE___PROGNAME=1 -DHAVE_PROGRAM_INVOCATION_SHORT_NAME=1 -DHAVE_PR_SET_NAME=1 -DHAVE_PROC_PID=1  -I.   -DTMUX_CONF="\"/etc/tmux.conf\""  -iquote.    -D_GNU_SOURCE -std=gnu99 -O2     -MT alerts.o -MD -MP -MF $depbase.Tpo -c -o alerts.o alerts.c &&\
        mv -f $depbase.Tpo $depbase.Po
// 생략
depbase=`echo compat/reallocarray.o | sed 's|[^/]*$|.deps/&|;s|\.o$||'`;\
        gcc -DPACKAGE_NAME=\"tmux\" -DPACKAGE_TARNAME=\"tmux\" -DPACKAGE_VERSION=\"2.3\" -DPACKAGE_STRING=\"tmux\ 2.3\" -DPACKAGE_BUGREPORT=\"\" -DPACKAGE_URL=\"\" -DPACKAGE=\"tmux\" -DVERSION=\"2.3\" -DSTDC_HEADERS=1 -DHAVE_SYS_TYPES_H=1 -DHAVE_SYS_STAT_H=1 -DHAVE_STDLIB_H=1 -DHAVE_STRING_H=1 -DHAVE_MEMORY_H=1 -DHAVE_STRINGS_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_STDINT_H=1 -DHAVE_UNISTD_H=1 -DHAVE_DIRENT_H=1 -DHAVE_FCNTL_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_PATHS_H=1 -DHAVE_PTY_H=1 -DHAVE_STDINT_H=1 -DHAVE_SYS_DIR_H=1 -DHAVE_TERM_H=1 -DHAVE_DIRFD=1 -DHAVE_FLOCK=1 -DHAVE_PRCTL=1 -DHAVE_SYSCONF=1 -DHAVE_CFMAKERAW=1 -DHAVE_NCURSES_H=1 -DHAVE_B64_NTOP=1 -DHAVE_FORKPTY=1 -DHAVE_DAEMON=1 -DHAVE_SETENV=1 -DHAVE_ASPRINTF=1 -DHAVE_STRCASESTR=1 -DHAVE_STRSEP=1 -DHAVE_CFMAKERAW=1 -DHAVE_OPENAT=1 -DHAVE_DECL_OPTARG=1 -DHAVE_DECL_OPTIND=1 -DHAVE_DECL_OPTRESET=0 -DHAVE_BSD_TYPES=1 -DHAVE___PROGNAME=1 -DHAVE_PROGRAM_INVOCATION_SHORT_NAME=1 -DHAVE_PR_SET_NAME=1 -DHAVE_PROC_PID=1  -I.   -DTMUX_CONF="\"/etc/tmux.conf\""  -iquote.    -D_GNU_SOURCE -std=gnu99 -O2     -MT compat/reallocarray.o -MD -MP -MF $depbase.Tpo -c -o compat/reallocarray.o compat/reallocarray.c &&\
        mv -f $depbase.Tpo $depbase.Po
gcc  -D_GNU_SOURCE -std=gnu99 -O2      -L/usr/local/lib -Wl,-rpath=/usr/local/lib   -o tmux alerts.o arguments.o attributes.o cfg.o client.o cmd-attach-session.o cmd-bind-key.o cmd-break-pane.o cmd-capture-pane.o cmd-choose-buffer.o cmd-choose-client.o cmd-choose-tree.o cmd-clear-history.o cmd-command-prompt.o cmd-confirm-before.o cmd-copy-mode.o cmd-detach-client.o cmd-display-message.o cmd-display-panes.o cmd-find.o cmd-find-window.o cmd-if-shell.o cmd-join-pane.o cmd-kill-pane.o cmd-kill-server.o cmd-kill-session.o cmd-kill-window.o cmd-list-buffers.o cmd-list-clients.o cmd-list-keys.o cmd-list-panes.o cmd-list-sessions.o cmd-list-windows.o cmd-list.o cmd-load-buffer.o cmd-lock-server.o cmd-move-window.o cmd-new-session.o cmd-new-window.o cmd-paste-buffer.o cmd-pipe-pane.o cmd-queue.o cmd-refresh-client.o cmd-rename-session.o cmd-rename-window.o cmd-resize-pane.o cmd-respawn-pane.o cmd-respawn-window.o cmd-rotate-window.o cmd-run-shell.o cmd-save-buffer.o cmd-select-layout.o cmd-select-pane.o cmd-select-window.o cmd-send-keys.o cmd-set-buffer.o cmd-set-environment.o cmd-set-hook.o cmd-set-option.o cmd-show-environment.o cmd-show-messages.o cmd-show-options.o cmd-source-file.o cmd-split-window.o cmd-string.o cmd-swap-pane.o cmd-swap-window.o cmd-switch-client.o cmd-unbind-key.o cmd-wait-for.o cmd.o colour.o control.o control-notify.o environ.o format.o grid-view.o grid.o hooks.o input-keys.o input.o job.o key-bindings.o key-string.o layout-custom.o layout-set.o layout.o log.o mode-key.o names.o notify.o options-table.o options.o paste.o proc.o resize.o screen-redraw.o screen-write.o screen.o server-client.o server-fn.o server.o session.o signal.o status.o style.o tmux.o tty-acs.o tty-keys.o tty-term.o tty.o utf8.o window-choose.o window-clock.o window-copy.o window.o xmalloc.o xterm-keys.o osdep-linux.o   compat/imsg.o compat/imsg-buffer.o compat/closefrom.o  compat/getprogname.o compat/setproctitle.o  compat/strlcat.o compat/strlcpy.o  compat/fgetln.o compat/fparseln.o compat/getopt.o   compat/vis.o compat/unvis.o compat/strtonum.o    compat/reallocarray.o  -lutil -lncurses   -levent -lrt  -lncurses -lresolv


[root@windowforsun tmux-2.3]# make install
make[1]: Entering directory `/root/tmux-2.3'
// 생략
make[2]: Leaving directory `/root/tmux-2.3'
make[1]: Nothing to be done for `install-data-am'.
make[1]: Leaving directory `/root/tmux-2.3'


[root@windowforsun ~]# pkill tmux
```  

- 설치 시에 사용했던 터미널을 닫고 새로운 터미널을 열어준다.
- 설치된 tmux 버전을 확인한다.

```
[windowforsun_com@windowforsun ~]$ tmux -V
tmux 2.3
```  

## tmux 설정

- `~/.tmux.conf` 로 tmux 설정 파일을 생성한다.

```
[windowforsun_com@windowforsun ~]$ vi ~/.tmux.conf
```  

- 아래 설정파일 내용을 추가한다.

```
# 0 is too far from ` ;)
set -g base-index 1

# Automatically set window title
set-window-option -g automatic-rename on
set-option -g set-titles on

#set -g default-terminal screen-256color
set -g status-keys vi
set -g history-limit 10000

setw -g mode-keys vi
setw -g mouse on
setw -g monitor-activity on

bind-key v split-window -h
bind-key s split-window -v

bind-key J resize-pane -D 5
bind-key K resize-pane -U 5
bind-key H resize-pane -L 5
bind-key L resize-pane -R 5

bind-key M-j resize-pane -D
bind-key M-k resize-pane -U
bind-key M-h resize-pane -L
bind-key M-l resize-pane -R

# Vim style pane selection
bind h select-pane -L
bind j select-pane -D 
bind k select-pane -U
bind l select-pane -R

# Use Alt-vim keys without prefix key to switch panes
bind -n M-h select-pane -L
bind -n M-j select-pane -D 
bind -n M-k select-pane -U
bind -n M-l select-pane -R

# Use Alt-arrow keys without prefix key to switch panes
bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

# Shift arrow to switch windows
bind -n S-Left  previous-window
bind -n S-Right next-window

# No delay for escape key press
set -sg escape-time 0

# Reload tmux config
bind r source-file ~/.tmux.conf

# THEME
set -g status-bg black
set -g status-fg white
set -g window-status-current-bg white
set -g window-status-current-fg black
set -g window-status-current-attr bold
set -g status-interval 60
set -g status-left-length 30
set -g status-left '#[fg=green](#S) #(whoami)'
set -g status-right '#[fg=yellow]#(cut -d " " -f 1-3 /proc/loadavg)#[default] #[fg=white]%H:%M#[default]'

bind-key y set-window-option synchronize-panes
```  

- tmux.conf 파일 적용하기

```
[windowforsun_com@dontrise ~]$ tmux source-file ~/.tmux.conf
```  

## tmux 명령어
- tmux 의 명령어는 `<prefix> + <명령 키>`로 사용 가능하다.

```
(ctrl + b) + <명령 키>
```  

- 직접 명령어를 입력할 때는 `<prefix> + :` 를 입력한 후 명령어를 입력한다.

```
(ctrl + b) + :
```  

### session 관련
- `tmux` : 새로운 세션을 생성한다.
	- 이름 없이 세션이 생성되었기 때문에 세션이름은 숫자로 지정된다.
	
- `tmux new -s <session-name>` : 이름을 지정해 새로운 세션을 생성한다.

- `tmux ls` : 현재 생성된 세션의 목록을 확인할 수 있다.

- `tmux attach -t <session-name>` : 백그라운드에 해당 있는 세션을 다시 사용한다. (attach)

- `<prefix> + d` : 현재 사용 중인 세션을 백그라운드로 하고, 세션을 빠져 나온다. (detach)

- `exit` : 현재 사용 중인 세션을 종료 한다.

- `<prefix> + $` : 현재 세션 이름을 수정 한다.

- `tmux kill-session -t <session-name>` : 해당 세션을 종료 시킨다.

### window 관련
- `<prefix> + c` : 세션에서 새로운 윈도우를 생성한다.

- `<prefix> + &` : 현재 윈도우를 종료한다.

- `<prefix> + <0-9>` : 윈도우 번호로 이동한다.

- `<prefix> + n` : 다음 윈도우로 이동한다.

- `<prefix> + p` : 이전 윈도우로 이동한다.

- `<prefix> + l` : 마지막 윈도우로 이동한다.

- `<prefix> + w` : 이동할 윈도우를 리스트에서 선택한다.

- `<prefix> + f` : 이동할 이름으로 검색해서 이동한다.
 
- `<prefix> + ,` : 현재 윈도우 이름을 수정 한다.

- `tmux kill-window -t <window-name>` : 해당 윈도우를 종료 시킨다.

- `tmux new -s <session-name> -n <window-name>` : 세션과 윈도우를 같이 생성한다.

### pane 관련
- `<prefix> + %` : 윈도우를 횡으로 분할 한다.

- `<prefix> + "` : 윈도우를 종으로 분할 한다.

- `<prefix> + q` : 윈도우에 분활된 팬의 번호를 보여준다.

- `<prefix> + 방향키` : 분할된 팬을 화살표 방향으로 이동한다.

- `<prefix> + <pane-number>` : 분할된 팬을 번호로 이동한다.

- `<prefix> + o` : 다음 팬으로 이동한다.

- `<prefix> + {` : 왼쪽 팬으로 이동한다.

- `<prefix> + }` : 오른쪽 팬으로 이동한다.

- `<prefix> + x` : 현재 팬을 삭제한다.

- `<prefix> + alt + 방향키` : 현재 팬을 사이즈를 방향키로 수정한다.

- `<prefix> + spacebar` : 현재 팬을 레이아웃을 변경한다.

- `<prefix> + z` : 현재 팬을 전체화면으로 확대하거나, 전체화면으로 확대된 팬을 다시 원래 팬 레아이웃으로 축소한다.

- `tmux list-windows` : 현재 구성된 팬 레아이웃 정보를 출력한다.

## 기타 단축키 관련

- `<prefix> + ?` : 단축키 목록을 출력한다.

- `shift + <mouse drags>` : 마우스 드래그 한 내용을 호스트 시스템 클립보드에 복사한다.

- `shift + <mouse right click>` : 호스트 시스템 클립보드에 복사된 내용을 붙여넣기한다.

- `set -g <option-name> <option-value>` : 현재 옵션을 수정한다.

- `setw -g <option-name> <option-value>` : 현재 윈도우에 대한 옵션을 수정한다.

---
## Reference
[CentOS 6 / CentOS 7에서 최신 tmux를 yum으로 넣은 tmux과 공존시킨 상태에서 설치](https://www.terabo.net/blog/install-latest-tmux-on-centos-6-and-7/)  
[Install tmux on Centos release 6.5](https://gist.github.com/cdkamat/fdb136230f67c563c7b1)  
[터미널 멀티플렉서 tmux를 배워보자](http://blog.b1ue.sh/2016/10/10/tmux-tutorial/)  
[TTY 멀티플랙서 tmux](https://blog.outsider.ne.kr/699)  
[tmux 입문자 시리즈 요약](https://edykim.com/ko/post/tmux-introductory-series-summary/)  
[Everything you need to know about Tmux copy paste - Ubuntu](https://www.rushiagr.com/blog/2016/06/16/everything-you-need-to-know-about-tmux-copy-pasting-ubuntu/)  