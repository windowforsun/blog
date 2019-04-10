--- 
layout: single
classes: wide
title: "Redis 란"
header:
  overlay_image: /img/redis-bg.png
excerpt: 'Redis 가 무엇이고, 어떠한 특징을 가지는 지 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Caching
  - In Memory
  - NoSQL
---  

# Redis 란
- Redis 는 REmote DIctionary System 의 약자로 메모리 기반의 Key/Value Store 이다.
- Cassandra, HBase 와 같은 NoSQL DBMS 로 분류되기도 하고, Memcached 와 같은 In Memory 솔루션으로 분리되기도 한다.
- 성능 Memcached 에 버금가고, 다양한 데이터 구조체를 지원해서 Message Queue, Shared Memory, Remote Dictionary 등의 용도로 널리 사용된다.


## Redis 의 특징
### 빠른 성능
- 데이터 저장소로 디스크 또는 SSD 를 사용하는 것이 아니라, 메모리를 사용한다.
- 성능은 초당 약 10만회 명령을 실행한다.
- 서버 사양에 따라 다르지만 일반적으로 초당 2만~20만회를 실행할 수 있다.
- Redis Server Instance 는 1개 프로세스로 수행되며, 따라서 CPU 1 core 만 사용한다.
- 서버 머신 또는 VM 하나에 여러 개의 Redis Server 를 사용할 수 있다.

### In Memory 데이터 구조
- Key-Value 저장방식으로 기본적은 데이터 유형은 String 이다.
- 텍스트 또는 Binary Data 가 해당되고, 최대 크기는 512MB 이다.
- String 을 추가한 순서, 점수, 필드와 값 목록 등으로 저장할 수 있는 다양한 데이터 타입을 제공한다.

### 다양성과 편의성
- 개발과 운영을 더욱 쉽고 빠르게 할 수 있는 여러가지 도구를 제공한다.
- Pub/Sub 은 메시지를 채널에 등록하고, 채널에서 구독자에게 전달되므로 채팅과 메시징에시스템에 매우 적합하다.
- TTL 키는 해당 기간 후에는 스스로 삭제하는 지정된 Time To Live 로 불필요한 데이터를 관리할 수 있다.
- Lua 스크립팅 언어를 사용할 수도 있다.

### 복제 및 지속성
- Master-Slave 아키텍쳐를 사용하여 비동기식 복제를 지원하고 데이터가 여러 Slave 서버에 복제될 수 있다.
- 데이터의 안전한 보관과 백업을 위해 다른 서버의 메모리에 실시간으로 복사본을 남길 수 있고, 디스크에 저장하는 방법을 제공한다.
- 이러한 복제를 통해 주 서버에 장애가 발생하는 경우 요청이 어려 서버로 분산될 수 있으므로 향상된 성능과 복구 기능을 모두 제공 할 수 있다.
- 안전성을 위해 특정 시틈 스냅샷과 데이터가 변경 될 때마다 이를 디스크에 저장하는 AOF(Append Only File) 생성을 모두 지원한다.


## Redis 의 데이터 구조
Redis(5.0) 는 String 을 포함한 총 7개의 데이터 구조를 제공한다.

### String
- 일반적인 문자열로 최대 512BM 크기의 텍스트 또는 바이너리 데이터 저장이 가능하다.
- Key:Value = 일:일 관계이다.

### Lists
- LinkedList 와 비슷한 구조로 추가된 순서가 유지되는 문자열의 모음이다.
- Key:Value = 일:다 관계이다.

### Sets
- 순서가 유지되지 않는 문자열 모음으로 다른 Set 간의 교집합, 합집합, 차집합 등의 연산을 지원하는 문자열의 모음이다.
- Key:Value = 일:다 관계이다.

### Sorted Sets
- Key 와 Value, Score 를 가지며, value 를 기준으로 순서가 지정되는 Sets 이다.
- Key:(Value,Score) = 일:다 관계이다.

### Hashes
- Key 와 Field, Value 를 가지며, Field 및 Value 의 목록을 저장하는 데이터 구조이다.
- Key:(Field,Value) = 일:다

### HyperLogLogs
- 데이터 Set 내 고유 항목을 추정하기 위한 확률적 데이터 구조이다.

### BitMaps
- 비트 수준 작업을 제공하는 데이터 유형이다.


## Redis 사용 사례
### 캐싱
- RDB 와 같은 데이터베이스의 앞단에 배치된 Redis 를 통해 In Memory Cache 로 사용할 수 있다.
- 이를 통해 액세스 지연 시간은 줄이고, 처리량은 늘려 RDB, NoSQL 데이터베이스의 부담을 줄일 수 있다.

### 세션 관리
- Redis 를 세션 키에 대한 적잘 한 TTL 과 함께 빠른 키 값 스토어로 사용하여 간단하게 세션 정보를 관리할 수 있다.

### 실시간 랭킹
- Redis Sorted Set 데이터 구조를 사용하여 점수에 따라 데이터를 정렬 할 수 있다.
- 이를 사용해서 손쉽게 실시간 랭킹에 대한 처리나 가장 먼저, 가장 많이에 대한 처리를 간단하게 구현할 수 있다.

### 속도 제한
- Redis 는 이벤트 속도를 츨정하고 필요한 경우 제한 할 수 있다.
- 클라이언트의 API 키에 연결된 Redis 카운터를 사용해서 특정 기간 동안 액세스 요청의 수를 세고 초과되는 경우 조치를 취할 수 있다.
- 속도 제한은 게시물 수를 제한하거나, 리소스 사용량 등을 제한 할 수 있다.

### 대기열
- Redis List 데이터 구조를 사용하여 간단한 영구 대기열을 구현할 수 있다.
- Redis List 는 자동 작업 및 차단 기능을 제공하므로 신뢰할 수 있는 메시지 브로커 또는 순환 목록이 필요한 다양한 애플리케이션에서 사용할 수 있따.

### 채팅 및 메시징
- Redis 에서 패턴 매칭과 Pub/Sub 표준을 지원한다.
- Redis 를 사용하여 고성능 채팅방, 실시간 코멘트 스트림 및 서버 상호 통신을 지원할 수 있다.
- Pub/Sub 을 이용하여 게시된 이벤트를 기반으로 작업을 트리거도 할 수 있다.

### 다양한 미디어 스트리밍
- Redis 는 라이브 스트리밍 사용 사례를 지원할 수 있는 빠른 In Memory 데이터 스토어를 제공한다.
- Redis 는 CDN 이 동시에 수백만 명의 모바일 및 데스크톱 사용자에게 비디오를 스트리밍할 수 있도록 사용자 프로필 및 열람 기록에 대한 메타데이터, 인증 정보/토큰, 매니페스트 파일을 저장하는데 사용할 수 있다.

### 지리 공간
- Redis 는 대규모 실시간 지리 공간 데이터를 빠르게 관리할 수 있는 데이터 구조 및 연산자를 제공한다.
- 지리 공간 데이터를 실시간으로 저장, 처리 및 분석하는 GEOADD, GEODIST, GEORADIUS 및 GEORADIUSBYMEMBER 와 같은 명령을 통해 지리공간을 처리할 수 있다.
- 이를 통해 주행 시간, 주행 거리 안내 표시와 같은 위치 기반 기능을 구현할 수 있다.

### Machine Learning
- Machine Learning 은 방대한 데이터와 신속한 처리를 통해 의사 결정의 자동화를 위한 기계학습이 필요하다.
- 게임 및 금융, 데이트, 카풀 서비스의 매치메이킹과 같은 경우 수십밀리초 이내로 의사 결정을 내려야 하기 때문에 Redis 를 통해 이를 구현할 수 있다.

### 실시간 분석
- Redis 는 Apache Kafka, Amazon Kinesis 등과 같은 스트리미이 솔루션에 In Memory 데이터 스토어로 사용하여 1밀리초 미만의 지연 시간으로 실시간 데이터를 수집, 처리 분석 할 수 있다.
- 소셜 미디어 분석, 광고 타게팅, 개인화 및 IoT 와 같은 실시간 분석 사용에 적합하다.


---
## Reference
[Redis란 무엇입니까?](https://aws.amazon.com/ko/elasticache/what-is-redis/)  
[Redis Introduction](http://redisgate.kr/redis/introduction/redis_intro.php)  
[In memory dictionary Redis 소개](https://bcho.tistory.com/654)  
[Redis 개념과 특징](https://goodgid.github.io/Redis/)  
[Introduction to Redis](https://redis.io/topics/introduction)  

