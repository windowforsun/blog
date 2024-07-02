--- 
layout: single
classes: wide
title: "[Spring 실습] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
toc: true
use_math: true
---  

## Spring Cloud Stream Elasticsearch Sink
`Spring Cloud Stream Application` 중 [ElasticsearchSink](https://github.com/spring-cloud/stream-applications/blob/main/applications/sink/elasticsearch-sink/README.adoc)
은 이미 제공중인 애플리케이션이 존재한다. 
하지만 구성 환경에서 버전문제로 인해(`Elasticsearch 7.10` 에 맞는 애플리케이션 필요)
이를 다시 구현해 보는 시간을 가졌다. 
`Spring Cloud` 에서 구현해둔 것을 대부분 차용하면서 해당 코드를 분석하고 파악하는 시간도 가졌다. 
그리고 추가적으로 `Elasticsearch` 인덱스를 구성할 때 `Properties` 로 시간 포맷을 넣어 롤링이 될 수 있도록 하는 기능도 추가해 보았다.  

구현한 전체 코드는 [Elasticsearch Sink](https://github.com/windowforsun/spring-cloud-stream-es-sink)
에서 확인 할 수 있다.  

### Properties

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder: localhost:9092
      function:
        bindings:
          elasticsearchConsumer-in-0: input
      bindings:
        input:
          destination: output
  elasticsearch:
    rest:
      uris: 'http://localhost:9200'
```  

`Upstream` 에서 방출되는 `out channel` 은 `ElasticsearchSink` 의 `input channel` 과 매핑되어 
`elasticsearchConsumer` 빈을 사용하는 `elasticsearchConsumer-in-0` 으로 바인딩 되어 이후 `Sink Process` 가 수행된다.   

