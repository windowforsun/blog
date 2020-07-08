--- 
layout: single
classes: wide
title: "[Redis 실습] Redis 에서 키 일괄 삭제하기 (Multi Delete)"
header:
  overlay_image: /img/redis-bg.png
excerpt: 'Redis 에서 특정 패턴에 일치하는 키를 모두 삭제하자'
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Delete
  - Multi Delete
  - redis-cli
---  

## 환경
- CentOS 6
- Redis 5.0

## Redis Multi Delete
- Redis 에서는 키를 일괄 삭제하는 기능을 공식적으로 지원하지 않는다.
- 특정 패턴의 키를 일괄 삭제하기 위해서는 `redis-cli` 의 명령어를 혼합해서 사용하면 가능하다.

### Redis 4 이전 버전

```
redis-cli --scan --pattern <키 패턴> | xargs redis-cli del
```  

- 키 패턴에 `user_info*` 와 같은 키 패턴을 넣어주면 해당 패턴과 일치하는 키를 모두 삭제 할 수 있다.

### Redis 4 이후 버전
- Redis 4 이후 버전 부터는 `unlink` 명령어를 사용해서 백그라운드에서 키 삭제가 가능하다.

```
redis-cli --scan --patern <키 패턴> | xargs redis-cli unlink
```  

### 명령어 설명
- `redis-cli --scan --patern <키 패턴>` 명령어에서 키 패턴과 일치하는 키목록을 가져 온다.
- `xargs` 을 통해 위에서 가져온 키 목록을 한줄로 조합해 `del` 혹은 `unlink` 커멘트에 넣어준다.
	- `del key1, key2 ,key3 ...`
- 키 패턴과 일치하는 키가 수천개 이상 있을 경우 `xargs` 는 `redis-cli` 를 여러 번 실행하며 명령어를 수행한다.

---
## Reference
[How to Delete Keys Matching a Pattern in Redis](https://rdbtools.com/blog/redis-delete-keys-matching-pattern-using-scan/)  