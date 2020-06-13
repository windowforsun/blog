--- 
layout: single
classes: wide
title: "Memcached 사용과 관리"
header:
  overlay_image: /img/memcached-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Memcached
tags:
    - Concept
toc: true
use_math: true
---  

## Memcached
- `Memcached` 에 대한 기본 설정은 [여기]({{site.baseurl}}{% link _posts/2019-03-25-memcached-intro.md %})
에 기술 돼있다.

## Memcached 설정
- `Memcached` 구동시 커스텀한 설정을 적용할 때는 아래 경로에 내용을 작성해주면 된다.

	```bash
	$ vi /etc/sysconfig/memcached
	```  
	
- `/etc/sysconfig/memcached` 의 기본 설정은 아래와 같다.

	```
	PORT="11211"  
    USER="memcached"
    MAXCONN="1024"
    CACHESIZE="64"
    OPTIONS=""
	```  
	
	- `PORT` : 사용할 포트를 설정한다.
	- `USER` : 데몬을 시작하고 사용할 계정을 설정한다.
	- `MAXCONN` : 허용할 커넥션의 최대 갯수를 설정한다.
	- `CACHESIZE` : 사용할 메모리 크기를 설정한다.
	- `OPTIONS` : 그 외 옵션으로 설정할 내용을 입력한다.
	
- `OPTIONS` 에 설정 가능한 항목은 아래와 같다.

	```
	-p, --port=<num>          TCP port to listen on (default: 11211)
    -U, --udp-port=<num>      UDP port to listen on (default: 0, off)
    -s, --unix-socket=<file>  UNIX socket to listen on (disables network support)
    -a, --unix-mask=<mask>    access mask for UNIX socket, in octal (default: 700)
    -A, --enable-shutdown     enable ascii "shutdown" command
    -l, --listen=<addr>       interface to listen on (default: INADDR_ANY)
    -d, --daemon              run as a daemon
    -r, --enable-coredumps    maximize core file limit
    -u, --user=<user>         assume identity of <username> (only when run as root)
    -m, --memory-limit=<num>  item memory in megabytes (default: 64)
    -M, --disable-evictions   return error on memory exhausted instead of evicting
    -c, --conn-limit=<num>    max simultaneous connections (default: 1024)
    -k, --lock-memory         lock down all paged memory
    -v, --verbose             verbose (print errors/warnings while in event loop)
    -vv                       very verbose (also print client commands/responses)
    -vvv                      extremely verbose (internal state transitions)
    -h, --help                print this help and exit
    -i, --license             print memcached and libevent license
    -V, --version             print version and exit
    -P, --pidfile=<file>      save PID in <file>, only used with -d option
    -f, --slab-growth-factor=<num> chunk size growth factor (default: 1.25)
    -n, --slab-min-size=<bytes> min space used for key+value+flags (default: 48)
    -L, --enable-largepages  try to use large memory pages (if available)
    -D <char>     Use <char> as the delimiter between key prefixes and IDs.
                  This is used for per-prefix stats reporting. The default is
                  ":" (colon). If this option is specified, stats collection
                  is turned on automatically; if not, then it may be turned on
                  by sending the "stats detail on" command to the server.
    -t, --threads=<num>       number of threads to use (default: 4)
    -R, --max-reqs-per-event  maximum number of requests per event, limits the
                              requests processed per connection to prevent
                              starvation (default: 20)
    -C, --disable-cas         disable use of CAS
    -b, --listen-backlog=<num> set the backlog queue limit (default: 1024)
    -B, --protocol=<name>     protocol - one of ascii, binary, or auto (default: auto-negotiate)
    -I, --max-item-size=<num> adjusts max item size
                              (default: 1m, min: 1k, max: 1024m)
    -S, --enable-sasl         turn on Sasl authentication
    -F, --disable-flush-all   disable flush_all command
    -X, --disable-dumping     disable stats cachedump and lru_crawler metadump
    -W  --disable-watch       disable watch commands (live logging)
    -Y, --auth-file=<file>    (EXPERIMENTAL) enable ASCII protocol authentication. format:
                              user:pass\nuser2:pass2\n
    -e, --memory-file=<file>  (EXPERIMENTAL) mmap a file for item memory.
                              use only in ram disks or persistent memory mounts!
                              enables restartable cache (stop with SIGUSR1)
	```  
	
- 작성한 설정내용을 적용하고 싶다면, `Memcached` 를 재시작하면 된다.

	```bash
	$ service memcached restart
	# OR
	$ systemctl restart memcached
	```  
	
## Memcached 상태 관리
- `memcached-tool` 을 통해 사용중인 메모리와 관련 상태 정보를 조회 할 수 있다.
	
	```bash
	$ memcached-tool
	Usage: memcached-tool <host[:port] | /path/to/socket> [mode]
	
	       memcached-tool 10.0.0.5:11211 display        # shows slabs
	       memcached-tool 10.0.0.5:11211                # same.  (default is display)
	       memcached-tool 10.0.0.5:11211 stats          # shows general stats
	       memcached-tool 10.0.0.5:11211 settings       # shows settings stats
	       memcached-tool 10.0.0.5:11211 sizes          # shows sizes stats
	       memcached-tool 10.0.0.5:11211 dump [limit]   # dumps keys and values
	
	WARNING! sizes is a development command.
	As of 1.4 it is still the only command which will lock your memcached instance for some time.
	If you have many millions of stored items, it can become unresponsive for several minutes.
	Run this at your own risk. It is roadmapped to either make this feature optional
	or at least speed it up.
	```  
	
- 현재 `slabs` 의 상태를 확인하면 아래와 같다.

	```bash
	$ memcached-tool 127.0.0.1:11211 display
      #  Item_Size  Max_age   Pages   Count   Full?  Evicted Evict_Time OOM
      1      96B         0s       1       0      no        0        0    0
      2     120B         0s       1       0      no        0        0    0
      3     152B         0s       1       0      no        0        0    0
      4     192B         0s       1       0      no        0        0    0
      
      .. 생략 ..
      
     37   308.5K         0s       1       0      no        0        0    0
     38   385.6K         0s       1       0      no        0        0    0
     39   512.0K         0s       1       0      no        0        0    0
	```  
	
- `Memcached` 의 상태 정보를 확인하면 아래와 같다.

	```bash
	$ memcached-tool 127.0.0.1:11211 stats
    #127.0.0.1:11211        Field         Value
                  accepting_conns             1
                        auth_cmds             0
                      auth_errors             0
                            bytes             0
                       bytes_read           137
                    bytes_written         66990
                       cas_badval             0
                                              
                       .. 생략 ..
                       
            slab_reassign_rescues             0
            slab_reassign_running             0
                      slabs_moved             0
                          threads             4
                             time    1592071086
       time_in_listen_disabled_us             0
                total_connections             8
                      total_items             0
                       touch_hits             0
                     touch_misses             0
                           uptime           568
                          version         1.6.5
	```  
	
- `Memcached` 의 설정 정보를 확인하면 아래와 같다.

	```bash
	$ memcached-tool 127.0.0.1:11211 settings
    #127.0.0.1:11211   Field       Value
          auth_enabled_ascii          no
           auth_enabled_sasl          no
            binding_protocol auto-negotiate
                 cas_enabled         yes
                  chunk_size          48
              detail_enabled          no
              
              .. 생략 ..
              
               temporary_ttl          61
                 track_sizes          no
                     udpport           0
                       umask         700
                   verbosity           2
                warm_lru_pct          40
             warm_max_factor        2.00
         watcher_logbuf_size      262144
          worker_logbuf_size       65536
	```  
	
## Memcached 메모리 할당과 사용

## Memcached LRU






---
## Reference
[Memcached wiki](https://github.com/memcached/memcached/wiki)  
[How to install and configure memcached](https://access.redhat.com/solutions/1160613)  
[Memcached Protocol](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)  
[Memcached LRU](https://memcached.org/blog/modern-lru/)  
[Memcached Memory Allocation](https://dzone.com/articles/memcached-memory-allocation)  
[memcached Cheat Sheet](https://lzone.de/cheat-sheet/memcached)  
[Memcached and unexpected evictions](https://medium.com/@ivaramme/memcached-and-unexpected-evictions-a3a50a239108)  
[Slabs, Pages, Chunks and Memcached](https://www.mikeperham.com/2009/06/22/slabs-pages-chunks-and-memcached/)  
[Journey to the centre of memcached](https://medium.com/@SkyscannerEng/journey-to-the-centre-of-memcached-b239076e678a)  
[Memcached의 확장성 개선](https://d2.naver.com/helloworld/151047)  
[Memcached Option](http://manpages.ubuntu.com/manpages/trusty/man1/memcached.1.html)  
[[입 개발] memcached slabclass 에 대해서…](https://charsyam.wordpress.com/2014/02/16/%EC%9E%85-%EA%B0%9C%EB%B0%9C-memcached-slabclass-%EC%97%90-%EB%8C%80%ED%95%B4%EC%84%9C/)  
