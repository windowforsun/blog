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
