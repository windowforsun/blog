--- 
layout: single
classes: wide
title: "Memcached 란"
header:
  overlay_image: /img/memcached-bg.jpg
excerpt: 'Memcached 가 무엇이고, 어떠한 특징을 가지는 지 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Memcached
tags:
  - Memcached
  - In Memory
  - Caching
---  

# Memcached 란
- Memcached 는 고성능 In-Memory 분산 환경 메모리 캐싱 시스템이다.
- 주로 DB 에서 생기는 부하를 완하시키고, 성능 향상을 위해 사용된다.
- 빠른 응답시간을 제공과 확장 가능한 오픈소스 솔루션이다.
- 캐시 및 세션 데이터 스토어로 많이 사용된다.
- 웹, 모바일, 게임 등의 다양한 분야에서 사용 중이다.

## Memcached 의 동작
- 데이터를 Disk(HDD, SSD) 에 저장하는 기존의 DB 와는 달리 Memcached 는 데이터를 Memory(RAM) 에 저장한다.
- 데이터의 CRUD 연산시에 Disk 에 액새스 할 필요가 없기 때문에 Memcached 와 같은 In-Memory 시스템은 지연 시간을 줄이고, 아주 빠른 속도로 데이터 처리가 가능하다.
- 기본적으로 데이터를 Key-Value 형식으로 저장한다.
- Memcached 는 분산형 이기때문에, 새로운 노드를 추가하여 손쉬운 확장이 가능하다.
- Memcached 는 Multi-Thread 이기 때문에 효율을 높일 수 있다.
- 속도, 확장성, 단순한 설계 메모리의 효율적인 관리와 다양한 언어에 대한 API 지원으로 고성능 대규모 캐싱으로 사용된다.

# Memcached 특징
## 빠른 응답 시간
- Memcached 는 모든 데이터를 Main-Memory(RAM) 에 유지한다.
- 많은 작업을 처리하고, 더 빠른 응답 시간을 지원 가능하다.
- 1밀리초 미만의 평균 읽기 및 쓰기 시간과 초당 수백만 개의 연산 지원가능한 빠른 성능을 보여준다.

## 단순함과 편리함
- Memcached 는 단순하고 직관적으로 설계되었다.
- 다양한 언어(Java, PHP, C, C++, Node, Go ..) 에서 사용할 수 있는 오픈소스 클라이언트를 사용할 수 있다.

## 확장성
![memcached 확장성]({{site.baseurl}}/img/memcached/memcached-usage.png)
- Memcached 의 분산 및 Multi-Thread Architecture 를 사용하면 손쉬운 확장이 가능하다.
- 여러 노드간에 데이터를 나눌 수 있고, 클러스터에 새로운 노드 추가를 통해 용량도 확장 가능하다.
- Memcached 는 Multi-Thread 이므로 주어진 노드에서 여러 개의 코어를 사용할 수 있어, 성능적인 확장도 가능하다.
- 일관된 성능을 보장하고 확장성 가능한 분산 캐싱 솔루션을 구축할 수 있다.

## 커뮤니티
- 활발한 커뮤니티에서 지원하는 오픈 소스 프로젝트이므로 다양한 정보를 쉽게 얻을 수 있다.

# 활용 사례
## 캐싱
- 고성능 In-Memory 캐시를 구현하여 데이터 액세스 지연 시간을 줄이고 처리량을 늘리며 백엔드에 대한 부하를 줄일 수 있다.
- 캐싱 종류
	- DB 쿼리 결과 캐싱
	- 세션 캐싱
	- 웹 페이지 캐싱
	- API 캐싱
	- 이미지, 파일 및 메타데이터 캐싱
	- 객체 캐싱

## 세션 저장
- 애플리케이션의 세션의 지속성이 중요하지 않다면 세션을 Memcached 로 관리하여 빠른 성능을 얻을 수 있다.

---
## Reference
[Memcached](https://memcached.org/)  
[AWS-Memcached](https://aws.amazon.com/ko/memcached/)  
