--- 
layout: single
classes: wide
title: "[Docker 실습] Spring Boot Docker 환경 구성"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Spring'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Docker
    - Practice
    - Spring
    - SpringBoot
---  

# 목표

docker container redis.conf bind 를 0.0.0.0 으로 모든 아이피를 허용 해야함
container 에서 설정관련은 dockerfile 에서 하는게 좋을 듯

# 방법
# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-1.png)

## pom.xml

---
## Reference
[Spring Boot: Run and Build in Docker](https://dzone.com/articles/spring-boot-run-and-build-in-docker)   
[Dockerize a Spring Boot application](https://thepracticaldeveloper.com/2017/12/11/dockerize-spring-boot/)   
[IntelliJ 에서 JAR 만들기](https://www.hyoyoung.net/100)   
[[IntelliJ] 실행 가능한 Jar 파일 생성 시 'xxx.jar에 기본 Manifest 속성이 없습니다.' 오류증상](http://1004lucifer.blogspot.com/2016/01/intellij-jar-xxxjar-manifest.html)   
[docker-compose - ADD failed: Forbidden path outside the build context](https://stackoverflow.com/questions/54287298/docker-compose-add-failed-forbidden-path-outside-the-build-context)   
[redis - Docker Hub](https://hub.docker.com/_/redis)   
[redis - Docker Hub](https://hub.docker.com/_/redis)   
[Setting up a Redis test environment using Docker Compose](https://cheesyprogrammer.com/2018/01/04/setting-up-a-redis-test-environment-using-docker-compose/)   
