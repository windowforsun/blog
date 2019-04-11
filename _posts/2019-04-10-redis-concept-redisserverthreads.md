--- 
layout: single
classes: wide
title: "Redis Server Threads 관련"
header:
  overlay_image: /img/redis-bg.png
excerpt: 'Redis Server 의 Thread 에 관련해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Thread
---  

## Redis 는 Single Thread ?
- Redis 는 Multi Thread 이다.
- Redis 3.2 까지는 3개 Thread, 4.0 부터는 4개 Thread 가 동작한다.
- 3.2 까지는 AOF 관련 처리만 Sub Thread 에서 수행했기 때문에 Single Thread 로 알려졌다.
- 4.0 부터는 UNLINK 명령을 Sub Thread 에서 처리하면서 Multi Thread 라고 해야한다.
- Sub Thread 관련 소스 코드는 bio.c 에서 확인 가능하다.

### Main Thread
- 아래 3개의 Sub Thread 에서 처리하는 것 이외에 거의 모든 명렁어 처리, 이벤트 처리 등을 수행한다.

### Sub Thread 1번(BIO_CLOSE_FILE)
- AOF Rewrite 작업에서 새파일에 Rewrite 완료하고 기존 파일을 close 할 때 동작한다.
- AOF 를 활성화하지 않아도 Thread 는 생성된다.

### Sub Thread 2번(BIO_AOF_FSYNC)
- 1초 마다 AOF 에 쓸 때 동작한다.

### Sub Thread 3번(BIO_LAZY_FREE)
- UNLINK, 비동기 FULSHALL 또는 FLUSHSUB 명령을 처리할 때 동작한다.
- 이 Thread 는 4.0 에 추가되었다.

## 리눅스에서 Redis Thread 확인하기

```
[root@windowforsun ~]# ps -eLF | grep redis
UID        PID  PPID   LWP  C NLWP    SZ   RSS PSR STIME TTY          TIME CMD
redis     1365     1  1365  0    4 38127  7400   0 Apr08 ?        00:06:52 /usr/bin/redis-server 127.0.0.1:6379      
redis     1365     1  1371  0    4 38127  7400   0 Apr08 ?        00:00:00 /usr/bin/redis-server 127.0.0.1:6379      
redis     1365     1  1372  0    4 38127  7400   0 Apr08 ?        00:00:00 /usr/bin/redis-server 127.0.0.1:6379      
redis     1365     1  1373  0    4 38127  7400   0 Apr08 ?        00:00:00 /usr/bin/redis-server 127.0.0.1:6379      
```  

- LWP 는 Light Weight Process 로, Thread ID 이다.
- NLWP 는 이 프로세스의 총 쓰레드 개수이다.
- 첫 번째가 Main Thread, 두 번째 ~ 네 번째까지가 Sub Thread 1, 2, 3번 이다.

---
## Reference
[Redis Server Threads 쓰레드](http://redisgate.kr/redis/configuration/redis_thread.php)  

