--- 
layout: single
classes: wide
title: "[Java 개념] Java Log 라이브러리"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Log 라이브러리에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Practice
    - Java
    - Logback
    - Log4j
    - SLF4J
---  

## 1. Log4j
- Log for Java 의 약자이다.
- Log4j는 Apache Logging Service 라는 Top Project 이다.

### 1.1. 기능 및 특징
- 속도에 최적화 되어 있다.
- 유연성을 고려하여 디자인되었다.
- 멀티스레드 환경에서도 안전하다.
- 계층적인 로그 설정과 처리를 지원한다.
- 아래와 같은 로그 출력을 지원한다.
 	- 파일
 	- 콘솔
 	- `java.io.OutputStream`
 	- `java.io.Writer`
 	- TCP 사용하는 원격 서버
 	- 원격 Unix Syslog 데몬
 	- 원격 JMS 구독자
 	- 윈도우 NT EventLog
 	- E-Mail
- 아래와 같은 로그 메시지 레벨을 사용한다.
 	- TRACE
 	- DEBUG
 	- INFO
 	- WARN
 	- ERROR
 	- FATAL
- 로그 출력 형식은 Layout 클래스를 확장을 통해 쉽게 변경 가능하다.
- 로그 출력의 대상과 방법은 Appender 인터페이스로 가능하다.
- 로거 하나에 다수의 Appender 를 할당할 수 있다.

### 1.2. Log4j 구조
- Logger(Category)
	- Log4j의 핵심 클래스이다.
	- 로킹 메시지를 Appender 에 전달한다.
	- 로그 출력 여부를 런타임에 조정 가능하다.
	- 로그레벨을 가지고 있고, 로그 출력 여부는 로그레벨을 통해 결정된다.
- Appender
	- 로그의 출력위치를 결정한다. 

	Class|Desc
	---|---
	ConsoleAppender|콘솔
	FileAppender|파일
	RollingFileAppender|로그의 크기가 지정한 용량 이상이되면 다른 이름의 파일 출력
	DailyRollingFileAppender|하루 단위 로그 메시지를 파일 출력
	SMTPAppender|메일
	NTEventLogAppender|윈도우 이벤트 로그 시스템
	
- Layout
	- 로그의 출력 포맷을 지정한다.
	
	Layout|Desc
	---|---
	%d|로그의 기록 시간
	%p|로그 레벨
	%F|로깅이 발생한 프로그램 파일명
	%M|" 메소드 이름
	%I|" 호출지의 정보
	%L|" 호출지의 라인 수
	%t|" Thread 이름
	%c|" 카테고리
	%C|" 클래스 이름
	%m|로그 메시지
	%n|개행 문자
	%%|% 출력
	%r|로깅 시점까지의 걸린 시간(ms)
	%x|" Thread 의 NDC(Nested Diagnostic Context) 출력
	%X|" Thread 의 MDC(Mapped Diagnostic Context) 출력
	
### 1.3. Log4j 로그 레벨

Log Level|Desc
---|---
FATAL|가장 크리티걸한 에러
ERROR|일반적인 에러
WARN|경고성 메시지
INFO|일반적인 정보
DEBUG|상세한 일반적인 정보
TRACE|경로 추적 메시지

- FATAL-ERROR-WARN-INFO-DEBUG-TRACE
	- 지정된 레벨이 왼쪽 모든 레벨을 포함한다.
	- DEBUG 레벨일 경우 FATAL~DEBUG 까지 모두 로깅이 된다.
	
### 1.4. Log4j 의 장단점
#### 1.4.1. 장점
- 프로그램의 문제 파악이 용이하다.
- 빠른 디버깅이 가능하다.
- 로그 파익이 쉽다.
- 로그 이력을 파일, DB등으로 남길 수 있다.
- 효율적인 디버깅이 가능하다.

#### 1.4.2. 단점
- 로그에 대한 입출력으로 인해 런타임 오버해드가 발생한다.
- 전체 코드 사이즈가 증가한다.
- 빈번하게 발생되는 로그로 인해 혼란을 야기시킬 수 있다.
- 개발 중 로킹 코드 추가가 힘들다.


## 2. SLF4J
- Simple Logging Facade for Java 의 약자이다.
- 다양한 Logging Framework 를 Facade Pattern 을 통해 추상화 한것이다.
- SLF4J API 를 사용하면 구현체(Log4J, Logback..)의 종류에 종속되지 않고 일관된 로킹 코드를 작성할 수 있다.
- 배포시에 원하는 Logging Framework 를 선택할 수 있다.
- SLF4J 는 API, Binding, Bridging 세 가지 모듈을 제공한다.


## 3. Logback
- Log4j 를 바탕으로 새롭게 구현된 Logging 라이브러리이다.
- Log4j 를 바탕으로 구현되었기 때문에 많은 부분에서 비슷하고 개선되었다.
- Spring Boot 의 기본 로그 객체이다.

### 3.1. Logback 기능
#### 3.1.1. Prudent mode
- 로그 수집인 Log Aggregation 기능이다.
- 하나의 서버에서 다수의 JVM이 가동중일 때 하나의 파일에 로그를 수집가능한 기능이다.
- Thread 간 Writing 경합이 발생할 수 있기 때문에 대량의 데이터의 경우 지양하는게 좋다.

#### 3.1.2. Lilith
- 발생되는 로그 이벤트에 대한 상태 정보를 모니터링 할 수 있는 뷰어이다.

#### 3.1.3. Conditional processing of configuration files
- 다른 환경(dev, qa, staging, prod..)에 따른 로그설정을 분리할 수 있다.
- Logback 설정파일에서 <if>, <then>, <else> 를 사용하여 가능하다.

#### 3.1.4. Filter

#### 3.1.5. SiftingAppender

#### 3.1.6. Stack traces with packaging data

#### 3.1.7. Logback-access

### 3.2. Logback 구조
- Logback 은 크게 Logger, Appender, Encoder 로 구성되어 있다.

#### 3.2.1. Logger
- 로깅을 수행하는 구성요소로 로깅 Level 설정으로 통해 로그의 레벨을 조절할 수 있다.
	
#### 3.2.2. Appender
- 로그 메시지가 출력될 대상을 결정하는 구성요소이다.

#### 3.2.3. Encoder
- Appender 에 포함되어 지정된 형식으로 로그 메시지를 변환하는 역할을 수행하는 구성요소이다. 

### 3.3. Logback 의 장점
- Log4j 보다 10배 정도 빠르고 메모리 효율도 더 좋다.
- Log4j 를 기반으로 많은 테스트를 통해 검증되었다.
- 문서화가 잘되어 있다.
- Logback 설정 파일을 수정하였을 때, 서버 재부팅 없이 변경 내용 적용이 가능하다.
- 서버 중지 없이 I/O Failure 에 대한 복구를 지원한다.
- RollingFileAppender 사용시에 자동적으로 오래된 로그를 지워주고, Rolling 백업을 처리한다.

### 3.4. Logback 로그 레벨

Log Level|Desc
---|---
ERROR|일반적인 에러
WARN|경고성 메시지
INFO|일반적인 정보
DEBUG|상세한 일반적인 정보
TRACE|경로 추적 메시지

- ERROR-WARN-INFO-DEBUG-TRACE
	- 지정된 레벨이 왼쪽 모든 레벨을 포함한다.
	- DEBUG 레벨일 경우 FATAL~DEBUG 까지 모두 로깅이 된다.
	
### 3.5 Logback 설정
- 아래 두 가지 방법으로 설정이 가능하다.
	- XML : 작성된 logback.xml 파일을 classpath 에 위치 시킨다.
	- Groovy : 작성된 logback.groovy 파일을 classpath 에 위치 시킨다.
- 설정 파일 우선순위 전략은 아래와 같다.
	1. logback.groovy 파일을 찾는다.
	1. 없다면 logback-test.xml 파일을 찾는다.
	1. 없다면 logback.xml 파일을 찾는다.
	1. 모두 없다면 기본 설정에 따른다.
- Spring Boot 에서 Logback 에 대한 설정을 할때 application.properties, application.yml 혹은 logback-spring.xml 을 사용한다.
	

## 4. Log4j2

---
## Reference
[Apache log4j](http://logging.apache.org/log4j/1.2/)  
[Logback Project](https://logback.qos.ch/)  
[Log4j 및 LogBack](https://goddaehee.tistory.com/45)  
[Log4j의 정의, 개념, 설정, 사용법 정리](https://cofs.tistory.com/354)  
[[Logging] SLF4J를 이용한 Logging](https://gmlwjd9405.github.io/2019/01/04/logging-with-slf4j.html)  
