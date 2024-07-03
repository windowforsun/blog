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

### Properties Class
아레는 `Spring Cloud Stream` 을 사용 할때 `Properties` 로 설정할 수 있는 옵션의 내용이다.  

```java
@Data
@ConfigurationProperties("elasticsearch.sink")
public class ElasticsearchSinkProperties {
    private String index;
    private String dateTimeRollingFormat;
    private Expression id;
    private String routing;
    private long timeoutSeconds;
    private boolean async;
    private int batchSize = 1;
    private long groupTimeout = -1L;
}
```  

- `index` : `Elasticsearch` 에 문서 저장할 인덱스 이름이다. 
- `dateTimeRollingFormat` : `index` 를 생성할 때 롤링 시킬 시간 포맷을 지정하면, `index-yyyy-MM-dd` 와 같은 인덱스를 사용하게 된다. 
- `id` : `Elasticsearch` 문서 색인을 위한 아이디를 [Expression](https://docs.spring.io/spring-framework/reference/core/expressions.html)(SpEL) 형식으로 지정한다. 
- `routing` : `Elasticsearch` 문서를 저장할 때 라우팅을 지정한다. 
- `timeoutSeconds` : 문서를 저장할 때 타임아웃 시간을 지정한다. 
- `async` : 문서를 저장할 때 비동기 방식을 사용할지에 대한 설정이다. 
- `batchSize` : 문서를 저장 할 때 한번에 전송할 수를 입력한다. 
- `groupTimeout` : 문서를 `batch` 방식으로 저장 할때 `batchSize` 까지 대기할 최대 시간을 입력한다.  

`Spring Cloud Data Flow` 에서 애플리케이션을 실행하거나, 
그렇지 않더라도 `Properties` 를 사용해서 소비하는 메시지에 맞춰 `Elasticsearch` 에 문서로 저장 할 수 있다.  
