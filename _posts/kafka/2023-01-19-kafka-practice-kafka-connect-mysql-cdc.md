--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Debezium MySQL CDC Source Connector"
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
    - CDC
    - Debezium
    - MySQL
toc: true
use_math: true
---  

## CDC
`Application` 를 개별 역할을 수행하는 컴포넌트 별로 나누어 구성하는 `MSA(Micro Service Architecture)` 에서 
`EDA(Event Driven Architecture`) 로 발전하기 위해서는 신뢰성있는 `Event Bus` 를 구축하는 것이 필수이다. 
`Kafka` 를 통해 `Event Bus` 로 사용해서 데이터 인입에 따라 전체 컴포넌트들이 유기적으로 동작 할 수 있는 `CDC` 에 대해 간단하게 알아본다. 

`CDC` 는 `Change Data Capture` 의 약자로 소스가 되는 데이터의 변경을 식별해서 필요한 후속처리를 자동화 하는 기술 또는 설계 기법이자 구조를 의미한다.  

`CDC` 라는 별도의 구조 없이 `Application Layer` 에서 별도 구현을 통해 `Event Bus` 를 구성하는 방법도 있다. 
하지만 `Application Layer` 에서 신뢰할 수 있는 이벤트 발행은 복잡한 





---  
## Reference
[Introduction to Kafka Connectors](https://www.baeldung.com/kafka-connectors-guide)  
[Connect REST Interface](https://docs.confluent.io/platform/current/connect/references/restapi.html#status-and-errors)  
[Kafka Connect](https://docs.confluent.io/platform/current/connect/index.html#how-kafka-connect-works)  
[아파치 카프카](https://search.shopping.naver.com/book/catalog/32441032476?cat_id=50010586&frm=PBOKPRO&query=%EC%95%84%ED%8C%8C%EC%B9%98+%EC%B9%B4%ED%94%84%EC%B9%B4&NaPm=ct%3Dlct7i9tk%7Cci%3D2f9c1d6438c3f4f9da08d96a90feeae208606125%7Ctr%3Dboknx%7Csn%3D95694%7Chk%3D60526a01880cb183c9e8b418202585d906f26cb4)  
[robcowart/cp-kafka-connect-custom](https://github.com/robcowart/cp-kafka-connect-custom)  
[conduktor/kafka-stack-docker-compose](https://github.com/conduktor/kafka-stack-docker-compose)  
