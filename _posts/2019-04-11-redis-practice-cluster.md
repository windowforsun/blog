--- 
layout: single
classes: wide
title: "Redis Cluster 설정하기"
header:
  overlay_image: /img/redis-bg.png
excerpt: 'Redis 에서 Cluster 설정을 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Redis
tags:
  - Redis
  - Cluster
---  

gem install redis

yum install ruby-devel ruby-irb ruby-rdoc ruby-ri

출처: https://binshuuuu.tistory.com/30 [지식저장소]




redis config file paht is /etc/redis.conf

redis local cluster 설정관련
https://binshuuuu.tistory.com/30
https://blog.leocat.kr/notes/2017/11/07/redis-simple-cluster
https://brunch.co.kr/@daniellim/31
http://redisgate.jp/redis/cluster/cluster_start.php



## 설치 안해도 될듯 redis-cli --cluster 로 가능한듯 내껀 5버전이라서
ruby 2.3 설치
https://zetawiki.com/wiki/CentOS6_ruby-2.3_%EC%84%A4%EC%B9%98

gem update

gem update --system

gem install redis


---
## Reference
[Redis CLUSTER Start](http://redisgate.kr/redis/cluster/cluster_start.php)  
[Redis 설치 및 Cluster 구성](https://brunch.co.kr/@daniellim/31)  
[[Redis] Redis cluster 구성하기](https://blog.leocat.kr/notes/2017/11/07/redis-simple-cluster)  
[CentOS (6.8ver) - redis cluster 구성 (master-slave & cluster)](https://binshuuuu.tistory.com/30)  
[[Redis] 클러스터 생성 및 운영](https://bstar36.tistory.com/361)  