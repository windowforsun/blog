--- 
layout: single
classes: wide
title: "[Java 개념] Java Log 라이브러리"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Java Log 라이브러리에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Practice
    - Java
    - LogBack
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
	- 오른쪽의 레벨이 모든 왼쪽레벨을 포함한다.
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

## SLF4J


	 



---
## Reference
[Log4j 및 LogBack](https://goddaehee.tistory.com/45)  
[Apache log4j](http://logging.apache.org/log4j/1.2/)  
[Log4j의 정의, 개념, 설정, 사용법 정리](https://cofs.tistory.com/354)  
