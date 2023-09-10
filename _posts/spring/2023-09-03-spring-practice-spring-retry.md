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

## Spring Retry
[Spring Retry](https://github.com/spring-projects/spring-retry)
는 실패한 동작에(예외 발생) 대해서 설정된 내용에 맞게 재시도를 수행 할 수 있도록 기능을 제공한다. 
`Spring` 대부분 기능들의 공통적인 특정처럼 기존 비지니스 로직을 크게 건들지 않고 몇가지 설정 등 추가로 적용 할 수 있다.  

어느 서비스를 개발하다 보면 단발적인 예외가 밸상하는 경우가 있다. 
이러한 예외 중 몇가지 케이스는 다시 시도를 통해 해결되는 경우가 있는데, 
이때 사용할 수 있는게 바로 `Spring Retry` 이다.  

`keep-alive` 를 통해 내부적으로 통신을 하는 두 애플리케이션이 있다고 가정한다. 
요청을 받는 서버 애플리케이션이 배포된다면 기존 `keep-alive connection` 은 서버로 인해 끊기게 된다.
(일반적으로 커넥션은 클라이언트에서 끊어야 정상적이다.) 
이런 경우 요청을 하는 클라이언트 애플리케이션에서는 `connection reset` 과 같은 예외가 발생하게 되는데, 
이 경우는 잠깐 대기 후 동일한 요청을 재시도 해주면 정상 응답이 가능한 경우가 대부분이다.  


`Spring Retry` 에서 제공하는 기능과 주요 특징을 나열 하면 아래와 같다. 

- `@EnableRetry`, `@Retryable` : `Annotation` 으로 재시작 동작을 정의하고 적용한다. `@EnableRetry` 로 애플리케이션에 선언하면 적용되고, `@Retryable` 로 재시작 타겟이 되는 메서드에 선언하며 재시도 동작을 정의 할 수 있다.  
- `RetryTemplate` : `Spring` 에서 제공하는 `RetryOperation` 의 간단한 구현체로 콜백에 직접 비지니스 로직을 넣어 재시작 동작을 정의하거나 커스텀 할 수 있다. 
- `Listeners` : `Retry Annotation` 이나 `RetryTemplate` 에 정의해서 재시도 시작, 재시도 콜백 호출, 재시도 종료 등의 콜백을 받을 수 있다. 
- `RetryContext` : `RetryCallback` 메서드가 받는 파라미터로, 실행을 반복하는 동안 필요한 데이터 속성을 저장해서 사용 할 수 있다. 
- `RecoveryCallback` : 재시도를 완료한 상태에서도 해결되지 않는 경우 `RecoveryCallback` 으로 제어권을 넘겨 최종적으로 실패한 경우 `rocevery` 로직이 실행 될 수 있도록 한다. 
- `Stateless Retry` : `RetryContext` 를 전역적인 사용하지 않고 콜 스택에서만 사용하는 경우로, 상태가 없는 재시작이라고 하며 이는 항상 동일 스레드에서 수행된다. 
- `Stateful Retry` : 트랜잭션과 역인 경우 등으로, `RetryCallback` 이 이미 콜스택에 나와 `Stateless Retry` 로는 해결이 불가한 경우 `RetryContext` 를 `RetryContextCache` 로 전역에 저장해서 사용하는 경우이다. 
- `Retry Policies` : 재사작 동작에 대한 정책을 정의 할 수 있다. 횟수, 소요시간, 적용할 예외, 제외할 예외 등이 있고, 대표적으로 `SimpleRetryPolicy` 와 `TimeoutRetryPolcy`(둘다 `Stateless Retry`) 가 있다. 
- `Backoff Policies` : 실패가 일시적인 경우 잠시 기다렸다가 하는 경우에 대한 정책을 정의한다. 대기 주기나, 시작 주기 등을 정의 할 수 있다.

### buidl.gradle
`Spring Retry` 를 프로젝트에 적용하기 위해서는 아래와 같은 의존성이 필요하다.  

```groovy
dependencies {
    // ...
    implementation 'org.springframework.retry:spring-retry'
    implementation 'org.springframework:spring-aspects'
    // ...
}
```  
