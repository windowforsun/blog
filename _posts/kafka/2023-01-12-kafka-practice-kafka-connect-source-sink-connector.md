--- 
layout: single
classes: wide
title: "[Kafka]"
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
toc: true
use_math: true
---  

## Kafka Connect Source, Sink Connector
`Kafka Connect` 는 크게 소스로 부터 데이터를 읽어와 카프카에 넣는 `Source Connector` 와 
카프카에서 데이터를 읽어 타겟으로 전달하는 `Sink Connector` 로 구성된다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-source-sink-connector-1.drawio.png)  

이번 포스트에서는 직접 `Source Connector` 와 `Sink Connector` 를 구현해서 `Kafka Connect` 를 통해 실행하는 방법에 대해 알아본다. 
그리고 커스텀하게 구현한 `Source Connector` 외 `Sink Connector` 를 `docker` 를 기반으로 `Kafka Cluster` 와 연동해 실행하는 방법까지 알아본다. 

### Source Connector
소스 커넥터는 소스 애플리케이션 혹은 소스 파일로 부터 데이터를 읽어와 토픽으로 넣는 역할을 한다. 
이미 구현돼 있는 오픈소스를 사용해도 되지만 라이센스 혹은 추가 기능등이 필요 한 경우는 카프카 커넥트 라이브러리에서 제공하는 
`SourceConnector` 와 `SourceTask` 를 사용해서 직접 구현 가능하다. 
직접 구현한 소스 커넥터는 빌드해서 `jar` 파일로 만든 후 카프카 커넥트 실행 시 플러그인으로 추가해서 사용해야 한다. 

![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-source-sink-connector-2.drawio.png)  

### Sink Connector


![그림 1]({{site.baseurl}}/img/kafka/kafka-connect-source-sink-connector-3.drawio.png)  



---  
## Reference
[Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)  
[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html#status-and-errors)  
[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#how-kafka-connect-works)  
[아파치 카프카](https://search.shopping.naver.com/book/catalog/32441032476?cat_id=50010586&frm=PBOKPRO&query=%EC%95%84%ED%8C%8C%EC%B9%98+%EC%B9%B4%ED%94%84%EC%B9%B4&NaPm=ct%3Dlct7i9tk%7Cci%3D2f9c1d6438c3f4f9da08d96a90feeae208606125%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3D60526a01880cb183c9e8b418202585d906f26cb4)  

[robcowart/cp-kafka-connect-custom](https://github.com/robcowart/cp-kafka-connect-custom)  
[conduktor/kafka-stack-docker-compose](https://github.com/conduktor/kafka-stack-docker-compose)  
