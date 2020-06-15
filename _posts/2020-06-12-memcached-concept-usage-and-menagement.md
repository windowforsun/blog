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
	
## Memcached 명령어
- 명령어 표준 프로토콜은 `item` 에 대한 동작을 수행하고, `item` 은 아래와 같은 구성을 갖는다.
	- `key` : `ASCII` 포맷으로 최대 250 byte 길이의 문자열을 가질 수 있다.
	- `flag` : 32 bit 플래그 데이터로 조회시 함께 반환된다.
	- `expired time` : `item` 의 만료 시간(초)으로 0 값일 경우 만료되지 않고, 최대 30일 까지 설정 가능하다. `unixtimestamp` 로 처리된다.
	- `cas` : `cas` 라는 동작에 사용되는 64 bit 의 유니크한 값이다.
	- `data` : 실제로 저장하려는 데이터이다.
- `cas` 는 옵션으로 `item` 을 구성하는데 더 많은 필드를 구성하고, `-c` 를 통해 완전히 비활성화 할 수 있다.
- `Memcached` 명령어 프로토콜관련 더 자세한 내용은 [여기](https://github.com/memcached/memcached/wiki/Protocols)
에서 확인 가능하다.

### No Replay
- `Memcached` 는 명령어 수행에 대해 [Text Protocol](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)
과 [Binary Protocol](https://github.com/memcached/memcached/wiki/BinaryProtocolRevamped) 
을 제공한다.
- `Text Protocol`(ASCII) 에서 대부분의 명령어는 `no reply` 기능을 제공하지만, 
요청에 대한 오류를 대응 시킬 수 없기 때문에 권장하는 방식은 아니다. 
해당 프로토콜에서는 `set`, `add` 명령과 같은 수행에서 결과에 대한 반환 패킷을 기다리지 않는 것을 목적으로 한다.
- `Binary Protocol` 은 `no reply` 기능을 알맞게 제공하기 때문에 기능을 활용할 수 있다.

### Storage Command (저장)

Command|Description|Example
---|---|---
set|데이터를 저장하는 명령어로, 존재할 경우 덮어쓰고 새로운 데이터의 경우 `LRU` 의 최상단에 위치하게 된다.|set <key> <flags> <ttl> <size>
add|새로운 데이터를 저장하는 명령어로, 새로운 데이터는 `LRU` 의 최상단에 위치하고 이미 존재할 경우 실패한다.|add <key> <flags> <ttl> <size>
replace|기존 데이터를 교체하는 명령어이다.|replace <key> <flags> <ttl> <size>
append|기존 데이터의 마지막 바이트 이후에 데이터를 추가하는 명령어로, `item` 크기 제한을 초과 할 수 없다.|append <key> <flag> <ttl> <size>
prepend|`append` 와 수행하는 동작이 같지만, 기존 데이터의 앞쪽에 추가한다.|prepend <key> <flags> <ttl> <size>
cas|`check and set`, `compare and swap` 의 동작을 수행하는 명령어로, 기존 데이터를 마지막에 읽은 클라이언트외 다른 클라이언트는 갱신하지 못한다. `race condition` 이 발생할 수 있는 데이터에 적접한 동작이다.|하단에서 별도로 설명

- `cas` 명령어는 `cas <key> <flags> <ttl> <size> <unique_cas_key> [noreply]` 의 구성을 갖는다.
	- `unique_cas_key` : 해당하는 `key` 에 대한 데이터를 조회(`gets`) 했을 때 얻은 64 bit 고유한 키이다.
	- `noreply` : 명령어 수행 후 서버에서 응답을 받을지 여부

### Retrieval Command (조회)

Command|Description|Example
---|---|---
get|저장된 데이터를 조회하는 명령어로, 하나 이상의 키를 전달해 조회 가능하다.|get <key key ...>
gets|`cas` 와 함께 사용한 `get` 명령어로, 데이터와 `cas` 식별자(`unique_cas_key`) 를 반환한다. 이후 `cas` 명령어에서 해당 식별자를 사용 할 수 있다.|gets <key>

### Delete Command (삭제)

Command|Description|Example
---|---|---
delete|존재하는 데이터를 삭제한다.|delete <key>

### Increase/Decrease Command (증가/감소)

Command|Description|Example
---|---|---
incr|저장된 숫자 형식 값을 증가시킨다.|incr <key> <value>
decr|저장된 숫자 형식 값을 감소시킨다.|decr <key> <value>

- 저장된 데이터가 64 bit 정수의 문자열 표현인 경우 사용가능하다.
- 두 명령어 모두 양수만 값으로 지정가능하고, 음수는 불가능하다.
- 저장되지 데이터의 경우 명령어는 실패한다.

### Statistics
- 현재 상태에 대한 통계 데이터를 조회 할 수 있는 명령어이다.
- 조회된 데이터 관련 설명은 [여기](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)
에서 확인 할 수 있다.

Command|Description|Example
---|---|---
stats|기본적인 통계 데이터를 반환한다.|stats
stats items|저장된 `item` 관련 통계로, `slabs` 분류 별로 저장된 상태를 반환한다.|stats items
stats slabs|`slabs` 에 저장된 상태 통계로, 데이터의 수보다는 성능관련 데이터를 반환한다.|stats slabs
stats sizes|`slabs` 에 저장되 데이터를 `slabs` 수가 아닌 32 bit 버킷으로 분할 할 경우 항목의 분배를 보여준다. `slabs` 사이징의 효율정에 대해 파악할 수 있다.|stats sizes

- 추가적으로 `stats sizes` 는 아래와 같은 주의할 점이 있다.
	- 해당 명령어는 개발 명령어이므로, 실 서비스에 영향을 줄 수 있다.
	- 데이터가 많은 상태에서 명령어를 수행하게 되면 몇 분 동안 반응이 없을 수 있다.
	
### flush_all

Command|Description|Example
---|---|---
flush_all|즉시 모든 데이터를 삭제한다.|flush_all
flush_all|전달된 초만큼 대기 후 모든 데이터를 삭제한다.|flush_all <sec>

- `flush_all` 명령어가 수행 할때 서버는 멈추지 않는다.
- 실제로 데이터를 모두 삭제하는 방식이 아닌, 모든 데이터를 `expire` 시키는 방식으로 수행된다.
	
	
## Memcached 메모리 할당과 사용

![그림 1]({{site.baseurl}}/img/memcached/concept-usage-and-management-1.png)

- `Memcached` 는 메모리 할당을 위해서 `slabs`, `pages`, `chunks` 라는 개념을 사용한다.
- 빠른 `key-value` 저장 방식을 제공하지만, 잘못하면 메모리 효율이 안좋아지고 지속적으로 `eviction` 이 발생할 수 있다.
- 저장되는 데이터 단위인 `item` 의 기본 설정 크기는 1MB 이다.

### Slabs
- `slabs` 는 `Memcached` 의 메모리를 구성하는 단위를 의미한다.
- 저장공간의 최소 단위인 `chunks` 의 크기를 결정한다.
- 미리 연산된 고정된 특정 크기로 생성되고, 저장되는 데이터는 크기가 가장 비슷한 `slabs` 에 저장된다.
- `memory fragmentation` 을 줄이기 위해 다수개의 고정된 크기의 `slabs` 을 활용한다.
- `slabs` 의 크기는 기본 설정 일때 최소 96 byte(64 architecture) 를 시작으로 설정된 `growth_factor`(1.25) 의 곱만큼 증가한다.
- 하나의 `slabs` 은 여러개의 `pages` 를 갖는다.

### Pages
- `pages` 는 1MB(기본) 단위의 메모리 저장공간을 의미한다.
- 하나의 `pages` 는 여러개의 `chunks` 를 포함한다.
- 각 `pages` 는 1MB 크기를 `chunks` 의 크기로 나눈 수 만큼의 `chunks` 를 포함한다.

### Chunks
- `chunks` 는 `Memcached` 에서 저장되는 데이터의 단위인 `item` 하나를 저장하는 최소 단위이다.
- `slabs` 의 최소 크기가 96 byte 인것 처럼 `chunks` 의 최소 크기또한 96 byte 이다.
- 저장하는 `item` 의 크기가 60 byte 이면 가장 가까운 96 byte 의 `chunks` 에 저장되고, 나머지 36 byte 는 낭비되는 공간으로 남게 된다.
- `chunks` 는 `key` + `value` + `flag` 로 구성된다.

### 기본적인 동작
- 앞서 설명한 것과 같이 `Memcached` 는 미리 정해진 다수개의 공간인 `slabs` 를 할당 받은 상태에서 데이터 크기에 따라 적절한 공간을 선택해 저장하게 된다.
- `slabs` 는 하나 이상의 `pages` 를 관리하고 다시 `pages` 는 하나 이상의 `chunks` 를 괸라하게 되고, 
`slabs` 크기만큼 정해진 `chunks` 의 공간에 저장하는 데이터인 `item` 이 저장된다.
- 계속해서 데이터가 추가되면 하나의 `pages` 에 저장할 수 있는 `chunks` 개수가 다 차게 되면 `slabs` 는 새로운 페이지를 만들어 데이터를 저장한다.
- 위와 같이 한번 할당된 공간은 서버가 재시작하기 전까진 계속해서 유지된다.

## Memcached LRU






























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
	
- 아래는 몇가 옵션을 적용한 예이다.

	```
	PORT="11211"
    USER="memcached"
    MAXCONN="1024"
    CACHESIZE="9000"
    OPTIONS="-e /mem/file -vv 2>> /mem/memcached.log -f 1.25 -I 5m -n 200"
	```  
	
	- `-e` 옵션을 통해 `SIGUSR1` 으로 중지됐을 때 사용할 덤프파일 저장 경로 설정
	- `-vv` 옵션으로 구동 중 클라이언트로 부터 실행되는 요청 응답 로그 경로 설정
	- `-f` 옵션으로 `slabs` 크기를 증가시키는 `growth_factor` 설정
	- `-I` 옵션으로 `item` 의 최대 크기 설정
	- `-n` 옵션으로 `chunks` 의 최소 크기 설정
	
- 작성한 설정내용을 적용하고 싶다면, `Memcached` 를 재시작하면 된다.

	```bash
	$ service memcached restart
	# OR
	$ systemctl restart memcached
	```  
	
## Memcached 상태 관리
- `memcached-tool` 을 통해 `Memcached` 의 상태 정보와 사용중인 메모리관련 정보를 조회 할 수 있다.
	
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
