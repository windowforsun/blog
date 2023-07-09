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
이번 포스트에서는 `Spring Cloud Stream Application` 에서 제공하는 것을 사용해서 메시지를 변환(`Transform`)하고, 
다른 채널로 전달(`Router`)하고 이를 다시 `Source` 로 사용하는 방법에 대해 알아본다. 

사용할 애플리케이션은 아래 3개이다. 
- `Transform` : [SpEL](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/functions/function/spel-function/README.adoc) 을 사용해서 메시지를 변환하는 `Processor`
- `Router` : [SpEl](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/applications/sink/router-sink/README.adoc#spel-based-routing) 혹은 [Groovy](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/applications/sink/router-sink/README.adoc#groovy-based-routing) 방식으로 메시지를 타겟이 되는 채널로 전송하는 `Sink`
- `Tap` or `Destination` : `SCDF` 에 정의된 특정 스트림 혹은 메시지의 토픽(채널)을 `Source` 로 사용할 수 있다. 


구현할 스트림을 도식화하면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/spring/scdf-spring-stream-application-transform-router-1.drawio.png)


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

![그림 1]({{site.baseurl}}/img/spring/scdf-spring-stream-application.transform-router-tap-1.png)


![그림 1]({{site.baseurl}}/img/spring/scdf-spring-stream-application.transform-router-tap-2.png)

![그림 1]({{site.baseurl}}/img/spring/scdf-spring-stream-application.transform-router-tap-3.png)

3개의 스트림을 모두 생성하면 아래 같이 리스트에서 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/spring/scdf-spring-stream-application.transform-router-tap-4.png)

가장 먼저 `time-transform-router` 를 배포한다. 
스트림을 클릭후 `DEPLOY STREAM` 눌러 배포 화면으로 들어간다. 
그리고 배포 화면에서 `Freetext` 를 눌러 배포시 사용할 애플리케이션 프로퍼티를 작성해 한다.  

```properties
# time-transform-router
app.time.spring.integration.poller.fixed-rate=1000
app.time.date-format=yyyy-MM-dd HH:mm:ss
# yyyy-MM-dd HH:mm:ss 중 마지막 자리의 값으로 변환한다. 
app.transform.spel.function.expression=payload.substring(payload.length() - 1)
# transform 에서 전달 받은 값이 짝수면 test-router-even, 홀수이면 test-router-odd 토픽에 저장한다. 
app.router.expression=new Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'
deployer.*.kubernetes.limits.cpu=2
deployer.*.kubernetes.limits.memory=1000Mi
spring.cloud.dataflow.skipper.platformName=default
```  

그리고 `DEPLOY STREAM` 을 눌러 배포를 수행한다. 
배포가 완료되고 스트림이 모두 정상적으로 구동되면 `Kafka` 에 접속해 토픽을 확인하면, 
아래와 같이 `Router` 에 `SpEL` 구문으로 입력한 토픽이 생성된 것을 확인 할 수 있다.  

```bash
$ kafka-topics --bootstrap-server localhost:29092 --list
test-router-even
test-router-odd

```  

두 토픽을 각각 구독해서 저장되는 값을 확인하면 아래와 같다.  

```bash
$ kafka-console-consumer --bootstrap-server localhost:29092 --topic test-router-even
2
4
6
8
0
2
4

$ kafka-console-consumer --bootstrap-server localhost:29092 --topic test-router-odd
1
3
5
7
9
1
3
```  

`Router` 에서 라우팅 해준 데이터를 각각 `Source` 로 사용해서 스트림이 연결 될 수 있도록, 
`test-router-even-log` 스트림과 `test-router-odd-log` 스트림을 배포한다. 

```
# test-router-even-log
app.log.name=even-log
deployer.*.kubernetes.limits.cpu=2
deployer.*.kubernetes.limits.memory=1000Mi
spring.cloud.dataflow.skipper.platformName=default

# test-router-odd-log
app.log.name=odd-log
deployer.*.kubernetes.limits.cpu=2
deployer.*.kubernetes.limits.memory=1000Mi
spring.cloud.dataflow.skipper.platformName=default
```  

배포 방법은 먼저 배포한 `time-transform-router` 와 동일하다. 
배포가 완료되면 `Log` 애플리케이션에서 아래와 같이, 각 토픽에서 생산되는 메시지를 그대로 구독해 출력하는 것을 확인 할 수 있다.  

```bash
.. test-router-even-log .. 

16:00:46.867  INFO [log-sink,33867be97c78ae14,a03a56495b46ef0f] 6 --- [container-0-C-1] even-log : 6
16:00:48.869  INFO [log-sink,b9cdf2efbae4ca00,6c74ab85a11df1ce] 6 --- [container-0-C-1] even-log : 8
16:00:50.868  INFO [log-sink,7f8678423ad21903,ea9c40055cb234ea] 6 --- [container-0-C-1] even-log : 0
16:00:52.868  INFO [log-sink,7f552257cfff8663,131ff62e288146ef] 6 --- [container-0-C-1] even-log : 2
16:00:54.867  INFO [log-sink,47d0d25db84ebc41,147d91070a8fd8e1] 6 --- [container-0-C-1] even-log : 4
16:00:56.888  INFO [log-sink,3b78af819c210d51,8192936d80e02865] 6 --- [container-0-C-1] even-log : 6
16:00:58.867  INFO [log-sink,acca1d1ad8779f0e,7a23dd180b0e8e1a] 6 --- [container-0-C-1] even-log : 8
16:01:00.868  INFO [log-sink,4172a6e791cd0ea4,23fb525413804188] 6 --- [container-0-C-1] even-log : 0
16:01:02.868  INFO [log-sink,5588bbf00f4bb505,5548773b27ce33eb] 6 --- [container-0-C-1] even-log : 2
16:01:04.867  INFO [log-sink,0f996db602923cbe,65b2590c2e0b45e2] 6 --- [container-0-C-1] even-log : 4
```  

```bash
.. test-router-odd-log ..

16:01:15.867  INFO [log-sink,d7a55b00e980b682,b4cc67bbb0722c3d] 6 --- [container-0-C-1] odd-log : 5
16:01:17.873  INFO [log-sink,d6458a0ef9b5ced5,63dbfc5aadb09a42] 6 --- [container-0-C-1] odd-log : 7
16:01:19.870  INFO [log-sink,2ba62815975db4a9,bc02f68eb8433cb3] 6 --- [container-0-C-1] odd-log : 9
16:01:21.867  INFO [log-sink,8bf8b4750da07adc,bb4624d47a9b1d15] 6 --- [container-0-C-1] odd-log : 1
16:01:23.875  INFO [log-sink,4b2095f24d3f2e38,02f068b8a90da1d6] 6 --- [container-0-C-1] odd-log : 3
16:01:25.867  INFO [log-sink,9e9527dfe3c24338,c683ae418247b19f] 6 --- [container-0-C-1] odd-log : 5
16:01:27.868  INFO [log-sink,b08d8eec15210f96,1ec34284679c7032] 6 --- [container-0-C-1] odd-log : 7
16:01:29.867  INFO [log-sink,373e66758cc3106d,0f2423397402f169] 6 --- [container-0-C-1] odd-log : 9
16:01:31.867  INFO [log-sink,e73fcbdc66ded677,850761515f231610] 6 --- [container-0-C-1] odd-log : 1
16:01:33.869  INFO [log-sink,51a165d6047d71d7,4ad13c211d6692d9] 6 --- [container-0-C-1] odd-log : 3
```  

`Spring Cloud Stream Application` 에서 기본으로 제공하는 애플리케이션을 사용해서, 
`Transfrom` 으로 데이터를 변환하고 `Router` 로 데이터를 각 분류에 맞는 목적지로 전달 한뒤, 
`Tap` 을 사용해서 기존 스트림의 결과를 목적지로 별도의 스트림이 연결될 수 있도록 구성해 보았다.  


---  
## Reference
[Transform Processor](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/applications/processor/transform-processor/README.adoc)
[Router Sink](https://github.com/spring-cloud/stream-applications/blob/v4.0.0-RC1/applications/sink/router-sink/README.adoc)
