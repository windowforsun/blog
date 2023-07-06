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
    - Spring Cloud Data Flow
    - SCDF
    - MySQL
toc: true
use_math: true
---  

## SCDF 에서 Transform, Router, Tap 사용하기 
`SCDF` 를 처음 구성하게 되면 기본으로 제공하는 [Spring Stream Application](https://github.com/spring-cloud/stream-applications)
을 사용할 수 있다. 
이번 포스트에서는 `Spring Stream Application` 에서 제공하는 것을 사용해서 메시지를 변환(`Transform`)하고, 
다른 채널로 전달(`Router`)하고 이를 다시 `Source` 로 사용하는 방법에 대해 알아본다. 

사용할 애플리케이션은 아래 3개이다. 
- `Transform` : [SpEL](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/functions/function/spel-function/README.adoc) 을 사용해서 메시지를 변환하는 `Processor`
- `Router` : [SpEl](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/applications/sink/router-sink/README.adoc#spel-based-routing) 혹은 [Groovy](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/applications/sink/router-sink/README.adoc#groovy-based-routing) 방식으로 메시지를 타겟이 되는 채널로 전송하는 `Sink`
- `Tap` : `SCDF` 에 정의된 특정 스트림 혹은 메시지의 토픽(채널)을 `Source` 로 사용할 수 있다. 


구현할 스트림을 도식화하면 아래와 같다. 

scdf-spring-stream-application-transform-router-1.drawio.png

실제로 생성할 스트림은 총3개로 간략한 설명은 아래와 같다. 

- `time-transform-router`
  - `Time` : 1초마다 `yyyy-MM-dd HH:mm:ss` 포맷의 시간 문자열을 생성한다. 
  - `Transform` : `Time` 에서 전달 받은 시간 문자열을 `SpEL` 을 사용해 맨끝 일의자리 초값으로 변환한다. 
  - `Router` : `Router` 에서 전달 받은 0 ~ 9 값을 `SpEL` 을 사용해 홀수 인지 짝수인지 판별 후 `test-router-even`, `test-router-odd` 채널(여기선 토픽)에 저장한다. 
- `test-router-even-log`
  - `Tap` : `test-router-even` 메시지 채널(토픽)을 `Source` 로 사용해서 스트림 연결
  - `Log` : `Tap` 으로 부터 전달 받은 짝수 값을 로그로 남김
- `test-router-even-log`
    - `Tap` : `test-router-even` 메시지 채널(토픽)을 `Source` 로 사용해서 스트림 연결
    - `Log` : `Tap` 으로 부터 전달 받은 홀수 값을 로그로 남김

### Stream 생성하기 
테스트를 위해 생성할 스트림의 정의는 아래와 같다. 

```
stream definition : time | transform | router
stream name : time-transform-router

stream definition : :test-router-even > log
stream name : test-router-even-log

stream definition : :test-router-odd > log
stream name : test-router-odd-log
```  

`CREATE STREAMS` 를 클릭한 뒤 위 `stream definition` 을 `Enter stream definition...` 에 입력해주면 된다. 

scdf-spring-stream-application.transform-router-tap-1.png

scdf-spring-stream-application.transform-router-tap-2.png

scdf-spring-stream-application.transform-router-tap-3.png

3개의 스트림을 모두 생성하면 아래 같이 리스트에서 확인 할 수 있다.  

scdf-spring-stream-application.transform-router-tap-4.png

```
stream definition : time | transform | router
stream name : time-transform-router

app.time.spring.integration.poller.fixed-rate=1000
app.time.date-format=yyyy-MM-dd HH:mm:ss
app.transform.spel.function.expression=payload.substring(payload.length() - 1)
app.router.expression=new Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'
deployer.*.kubernetes.limits.cpu=2
deployer.*.kubernetes.limits.memory=1000Mi
spring.cloud.dataflow.skipper.platformName=default

stream definition : :test-router-even > log
stream name : test-router-even-log

app.log.name=even-log
deployer.*.kubernetes.limits.cpu=2
deployer.*.kubernetes.limits.memory=1000Mi
spring.cloud.dataflow.skipper.platformName=default

stream definition : :test-router-odd > log
stream name : test-router-odd-log

app.log.name=odd-log
deployer.*.kubernetes.limits.cpu=2
deployer.*.kubernetes.limits.memory=1000Mi
spring.cloud.dataflow.skipper.platformName=default
```



---  
## Reference
[Spring Cloud Data Flow Deploying with kubectl](https://dataflow.spring.io/docs/installation/kubernetes/kubectl/)
[Stream Processing using Spring Cloud Data Flow](https://dataflow.spring.io/docs/stream-developer-guides/streams/data-flow-stream/)
