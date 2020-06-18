--- 
layout: single
classes: wide
title: "[Linux 실습] CLI 로 성능 분석하기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: '리눅스에서 CLI 를 통해 성능을 분석하거나 상태를 파악해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
---  

## 환경
- CentOS 7

## 개요
- 서버 성능이나 현재 상태를 파악이 필요할 때, 클라우드 서비스를 사용한다면 자체적으로 제공하는 모니터링 툴을 이용할 수 있다.
- 별도의 툴을 사용해서 성능 체크나 모니터링도 가능하고 대부분의 이슈 파악까지 해결할 수 있다.
- 특정 상황에서 직접 서버 인스턴스에 로그인해서 `CLI` 를 통해 리눅스 표준 성능 체크 툴이 필요할 경우에 대해 설명한다.

## uptime

```bash
$ uptime
 22:14:25 up 12 days, 11:51,  3 users,  load average: 8.48, 7.65, 6.37
```  

- `uptime` 은 현재 대기중인 프로세스가 얼마나 있는지를 의미하는 `load average` 를 확인하기 좋은 명령어이다.
- `load average` 는 대기 중인 프로세스, Disk I/O 와 같은 I/O 작업으로 부터 블럭된 프로세스까지 포함한다.
- 대략적으로 얼만큼의 리소스가 사용되고 있는지 확인 가능하다. 정확한 파악을 위해서는 이후에 나오는 `vmstat`, `mpstat` 등의 명령어가 필요하다.
- `8.48, 7.65, 6.37` 의 값은 순서대로 1분, 5분, 15분에 대한 `load average` 의 값이다.
- 장애 상황일 때 1분의 값이 5분이나 15분의 값보다 작다면, 확인이 너무 늦었다는 의미가 될 수 있고, 1분의 값이 다른 값들보다 현저하게 높다면 최근에 장애가 발생했다고 파악할 수 있다.
	
```bash	
$ uptime --help

Usage:
 uptime [options]

Options:
 -p, --pretty   show uptime in pretty format
 -h, --help     display this help and exit
 -s, --since    system up since
 -V, --version  output version information and exit

For more details see uptime(1).
```  

## dmesg | tail

```bash
$ dmesg | tail
[1078468.902282] device vethc09632a entered promiscuous mode
[1078468.903237] IPv6: ADDRCONF(NETDEV_UP): vethb1bb04d: link is not ready
[1078468.904539] IPv6: ADDRCONF(NETDEV_UP): vethc09632a: link is not ready
[1078468.904588] IPv6: ADDRCONF(NETDEV_CHANGE): vethc09632a: link becomes ready
[1078468.904622] docker0: port 1(vethc09632a) entered blocking state
[1078468.904626] docker0: port 1(vethc09632a) entered forwarding state
[1078468.904651] IPv6: ADDRCONF(NETDEV_CHANGE): vethb1bb04d: link becomes ready
[1078468.929065] docker0: port 1(vethc09632a) entered disabled state
[1078468.932709] docker0: port 1(vethc09632a) entered blocking state
[1078468.932712] docker0: port 1(vethc09632a) entered forwarding state
```  

- `dmesg` 는 시스템 메시지를 확인 할 수 있는 명령어이다.
- 인스턴스가 부팅한 시점으로 부터 커널 메시지를 확인 할 수 있다.
- 메시지를 통해 성능에 문제가 될 수 있는 에러를 찾아 파악할 수 있다.

```bash
$ dmesg --help

Usage:
 dmesg [options]

Options:
 -C, --clear                 clear the kernel ring buffer
 -c, --read-clear            read and clear all messages
 -D, --console-off           disable printing messages to console
 -d, --show-delta            show time delta between printed messages
 -e, --reltime               show local time and time delta in readable format
 -E, --console-on            enable printing messages to console
 -F, --file <file>           use the file instead of the kernel log buffer
 -f, --facility <list>       restrict output to defined facilities
 -H, --human                 human readable output
 -k, --kernel                display kernel messages
 -L, --color                 colorize messages
 -l, --level <list>          restrict output to defined levels
 -n, --console-level <level> set level of messages printed to console
 -P, --nopager               do not pipe output into a pager
 -r, --raw                   print the raw message buffer
 -S, --syslog                force to use syslog(2) rather than /dev/kmsg
 -s, --buffer-size <size>    buffer size to query the kernel ring buffer
 -T, --ctime                 show human readable timestamp (could be
                               inaccurate if you have used SUSPEND/RESUME)
 -t, --notime                don't print messages timestamp
 -u, --userspace             display userspace messages
 -w, --follow                wait for new messages
 -x, --decode                decode facility and level to readable string

 -h, --help     display this help and exit
 -V, --version  output version information and exit

Supported log facilities:
    kern - kernel messages
    user - random user-level messages
    mail - mail system
  daemon - system daemons
    auth - security/authorization messages
  syslog - messages generated internally by syslogd
     lpr - line printer subsystem
    news - network news subsystem

Supported log levels (priorities):
   emerg - system is unusable
   alert - action must be taken immediately
    crit - critical conditions
     err - error conditions
    warn - warning conditions
  notice - normal but significant condition
    info - informational
   debug - debug-level messages


For more details see dmesg(q).
```  

## vmstats 1

```bash
[root@localhost vagrant]# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 4721372   2068 4097580    0    0     1     7   36   26  0  0 100  0  0
 0  0      0 4720496   2068 4097624    0    0     0     0  357  458  1  1 98  0  0
 0  0      0 4720552   2068 4097624    0    0     0     0  276  460  0  1 99  0  0
 0  0      0 4720268   2068 4097624    0    0     0    39  362  590  1  1 99  0  0
 0  0      0 4720304   2068 4097624    0    0     0     0  274  498  0  0 100  0  0
```  

- `vmstat` 은 `virtual memory stat` 의 약자로 프로세스, 메모리, 페이징, I/O 블럭, CPU 사용에 대한 정보를 확인 할 수 있다.
- 인자에 `1` 을 주어 1초마다 정보를 출력하도록 한 예시이다.
- 프로세스를 나타내는 `procs` 필드에 대한 설명은 아래와 같다.
	- `r` :  CPU 에서 동작중인 프로세스의 숫자로 CPU 자원이 포화가 발생하는지 확인 할 수 있다. `r` 값이 CPU 의 값보다 크다면 포화상태에 있다고 판단할 수 있다.
	- `b` : 인터럽트 되지않는 `sleep` 프로세스의 수
- 메모리를 나타내는 `memory` 필드에 대한 설명은 아래와 같고 단위는 `KB` 이다.
	- `swpd` : 가상 메모리 사용량
	- `free` : 사용가능한 메모리 사용량으로, 더 자세한 정보는 `free -m` 을 통해 확인 할 수 있다.
	- `buff` : 버퍼 메모리 양
	- `cache` : 캐시 메모리 양
- 스왑 메모리를 나타내는 `swap` 필드에 대한 설명은 아래와 같다.
	- `si/so` : 디스크 -> 메모리 / 메모리 -> 디스트 스왑량(/s)으로 0 이 아닐 경우 시스템 메모리가 부족하다고 판단 할 수 있다.
- 입출력 I/O 관련 필드인 `bi/bo` 는 장치에서 받아오는 블록, 장치로 보내는 블록(block/s)
- 시스템 관련 필드인 `system` 에서 `in` 은 초당 인터럽트 수, `cs` 는 초당 문맥교환 수 이다.
- CPU 사용관련 필드인 `cpu` 는 CPU time 을 측정 할 수 있고, 관련 설명은 아래와 같다.
	- `us` : 커널이 아닌 코드 사용 시간 (사용자 시간)
	- `sy` : 커널 코드 사용 시간 (시스템 시간)
	- `id` : 유휴 시간
	- `wa` : 입출력 대기 시간
	- `st` : 가상머신으로 부터 뺏은 시간
	
```bash
$ vmstat --help

Usage:
 vmstat [options] [delay [count]]

Options:
 -a, --active           active/inactive memory
 -f, --forks            number of forks since boot
 -m, --slabs            slabinfo
 -n, --one-header       do not redisplay header
 -s, --stats            event counter statistics
 -d, --disk             disk statistics
 -D, --disk-sum         summarize disk statistics
 -p, --partition <dev>  partition specific statistics
 -S, --unit <char>      define display unit
 -w, --wide             wide output
 -t, --timestamp        show timestamp

 -h, --help     display this help and exit
 -V, --version  output version information and exit

For more details see vmstat(8).
```  
	
## mpstat -p ALL 1

```bash
$ mpstat -P ALL 1
Linux 3.10.0-957.27.2.el7.x86_64 (localhost.localdomain)        06/18/2020      _x86_64_        (2 CPU)

12:20:24 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
12:20:25 PM  all    0.00    0.00    1.01    0.00    0.00    0.00    0.00    0.00    0.00   98.99
12:20:25 PM    0    0.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00   99.00
12:20:25 PM    1    0.00    0.00    1.01    0.00    0.00    0.00    0.00    0.00    0.00   98.99
```  

- `mpstat` 은 CPU time 을 CPU 별로 측정 가능한 명령어이다.
- 설치 돼있지 않다면 `yum install sysstat` 을 통해 설치 가능하다.
- CPU 별로 불균형한 상태를 확인 할 수 있다.

```bash
[root@localhost docker3]# mpstat --help
Usage: mpstat [ options ] [ <interval> [ <count> ] ]
Options are:
[ -A ] [ -u ] [ -V ] [ -I { SUM | CPU | SCPU | ALL } ]
[ -P { <cpu> [,...] | ON | ALL } ]
```  

## pidstat 1

```bash
$ pidstat 1
Linux 3.10.0-957.27.2.el7.x86_64 (localhost.localdomain)        06/18/2020      _x86_64_        (2 CPU)

12:23:03 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
12:23:04 PM     0      1293    0.00    0.99    0.00    0.99     1  dockerd
12:23:04 PM   999      5447    0.00    0.99    0.00    0.99     0  mysqld
12:23:04 PM     0     24940    0.00    0.99    0.00    0.99     1  pidstat

12:23:04 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
12:23:05 PM     0      4519    1.00    0.00    0.00    1.00     1  containerd-shim

12:23:05 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
12:23:06 PM   999      5447    0.00    1.00    0.00    1.00     0  mysqld
12:23:06 PM     0     24940    0.00    1.00    0.00    1.00     1  pidstat
```  

- `pidstat` 은 프로세스당 `top` 명령어를 수행한 것과 비슷한 결과를 보여주는 명령어이다.
- `CPU` 필드의 값이 100 이면 1개 코어를 모두 사용 중인 것으로 파악할 수 있고, 1200 이라면 12 코어를 사용중이라고 판단할 수 있다.

## iostat -xz 1

```bash
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          63.82    0.00   34.67    0.00    0.00    1.51

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00   18.00     0.00   735.00    81.67     0.01    0.78    0.00    0.78   0.28   0.50
dm-1              0.00     0.00    1.00   12.00    64.00   104.00    25.85     0.00    0.23    0.00    0.25   0.23   0.30
dm-3              0.00     0.00    0.00   10.00     0.00    40.00     8.00     0.01    0.50    0.00    0.50   0.50   0.50
```  

- `iostat` 은 HDD, SSD 와 같은 `block device` 의 상태를 확인 할 수 있는 명령어이다.
- `r/s`, `w/s`, `wkB/s` : 순서대로 초당 읽기, 초당 쓰기, 초당 쓴 KB 크기, 초당 읽은 KB 크기를 의미한다. 과도한 쓰기와 과도한 읽기에 대해 파악할 수 있다.
- `await` : I/O 처리의 평균 시간을 밀리초로 표현한 값으로, 
구동되는 프로세스 입장에서는 I/O 요청이 `queue` 되고 실제로 수행되기 까지 걸리는 시간이게 되고 그 동안 대기하게 된다. 
장치의 요청 처리 시간보다 긴 경우 장치 자체의 문자가 있거나 포화된 상태라고 판단할 수 있다.
- `avgqu-sz` : 장치에 실행된 평균 요청 수로, 값이 1보다 크다면 포화 상태라고 판단 할 수 있다.
- `%util` : 장치의 사용률 나타내는 값으로, 60% 보다 크다면 성능 저하가 발생할 수 있고 상황마다 다를 수 있지만 100% 값이라면 포화 상태라고 판단할 수 있다.

```bash
$ iostat --help
Usage: iostat [ options ] [ <interval> [ <count> ] ]
Options are:
[ -c ] [ -d ] [ -h ] [ -k | -m ] [ -N ] [ -t ] [ -V ] [ -x ] [ -y ] [ -z ]
[ -j { ID | LABEL | PATH | UUID | ... } ]
[ [ -T ] -g <group_name> ] [ -p [ <device> [,...] | ALL ] ]
[ <device> [...] | ALL ]
```  

## free -m

```bash
$ free -m
              total        used        free      shared  buff/cache   available
Mem:           9623        1010        4600          16        4012        8102
Swap:          2047           0        2047
```  

- `free` 는 메모리에 대해서 사용, 여유에 대한 크기를  KB 단위로 보여주는 명령어이다.
- `-m` 은 MB, `-g` 는 GB 단위로 조회도 가능하다.
- 각 필드에 대한 설명은 아래와 같다.
	- `total` : 전체 메모리
	- `used` : 사용 메모리
	- `free` : 여유 메모리
	- `shared` : 공유 메모리로 프로세스, 스레드간 통신에서 사용됨
	- `buffers` : 버퍼 메모리, I/O 버퍼, 파일 등의 입출력에 사용됨
	- `cached` : 캐시 메모리, 재사용을 위해 메모리 내용을 임시 보존
- `buffers`, `cached` 값이 0에 가까워 진다는 것은 높은 Disk I/O 가 발생하고 있음을 의미하기 때문에 위험할 수 있다.

```bash
$ free --help

Usage:
 free [options]

Options:
 -b, --bytes         show output in bytes
 -k, --kilo          show output in kilobytes
 -m, --mega          show output in megabytes
 -g, --giga          show output in gigabytes
     --tera          show output in terabytes
     --peta          show output in petabytes
 -h, --human         show human-readable output
     --si            use powers of 1000 not 1024
 -l, --lohi          show detailed low and high memory statistics
 -t, --total         show total for RAM + swap
 -s N, --seconds N   repeat printing every N seconds
 -c N, --count N     repeat printing N times, then exit
 -w, --wide          wide output

     --help     display this help and exit
 -V, --version  output version information and exit

For more details see free(1).
```  

## sar -n DEV 1

```bash
$ sar -n DEV 1
Linux 3.10.0-1062.18.1.el7.x86_64 (cpb2015-qa-load-01.com2us.kr)        06/18/2020      _x86_64_        (2 CPU)

03:20:00 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
03:20:01 PM veth85c98df     67.00     67.00      7.39      8.25      0.00      0.00      0.00
03:20:01 PM      eth0     66.00     68.00      8.21     10.03      0.00      0.00      0.00
03:20:01 PM      eth1      1.00      0.00      0.06      0.00      0.00      0.00      0.00
03:20:01 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
03:20:01 PM   docker0     67.00     67.00      6.47      8.25      0.00      0.00      0.00
```  

- `sar` 은 `network throughtput` 인 `Rx` 와 `Tx` 를 `KB/s` 로 측정한 결과를 확인 할 수 있다.

```bash
$ sar --help
Usage: sar [ options ] [ <interval> [ <count> ] ]
Options are:
[ -A ] [ -B ] [ -b ] [ -C ] [ -d ] [ -F [ MOUNT ] ] [ -H ] [ -h ] [ -p ] [ -q ] [ -R ]
[ -r ] [ -S ] [ -t ] [ -u [ ALL ] ] [ -V ] [ -v ] [ -W ] [ -w ] [ -y ]
[ -I { <int> [,...] | SUM | ALL | XALL } ] [ -P { <cpu> [,...] | ALL } ]
[ -m { <keyword> [,...] | ALL } ] [ -n { <keyword> [,...] | ALL } ]
[ -j { ID | LABEL | PATH | UUID | ... } ]
[ -f [ <filename> ] | -o [ <filename> ] | -[0-9]+ ]
[ -i <interval> ] [ -s [ <hh:mm:ss> ] ] [ -e [ <hh:mm:ss> ] ]
```  
## sar -n TCP,ETCP 1

```bash
$ sar -n TCP,ETCP 1
Linux 3.10.0-1062.18.1.el7.x86_64 (cpb2015-qa-load-01.com2us.kr)        06/18/2020      _x86_64_        (2 CPU)

03:26:54 PM  active/s passive/s    iseg/s    oseg/s
03:26:55 PM      0.00      0.00      1.00      1.00

03:26:54 PM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
03:26:55 PM      0.00      0.00      0.00      0.00      0.00

03:26:55 PM  active/s passive/s    iseg/s    oseg/s
03:26:56 PM      0.00      0.00      2.00      2.00

03:26:55 PM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
03:26:56 PM      0.00      0.00      0.00      0.00      0.00

03:26:56 PM  active/s passive/s    iseg/s    oseg/s
03:26:57 PM      0.00      1.00      6.00      6.00

03:26:56 PM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
03:26:57 PM      0.00      0.00      0.00      0.00      0.00

03:26:57 PM  active/s passive/s    iseg/s    oseg/s
03:26:58 PM      0.00      1.00      6.00      6.00
```  

- 위 명령어를 통해 TCP 통신량에 대해 파악이 가능하다.
- `active/s` : 로컬에서부터 요청한 초당 TCP 커넥션 수 ex) `connection()` 을 통한 연결
- `passive/s` : 원격으로부터 요청된 초당 TCP 커넥션 수 ex) `accept()` 을 통한 연결
- `retrans/s` : 초당 TCP 재연결 수
- `active`, `passive` 수를 통해 서버의 부하를 대략적으로 측정 가능하다. 
- `retrans` 의 수가 많다면 네트워크나 서버의 이슈가 있다는것으로 판단할 수 있다. 좋지않은 네트워크 환경이나, 처리 이상의 커넥션으로 패킷 드랍 현상이 발생하는 경우가 될 수 있다.

## top

```bash
$ top
top - 15:34:05 up 6 days,  1:54,  1 user,  load average: 0.00, 0.02, 0.08
Tasks: 117 total,   1 running, 116 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.2 us,  0.2 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  9854472 total,  4732900 free,  1030696 used,  4090876 buff/cache
KiB Swap:  2097148 total,  2097148 free,        0 used.  8344504 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9109 vagrant   20   0  154496  34840   1584 S   0.7  0.4   3:23.94 tmux: server
 1293 root      20   0  845632 156112  34108 S   0.3  1.6  42:57.70 dockerd
 5447 polkitd   20   0 1851816 431724  17536 S   0.3  4.4  61:24.20 mysqld
    1 root      20   0  128184   6792   4176 S   0.0  0.1   0:10.35 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.08 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   0:05.68 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    7 root      rt   0       0      0      0 S   0.0  0.0   0:01.06 migration/0
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
    9 root      20   0       0      0      0 S   0.0  0.0   0:40.49 rcu_sched
   10 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 lru-add-drain
   11 root      rt   0       0      0      0 S   0.0  0.0   0:04.42 watchdog/0
   12 root      rt   0       0      0      0 S   0.0  0.0   0:03.80 watchdog/1
   13 root      rt   0       0      0      0 S   0.0  0.0   0:01.03 migration/1
   14 root      20   0       0      0      0 S   0.0  0.0   0:15.97 ksoftirqd/1
   16 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/1:0H
   18 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kdevtmpfs
   19 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 netns
```  

- `top` 명령어는 지금까지 언급되었던 부분들에 대해 시스템 전반적인 값을 확인할 수 있는 명령어이다.
- 계속해서 화면의 값이 변경되기 때문에 `ctrl+s` 는 일지정지, `ctrl+q` 는 다시시작 단축키를 활용하면 더욱 편리하다.

	
---
## Reference
[Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)  