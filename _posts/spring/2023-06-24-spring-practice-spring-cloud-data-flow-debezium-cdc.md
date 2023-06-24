--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Data Flow(SCDF) MySQL 연동"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'SCDF 의 데이터를 외부 저장소 MySQL 과 연동해 저장하고 관리해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Cloud Data Flow
    - SCDF
    - MySQL
toc: true
use_math: true
---  




```


./gradlew debezium-cdc-index-processor:jibDockerBuild

docker tag debezium-cdc-index-processor:v1 windowforsun/debezium-cdc-index-processor:v1

docker push windowforsun/debezium-cdc-index-processor:v1

```


---  
## Reference
[]()  