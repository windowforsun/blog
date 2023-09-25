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
    - REST API
    - Rest API
toc: true
use_math: true
---  

## Spring Logback Logstash
[logstash-logback-encoder](https://github.com/logfellow/logstash-logback-encoder)
는 `Spring` 에서 기본 로깅으로 사용하는 `Logback` 을 통해 `Logstash` 로 바로 애플리케이션 로그를 전송할 수 있는 
`Appender` 와 `Encoder` 를 제공한다.  

일반적으로 애플리케이션의 로그를 `Aggegation` 하기 위해 파일, `stdout` 혹은 `syslog` 
등으로 남긴 로그를 `Filebeat` 을 통해 `Logstash` 로 전송하는 경우가 많다. 
물론 이러한 방식을 사용하는 것은 각 요소마다 역할이 있기 때문이지만, 
애플리케이션의 출력 로그를 바로 `Logstash` 를 전송 할 수 있는 라이브러리가 있어 사용해보려 한다.  

라이브러를 사용해서 `logstash` 에 로그를 전송하는 간단한 예제와 
그 성능에 대해 알아본다.  
