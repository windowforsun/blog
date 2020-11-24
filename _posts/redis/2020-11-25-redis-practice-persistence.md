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
그리고 `docker exec` 명령으로 컨테이너 `Bash` 에 접속한다. 

```bash
$ docker run -d --rm -v ${PWD}/data:/data --name redis-test redis:6 redis-server --appendonly yes --auto-aof-rewrite-min-size 10b
d126ab607dacd2d9861a02adb585f996210a42cca81a5d4e8bb94054508fa8de
$ docker exec -it redis-test /bin/bash
root@d126ab607dac:/data#
```  


























































---
## Reference
[Redis Persistence](https://redis.io/topics/persistence)
[Redis Persistence Introduction](http://redisgate.kr/redis/configuration/persistence.php)