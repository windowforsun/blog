--- 
layout: single
classes: wide
title: "[Kafka] "
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Connect
toc: true
use_math: true
---  

## Kafka Connect
`Kafka Connect` 는 카프카 오픈소스로 데이터 파이프라인 생성 시 반복 작업을 최소화 하고 효율적인 전송을 도와주는 애플리케이션이다. 
`Kafka Connect` 가 없으면 파이프라인을 생성할 때 마다 프로듀서, 컨슈머 애플리케이션을 매번 개발 및 배포 운영 해줘야 한다. 
이러한 불편함은 `Kafka Connect` 를 사용해서 파이프라인의 작업을 템플릿으로 만들어 `Connector` 형태로 실행하는 방식으로 반복 작업을 중인다.  

`Connector` 는 프로듀서 역할을 하는 `Source Connector(소스 커넥터)` 와 컨슈머 역할을 하는 `Sink Connector(싱크 커넥터)` 로 나뉜다. 
이러한 `Connector` 를 통해 파일, `MySQL`, `S3`, `MongoDB` 등에 데이터를 가져오거나 데이터를 저장 할 수 있다.  






---  
## Reference
[Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)  
[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html#status-and-errors)  
[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#how-kafka-connect-works)  
