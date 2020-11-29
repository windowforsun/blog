--- 
layout: single
classes: wide
title: "[Redis 실습] Redis Persistence"
header:
  overlay_image: /img/redis-bg.png
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Practice
  - Persistence
---  

## Redis Persistence
`Redis` 에서 지속성(`Persistence`) 를 위한 기능으로는 `AOF`, `RDB` 방식이 있다. 
먼저 `AOF` 는 `Append Only File` 의 약자로 명령이 실행 될때 마다 파일에 기록하는 방식을 의미하고, 
데이터 손실은 거의 발생하지 않는다. 
`RDB` 는 설정된 특정 간격마다 메모리에 있는 `Redis` 데이터 전체를 스냅샷 하는 것과 같이 디스크에 쓰는 것으로, 
특정 시점 마다 백업 용도로 사용할 수 있다.  

## AOF
### 파일명
`AOF` 를 설정하게 되면 기본 값으로 `appendonly.aof` 라는 파일이 생성되고, 
해당 파일에 입력, 수정, 삭제 명령이 실행될대 마다 기록된다. 
조회 관련 명령은 파일에 기록 되지 않는다.  

### rewrite
`AOF` 방식을 살펴보면 장기간 사용된 `Redis` 서버의 경우 `appendonly.aof` 파일 크기가 방대하게 커질 수 있는 구조임을 알 수 있다. 
키에 대해 값을 증가하는 `INCR` 명령어를 1만번 수행하게되면 `appendonly.aof` 파일에도 명령어 수행 기록이 1만번 작성된다. 
결과적으로 `INCR` 를 수행한 키의 값은 1만이지만 명령어를 수행한 만큼 파일에 작성해야 하기때문에 파일 크기가 쉽게 커질 수 있다.  

`appendonly.aof` 파일 크기가 커지면 `OS` 파일 사이즈 제한이 걸려 이후 기록이 되지 않거나, 
`Redis` 서버 시작 시 `appendonly.aof` 파일 로드 시간에 큰 영향을 줄 수 있다.  

위와 같은 상황을 방지하기 위해 있는 기능이 `rewrite` 기능이다. 
`rewrite` 는 설장된 파일 크기와 비율 간격마다 기존 기록을 지우고 최종적으로 메모리에 있는 데이터를 기록한다. 
그리고 이후 명령은 다시 명령어 단위로 기록하게 된다.  

`rewrite` 동작 순서는 아래와 같다. 
1. `rewrite` 를 수행할 자식 프로세스를 `fork()` 한다. 
1. 자식 프로세스는 메모리 데이터를 새로운 `AOF temp` 파일에 작성한다. (1차 쓰기)
1. 위 과정 수행 중에 수행된 명령어는 부모 프로세스가 메모리 버퍼에 기록하고 기존 `AOF` 파일에도 작성한다. 
이러한 방법으로 `rewrite` 작업이 실패 하더라도 `AOF` 파일의 데이터가 보존 될 수 있도록 한다. 
1. 자식 프로세스에서 `AOF tmp` 파일 작성이 왼료되면 부모 프로세스로 `stop` 시그널을 보낸다. 
1. 부모 프로세스는 `stop` 시그널을 받으면 자식 프로세스가 `AOF temp` 파일을 작성하는 동안 수행한 명령어 데이터를 자식 프로세스에게 보낸다. 
자식 프로세스는 이 데이터를 다시 `AOF temp` 파일에 쓰고 완료되면 부모 프로세스에게 완료 시그널을 보낸다. (2차 쓰기)
1. 부모 프로세스는 `AOF temp` 파일을 열어서 2차 쓰기 동안 발생한 데이터를 다시 쓴다. (3차 쓰기)
1. 3차 쓰기까지 완료되면 기존 `AOF` 파일을 `AOF temp` 파일로 교체 하고, 계속해서 파일 쓰기를 수행한다. 

### 파일 수정
`appendonly.aof` 파일은 단순 테스트 파일이다. 
만약 실수로 수행한 명령이 있다면 `rewrite` 전에 파일을 열어 명령어를 지우는 방식으로 롤백할 수 있다. 
간단한 예로 `flushall` 을 잘못 수행한 경우, 잠시 `Redis` 서버를 `shutdown` 한 상태에서 `appendonly.aof` 파일에 아래 문구를 찾아 지워준다. 

```
*1
$8
flushall
```  

그리고 다시 `Redis` 서버를 재시작하면 `flushall` 명령은 수행되지 않았고 모든 데이터는 존재하는 상태가 된다. 

### AOF 관련 설정
- `appendonly` : 기본 값은 `no` 이고, `yes` 로 설정하면 `AOF` 기능이 활성화 된다. 
`appendonly.aof` 파일은 `appendonly yes` 인 상태에서 `Redis` 서버 재시작 시에만 파일을 읽고, 
`appendonly no` 인경우 `appendonly.aof` 파일이 있더라도 파일을 읽지 않는다. 
- `appendfilename` : 기본 값은 `"appendonly.aof"` 이고, 필요에 따라 사용할 파일 이름을 지정할 수 있다. 
경로는 별도로 지정 할 수 없고 `working directory` 를 따른다. 
- `appendfsync` : `appendonly.aof` 파일에 명령을 기록하는 시점을 설정한다. 
    - `always` : 명령어가 수행 될때 마다 `AOF` 파일에 기록하는 방법으로 데이터 유실은 발생하지 않지만, 
    `Redis` 서버 성능에 영향을 미칠 수 있다. 
    - `everysec` : 1초 마다 `AOF` 파일에 기록하는 방법으로 1초 간격에 대한 데이터 유실은 발생할 수 있지만, 
    `Redis` 서버 성능에 거의 영향을 미치지 않는다. 기본값인 만큼 권장하는 설정 값이다. 
    - `no` : `AOF` 파일 기록 시점을 `OS` 가 정하는 방법으로 일반적인 리눅스 디스크 기록 시간은 30초이다. 
    `OS` 설정에 따라 데이터 유실이 발생할 수 있다. 
- `rewrite` 관련 설정
    - `auto-aof-rewrite-percentage` : 기본 값은 `100` 으로 `AOF` 파일 크기가 `100%` 이상 커지면 `rewrite` 를 수행한다. 
    `AOF` 파일 크기 기준은 `Redis` 서버가 시작된 시점의 `AOF` 파일 크기를 기준으로 한다. 
    이후 `rewrite` 가 수행한 파일 크기가 다음 `rewrite` 를 수행하는 파일 크기의 기준이 된다. 
    만약 `0` 으로 설정할 경우 `rewrite` 는 일어나지 않는다. 
    - `auto-aof-rewrite-min-size` : 기본 값은 `64mb` 이고, 해당 파일 크기 보다 작은 경우 `rewrite` 를 수행하지 않는다. 
    해당 설정 값을 통해 초기에 `rewrite` 가 빈번하게 수행하는 것을 방지할 수 있다. 


### AOF 테스트
테스트는 `Docker` 기반으로 진행한다. 
먼저 아래 `docker run` 명령으로 `AOF` 를 활성화하고, `AOF` 파일 `rewrite` 크기가 `10b` 인 옵션으로 `Redis Server` 컨테이너를 실행한다. 
`Docker` 컨테이너 실행 명령어 마지막에 `Redis` 서버 관련 명령을 기입해주면 별도의 설정 파일없이 원하는 설정으로 실행 할 수 있다. 
그리고 `docker exec` 명령으로 컨테이너 `Bash` 에 접속한다. 
`Redis` 서버 접속은 `redis-cli` 명령을 사용해서 가능하고 포트를 별도로 설정한경우 `-p <포트명>` 을 사용해서 접속할 수 있다.  

```bash
$ docker run -d --rm -v ${PWD}/data:/data --name redis-aof redis:6 redis-server --appendonly yes --auto-aof-rewrite-min-size 10b
d126ab607dacd2d9861a02adb585f996210a42cca81a5d4e8bb94054508fa8de
$ docker exec -it redis-aof /bin/bash
root@d126ab607dac:/data# redis-cli
127.0.0.1:6379> info persistence
# Persistence
loading:0

.. 생략 ..

aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_last_cow_size:0
module_fork_in_progress:0
module_fork_last_cow_size:0
aof_current_size:0
aof_base_size:0
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0
```  

`info persistence` 명령으로 설정 정보를 조회하면 `AOF` 가 활성화 된것을 확인할 수 있다. 
`Redis` 서버에서 데이터를 추가하고 `AOF` 파일을 확인하면 아래와 같다. 

```bash
.. Redis Server ..
127.0.0.1:6379> set m_key m_value
OK

.. Bash ..
root@d126ab607dac:/data# cat appendonly.aof
set
$5
m_key
$7
m_value
```  

위와 같이 `SET` 명령어를 사용해서 `m_key` 라는 키에 `m_value` 를 저장하면 명령어 수행에 대한 내용이 `AOF 파일에 저장된다. 
그리고 `rewrite` 를 수행하면 아래와 같이 현재 메모리에 있는 데이터 전체가 저장된다. 

```bash
.. Redis Server ..
127.0.0.1:6379> bgrewriteaof
Background append only file rewriting started

.. Bash ..
root@d126ab607dac:/data# cat appendonly.aof
REDIS0009�      redis-ver6.0.9�
�edis-bits�@�ctime�-G�_used-mem¨0
 aof-preamble���������m_keym_value�x�sN�֔
```  

그리고 `rewrite` 된 이후 다음 `rewrite` 까지는 계속 동일하게 명령어 수행에 대한 내용이 작성된다.  

다음으로 `AOF` 파일을 삭제하고 `Redis` 컨테이너를 재시작 하는 방법으로 `Redis` 를 초기화 한다. 
초기 상태에서 다시 `info persistence` 명령으로 설정 정보를 출력하고 필요한 정보만 추리면 아래와 같다. 

```bash
127.0.0.1:6379> info persistence
# Persistence
aof_enabled:1  # AOF 기능 활성화 여부
aof_rewrite_in_progress:0  # rewrite 가 현재 실행 중인지 
aof_last_rewrite_time_sec:-1  # 마지막 rewrite 를 수행할때 소요된 시간
aof_current_rewrite_time_sec:-1  # rewrite 를 시작하고 지금까지 경과한 시간
aof_last_bgrewrite_status:ok  # 마지막 bgrewrite 상태
aof_last_write_status:ok  # 마지막 rewrite 상태
aof_current_size:0  # 현재 AOF 파일 크기
aof_base_size:0  # 기본 AOF 파일 크기로, current_size 와 base_size 비교해서 rewrite-percentage 이상이면 rewrite 를 수행한다. 
```  

데이터 하나를 추가하고 다시 `info persistence` 를 호출해서 살펴보면 아래와 같다. 

```bash
127.0.0.1:6379> set 1 1
OK
127.0.0.1:6379> info persistence
# Persistence
aof_enabled:1
aof_rewrite_in_progress:0
aof_last_rewrite_time_sec:1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:50
aof_base_size:50
```  

현재 `rewrite-min-size` 가 `10b` 로 설정돼있기 때문에, 
데이터 하나만 추가한 상태인데 `aof_last_rewrite_time_sec` 필드가 `1` 로 출력되는 것으로 보아 `rewrite` 가 수행된 상태인것으로 보인다. 
`rewrite` 수행은 `base_size` 를 기준으로 `current_size` 가 `rewrite-percentage`(`100%`) 비율 이상인 상태여야 한다고 했었다.  

```bash
127.0.0.1:6379> set 2 2
OK
127.0.0.1:6379> info persistence
# Persistence
aof_last_rewrite_time_sec:0
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:100
aof_base_size:100  # rewrite 수행됨
127.0.0.1:6379> set 3 3
OK
127.0.0.1:6379> info persistence
# Persistence
aof_last_rewrite_time_sec:0
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:150
aof_base_size:100  
127.0.0.1:6379> set 4 4
OK
127.0.0.1:6379> info persistence
# Persistence
aof_last_rewrite_time_sec:0
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:177
aof_base_size:100
127.0.0.1:6379> set 5 5
OK
127.0.0.1:6379> info persistence
# Persistence
aof_last_rewrite_time_sec:0
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:204
aof_base_size:204  # rewrite 수행 됨
```  

명령 출력 결과를 살펴보면 `base_size` 는 `50 -> 100 -> 204` 순으로 증가 했다. 
`rewrite` 가 실제로 수행되면서 발생한 로그를 확인하면 아래와 같다. 

```
# rewrite 수행 조건 만족
1:M 26 Nov 2020 12:24:53.489 * Starting automatic rewriting of AOF on 100% growth
# 자식 프로세스 실행
1:M 26 Nov 2020 12:24:53.491 * Background append only file rewriting started by pid 28
# 1차 쓰기 완료
1:M 26 Nov 2020 12:24:53.523 * AOF rewrite child asks to stop sending diffs.
28:C 26 Nov 2020 12:24:53.523 * Parent agreed to stop sending diffs. Finalizing AOF...
# 1차 쓰기 중 발생한 데이터를 다시 쓰는 2차 쓰기 수행
28:C 26 Nov 2020 12:24:53.523 * Concatenating 0.00 MB of AOF diff received from parent.
28:C 26 Nov 2020 12:24:53.528 * SYNC append only file rewrite performed
28:C 26 Nov 2020 12:24:53.528 * AOF rewrite: 12 MB of memory used by copy-on-write
1:M 26 Nov 2020 12:24:53.591 * Background AOF rewrite terminated with success
# 3차 쓰기 완료
1:M 26 Nov 2020 12:24:53.593 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
1:M 26 Nov 2020 12:24:53.596 # Unable to obtain the AOF file length. stat: No such file or directory
1:M 26 Nov 2020 12:24:53.596 * Background AOF rewrite finished successfully
```  

다음 `rewrite` 는 `current_size` 가 `408` 이상일 때 수행 될 것이다. 
아직 `rewrite` 조건을 충족하지 못했지만 `AOF` 파일에 현재 데이터를 쓰고 싶을 경우나, 
현재 `AOF` 파일이 너무 커진 경우에는 `BGREWRITEAOF` 명령을 사용해서 명시적으로 수행할 수 있다. 

```bash
127.0.0.1:6379> set 5 5
OK
127.0.0.1:6379> info persistence  # rewrite 조건을 만족하지 않기 때문에 current_size 외 다른 부분이 없다. 
# Persistence
aof_enabled:1
aof_last_rewrite_time_sec:0
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:254
aof_base_size:204
127.0.0.1:6379> bgrewriteaof
Background append only file rewriting started
127.0.0.1:6379> info persistence # rewrite 가 수행되었기 때문에 base_size 와 current_size 가 동일하다. 
# Persistence
aof_enabled:1
aof_last_rewrite_time_sec:0
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_current_size:254
aof_base_size:254
```  

`BGREWRITEAOF` 을 통해 명시적으로 실행한 `rewrite` 의 로그는 아래와 같다. 

```
# BGREWRITEAOF 명령을 통해 rewrite 시작
1:M 26 Nov 2020 12:46:20.887 * Background append only file rewriting started by pid 30
1:M 26 Nov 2020 12:46:20.921 * AOF rewrite child asks to stop sending diffs.
30:C 26 Nov 2020 12:46:20.921 * Parent agreed to stop sending diffs. Finalizing AOF...
30:C 26 Nov 2020 12:46:20.921 * Concatenating 0.00 MB of AOF diff received from parent.
30:C 26 Nov 2020 12:46:20.926 * SYNC append only file rewrite performed
30:C 26 Nov 2020 12:46:20.927 * AOF rewrite: 12 MB of memory used by copy-on-write
1:M 26 Nov 2020 12:46:20.965 * Background AOF rewrite terminated with success
1:M 26 Nov 2020 12:46:20.967 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
1:M 26 Nov 2020 12:46:20.970 # Unable to obtain the AOF file length. stat: No such file or directory
1:M 26 Nov 2020 12:46:20.970 * Background AOF rewrite finished successfully
```  

## RDB
`RDB` 는 특정 시점의 메모리 데이터 전체를 바이너리로 파일에 저장하는 것을 의미한다. 
다른 용어로는 `snapshot` 이라고 불리기도 한다. 
앞서 설명한 `AOF` 에 비해 수행된 명령에 대한 정보가 없어 `AOF` 파일 보다는 크기가 작다. 
그로인해 `Redis` 서버 시작 시점에 로딩 속도가 더 빠를 수 있다. 
대략적으로 `30%` 정도의 시간차이를 보인다고 한다.  

### snapshot

`RDB` 방식의 저장 시점은 설정파일에서 `save` 파라미터를 사용해서 설정 가능하다. 
하나 이상 지정 가능하고 그 예시는 아래와 같고, 여러 설정을 지정한 경우 `or` 조건으로 판별한다. 

```
save 900 1  # 900초 동안 1번 이상 key 변경이 발생하면 저장
save 300 10  # 300초 동안 10번 이상 key 변경이 발생하면 저장
save 60 1000  # 60초 동안 1000번 이상 key 변경이 발생하면 저장
```  

`snapshot` 으로 저장되는 `RDB` 파일 이름은 `dump.rdb` 이다. 
그리고 `BGSAVE`, `SAVE` 명령어를 사용해서 명시적으로 `RDB` 파일을 저장 할 수 있다.  


### RDB 관련 설정
- `stop-write-on-bgsave-error` : 기본 값은 `yes` 이고, 
`yes` 일때 `RDB` 파일 저장 수행 중 실패하면 모든 쓰기 요청을 거부한다. 
명시적으로 `RDB` 수행 실패를 알리는 역할을 할 수 있다. 
`no` 로 설정하게 되면 실패하더라도 `Redis` 서버는 정상적으로 모든 요청을 수행하게 된다. 
서비스의 가용성이 중요하고 모니터링으로 식별 가능하다면 `no` 로 설정하더라도 무방하다. 
`RDB` 저장이 실패하는 경우는 디스크 공간이 부족하거나, 권한 문제, 물리 장비 이슈 등이 있다. 
해당 옵션에 대한 동작은 설정파일을 통한 `save` 이벤트에만 해당하고 `BGSAVE` 를 명시적으로 사용한 경우에는 해당되지 않는다. 
- `rdbcompression` : 기본 값은 `yes` 이고, 
`RDB` 파일을 쓸때 압축 수행 여부를 결정한다. 
사용하는 압축 알고리즘은 `LZF` 이고 압축률은 높지 않다고 한다. 
- `rdbchecksum` : 기본 값은 `yes` 이고, 
`RDB` 파일 끝에 `CRC64 checksum` 값을 기록한다. 
- `dbfilename` : 기본 값은 `dump.rdb` 이고, 
`RDB` 파일 명을 지정할수 있지만, 경로는 지정할 수 없다. 


### RDB 테스트
테스트를 위해 `save 1 2` 설정으로 `Redis` 서버를 `Docker` 를 사용해서 실행한다. 
그리고 `Redis` 서버에 접속한 다음 `info persistence` 명령으로 `RDB` 관련 설정 정보를 확인하면 아래와 같다. 

```bash
$ docker run -d --rm -v ${PWD}/data:/data --name redis-rdb redis:6 redis-server --save 1 2
d7bd7ca1198db006bf700f571d919932a0742a87398e3a8183150404eda4854b
$ docker exec -it redis-rdb /bin/bash
root@d7bd7ca1198d:/data# redis-cli
127.0.0.1:6379> info persistence
# Persistence
loading:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0  # 현재 진행 중인 rewrite 가 있는지 여부
rdb_last_save_time:1606385468  # 마지막 RDB 파일 저장 시간
rdb_last_bgsave_status:ok  # 마지막 RDB 파일 저장 성공 여부
rdb_last_bgsave_time_sec:-1  # 마지막 RDB 파일 저장에 걸린 시간
rdb_current_bgsave_time_sec:-1  # 현재 RDB 파일 저장 시작하고 경과 시간

.. 생략 ..
```  

현재 `RDB` 저장 시점은 `save 1 2` 이기 때문에, 
매초 마다 2번 이상 변경이 발생하면 `RDB` 파일을 저장하게 될 것이다. 

```bash
127.0.0.1:6379> set 1 1
OK
127.0.0.1:6379> info persistence
# Persistence
loading:0
rdb_changes_since_last_save:1
rdb_bgsave_in_progress:0
rdb_last_save_time:1606537251
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
127.0.0.1:6379> set 2 2
OK
127.0.0.1:6379> info persistence
# Persistence
loading:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0
rdb_last_save_time:1606537588
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:0
rdb_current_bgsave_time_sec:-1
```  

2번 이상의 변경이 발생한 시점 이후 `info persistence` 출력 결과를 살펴보면, 
`rdb_last_save_time`, `rdb_last_bgsave_time_sec` 등의 값이 변경 된 것을 확인할 수 있다. 
`Redis` 서버 로그를 확인하면 아래와 같다. 

```bash
$ docker logs redis-rdb

.. 생략 ..

# save 조건 만족
1:M 28 Nov 2020 04:26:28.560 * 2 changes in 1 seconds. Saving...  
# RDB 파일 저장을 위해 자식 프로세스 fork()
1:M 28 Nov 2020 04:26:28.561 * Background saving started by pid 25
# 자식 프로세스에서 데이터를 디스크에 저장
25:C 28 Nov 2020 04:26:28.586 * DB saved on disk
25:C 28 Nov 2020 04:26:28.586 * RDB: 4 MB of memory used by copy-on-write
1:M 28 Nov 2020 04:26:28.661 * Background saving terminated with success
```  

`save` 조건을 만족했을 때 `BGSAVE` 를 통해 자식 프로세스를 사용해서 메모리 데이터를 디스크에 쓰는 것을 확인 할 수 있다.  

`BGSAVE` 는 필요에 따라 명시적으로 명령을 호출해서 파일을 저장하는 것도 가능하다. 
`BGSAVE` 의 동작 과정은 아래와 같다. 
1. 부모(메인) 프로세스에서 자식 프로세스 `fork()` 한다. 
1. 자식 프로세스는 메모리에 있는 데이터를 새로운 `RDB` 임시 파일에 쓴다. 
1. 임시 파일 쓰기가 완료되면 기존 파일은 삭제하고, 임시 파일의 이름을 변경한다. 

```bash
.. Redis Server ..
127.0.0.1:6379> bgsave
Background saving started

.. Bash ..
$ docker logs redis-rdb

.. 생략 .. 

1:M 28 Nov 2020 18:10:07.482 * Background saving started by pid 25
25:C 28 Nov 2020 18:10:07.487 * DB saved on disk
25:C 28 Nov 2020 18:10:07.488 * RDB: 4 MB of memory used by copy-on-write
1:M 28 Nov 2020 18:10:07.555 * Background saving terminated with success
```  

자식 프로세스가 먼저 쓰는 임시 파일의 이름은 `temp-<pid>.rdb`(`temp-25.rdb`) 이다.  

먼저 살펴본 `BGSAVE` 는 메인 프로세스에서 쓰기 작업을 수행하는 것이 아닌, 
자식 프로세스를 만들어 메인 프로세스는 계속해서 명령어 수행 작업이 가능하다. 
하지만 `SAVE` 는 메인 프로세스에서 `RDB` 파일 쓰기 작업을 수행하기 때문에, 
작업이 끝날 때까지 명령어 수행작업이 지연된다. 
`SAVE` 의 동작 과정은 아래와 같다. 
1. 메인 프로세스가 메모리의 데이터를 새로운 `RDB` 임시 파일로 쓴다. 
1. 임시 파일 쓰기 작업이 완료되면 기존 파일은 삭제하고, 임시 파일의 이름을 변경한다. 

```bash
.. Redis Server ..
127.0.0.1:6379> save
OK

.. Bash ..
$ docker logs redis-rdb

.. 생략 ..

1:M 28 Nov 2020 18:17:18.434 * DB saved on disk
```  


## 권장 설정 
`Redis` 서버의 데이터에 대한 지속성을 보장하기 위해서는 `AOF` 와 `RDB` 를 모두 적절하게 사용하는 것을 권장한다. 
`AOF` 를 기본으로 사용하고 `RDB` 는 옵션의 역할을 수행하는 방식이다. 
여기서 `AOF` 의 주기는 `erverysec` 으로 하고 `rewrite` 는 적절하게 운용될 수 있도록 설정이 필요하다.  

`Redis` 서버가 `AOF`, `RDB` 파일을 읽어 들이는 시점은 서버를 재시작 할 때이다. 
별도로 서버가 실행 중인 상태에서 데이터를 읽는 명령은 제공하지 않는다. 
그리고 `AOF` 와 `RDB` 파일이 모두 존재하는 경우 읽어 들이는 파일은 설정옵션에 따라 아래와 같다. 
1. `appendonly yes` 이라면 `AOF` 파일을 읽는다. 
1. `appendonly no` 이라면 `RDB` 파일을 읽는다. 

`AOF` 가 활성화 된 상태에서 `AOF` 파일은 존재하지 않고 `RDB` 파일이 존재하더라도 `RDB` 파일은 읽지 않는다. 
위 상황과 반대로 `AOF` 가 비활성화 된 상태에서 `RDB` 파일은 존재하지 않고 `AOF` 파일이 있더라도 읽지 않는다. 

### 테스트
가장 만저 `Redis` 서버가 시작 될때 `AOF`, `RDB` 파일을 읽으면서 발생하는 로그를 살펴본다.  

서버가 시작할 때 `AOF` 파일을 읽는 로그는 아래와 같이 `DB loaded from append only file: 0.002 seconds` 라고 출력된다. 

```bash
.. AOF Redis Server 컨테이너 실행 ..
$ docker run -d --rm -v ${PWD}/data:/data --name redis-aof redis:6 redis-server --appendonly yes --auto-aof-rewrite-min-size 10b
19f345545e3451c1bf940469a0534be0644568914a78c4c5e55ba06010c341df
$ docker exec -it redis-aof /bin/bash
root@19f345545e34:/data# redis-cli

.. 데이터 추가 ..
127.0.0.1:6379> set aof1 1
OK
127.0.0.1:6379> set aof2 2
OK
127.0.0.1:6379> exit
root@19f345545e34:/data# exit
exit

.. AOF Redis Server 컨테이너 재시작 ..
$ docker restart redis-aof
redis-aof
$ docker logs redis-aof
.. 생략 ..
1:M 28 Nov 2020 18:32:52.994 * DB loaded from append only file: 0.002 seconds
1:M 28 Nov 2020 18:32:52.994 * Ready to accept connections
```  

서버가 시작 할때 `RDB` 파일을 읽는 로그는 아래와 같이 `DB loaded from disk: 0.002 seconds` 라고 출력된다. 

```bash
.. RDB Redis Server 컨테이너 실행 ..
$ docker run -d --rm -v ${PWD}/data:/data --name redis-rdb redis:6 redis-server --save 1 2
7bc103b8515b62230638e385ad0cb583d8df96b3d167ba119bb1b19517f92086
$ docker exec -it redis-rdb /bin/bash
root@7bc103b8515b:/data# redis-cli

.. 데이터 추가 ..
127.0.0.1:6379> set rdb1 1
OK
127.0.0.1:6379> set rdb2 2
OK
127.0.0.1:6379> exit
root@7bc103b8515b:/data# exit
exit

.. RDB Redis Server 컨테이너 재시작 ..
$ docker restart redis-rdb
redis-rdb
$ docker logs redis-rdb
.. 생략 ..
1:M 28 Nov 2020 18:39:10.851 * DB loaded from disk: 0.002 seconds
1:M 28 Nov 2020 18:39:10.851 * Ready to accept connections
```  


다음으로는 `RDB` 파일만 있을때 `AOF` 를 활성화해서 이후에도 `AOF` 파일로 운용이 가능하도록 하는 방법에 대해 알아본다. 
먼저 테스트를위해 `RDB` 만 활성화된 `Redis` 서버를 실행하고, 몇개의 데이터를 추가하면 `/data` 경로에 `dump.rdb` 파일이 생성된 것을 확인 할 수 있다. 

```bash
$ docker run -d --rm -v ${PWD}/data:/data --name redis-test redis:6 redis-server --save 1 2
4c80566187a23dde77709c94d60f46a707a748a4e46399c0ddaf157bd61276b8
$ docker exec -it redis-test /bin/bash
root@4c80566187a2:/data# redis-cli
127.0.0.1:6379> set rdb1 1
OK
127.0.0.1:6379> set rdb2 2
OK
127.0.0.1:6379> exit
root@4c80566187a2:/data# ls
dump.rdb
```  

현재 상황은 `RDB` 파일만 존재하기 때문에 `RDB` 파일 저장전 서버에 이상이 생기면 마지막 저장시점 이후의 데이터는 모두 손실된다. 
이 상황에서 `Redis` 서버를 재시작하지 않고 `AOF` 를 활성하과 하고 기존 



















다음으로는 `RDB` 파일만 있을때 이를 `AOF` 를 활성화 해서 읽어 들이는 방법에 대해 알아본다. 
또는 그 반대 상황도 .. ??

















































---
## Reference
[Redis Persistence](https://redis.io/topics/persistence)
[Redis Persistence Introduction](http://redisgate.kr/redis/configuration/persistence.php)