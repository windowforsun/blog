--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Redis Data 로 Optimistic Locking 구현하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Optimistic Locking 을 실제로 구현해서 Synchronized 동작을 수행해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
  - Spring
  - Practice
  - Redis
  - Synchronized
---  

## Spring Redis Data 에서 Optimistic Locking
- Redis 에서 Transaction 이나 Optmistic Locking 을 구현하기 위해서는 `WATCH`, `MULTI`, `EXEC` 을 통해 가능하다.
	- 더 자세한 [링크]({{site.baseurl}}{% link _posts/redis/2019-04-24-redis-concept-redistransaction.md %})
- Spring Redis Data 에서는 이러한 동작을 `RedisCallback`, `SessionCallback` 인터페이스를 구현해서 수행 가능하다.
	- Pipeline 동작, Transaction 동작도 구현 가능하다.

---
## Reference
