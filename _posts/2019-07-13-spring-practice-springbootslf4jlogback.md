--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Boot SLF4J Logback"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Boot 에서 SLF4J Logback 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Spring
    - Practice
    - Spring Boot
    - SLF4J
    - Logback
---  

# 목표
- Spring Boot 에서 로그를 남기는 방법에 대해 익힌다.
- 환경을 분리해 로그설정파일을 관리한다.
- 로그레벨에 따라 로그를 분리해서 남긴다.
- 다양한 방법(콘솔, 파일..)으로 로그를 남겨 본다.
- 프로파일에 따라 로그설정을 분리한다.

# 방법
- 로그를 남기기 위해서 `SLF4J` 와 `Logback` 을 사용한다.
- `Logback`은 로그를 남겨주는 라이브러리 이다.
- `SLF4J`는 `Logback` 과 같은 로그 라이브러리들을 공통적으로 사용할 수 있게끔 해주는 `Facade Pattern` 구조를 가진 라이브러리 이다.
- Java 로그 관련 더 자세한 내용은 [Java Log 라이브러리]({{site.baseurl}}{% link _posts/2019-07-09-java-concept-log.md %})에서 확인 가능하다.
- Spring Boot 에서 사용하고 있는 기본 로그 라이브러리가 `SlF4J` + `Logback` 이기 때문에 추가적인 의존성 추가는 필요없다.

# 예제
## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-1.png)

## pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-boot-slf4j-logback</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>RELEASE</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```  

## Logback 설정하기
- `application.properties` 또는 `application.yaml` 을 통해 설정가능하다.

	```properties
	# logging.level.<package-dir>
	logging.level.org.springframework=ERROR
	logging.level.com.example.demo=DEBUG
	
	# log file path
	logging.path=./logs
	logging.file=app.log
	
	# logging file format
	logging.pattern.file=%d %p %c{1.} [%t] %m%n
	
	# logging console format
	logging.pattern.console=%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - $msg%n
	```  
	
- 좀더 커스텀마이징한 설정을 위해서는 `resource` 경로에 `logback-spring.xml` 파일을 통해 설정해야 한다.
	- Spring 의 경우 `logback.xml` 파일이 필요하지만 Spring Boot 에서는 `logback-spring.xml` 파일명을 사용해야 한다.
	
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 로그파일 경로 -->
    <property name="LOG_PATH" value="./logs"/>

    <!-- Appender 를 통해 출력 위치를 결정한다. -->
    <!-- 콘솔 로깅 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- Appender에 속한 Layout으로 출력 형식을 설정할 수 있다. -->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>
                %d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n
            </Pattern>
        </layout>
    </appender>
    <!-- 콘솔 로깅 -->
    <appender name="DEMO" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                [%d{yyyy-MM-dd HH:mm:ss}:%-3relative][%thread] %-5level %logger{35} - %msg%n
            </pattern>
        </encoder>
    </appender>
    <!-- 파일 로깅 -->
    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <!-- 파일 경로 -->
        <file>${LOG_PATH}/file/app.log</file>
        <encoder>
            <pattern>
                [%d{yyyy-MM-dd HH:mm:ss}:%-3relative][%thread] %-5level %logger{35} - %msg%n
            </pattern>
        </encoder>
    </appender>
    <!-- 롤링 파일 로깅 -->
    <appender name="ROLLING" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/rolling/app.log</file>
        <encoder>
            <pattern>
                [%d{yyyy-MM-dd HH:mm:ss}:%-3relative][%thread] %-5level %logger{35} - %msg%n
            </pattern>
        </encoder>
        <!-- 롤링 설정 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 기간 및 크기가 다 되었을 때 롤링 파일이름 형식 -->
            <fileNamePattern>
                ${LOG_PATH}/rolling/app.%d{yyyy-MM-dd}.%i.log.zip
            </fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <!-- KR, MB, GB -->
                <maxFileSize>100KB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>2</maxHistory>
        </rollingPolicy>
    </appender>

    <!-- Logger에 패키지와 레벨을 설정하고 알맞는 Appender 를 설정해 준다. -->
    <logger name="com.example.demo.plus" level="DEBUG">
        <appender-ref ref="DEMO"/>
    </logger>
    <logger name="com.example.demo.minus" level="DEBUG">
        <appender-ref ref="FILE"/>
    </logger>
    <logger name="com.example.demo.multiply" level="DEBUG">
        <appender-ref ref="ROLLING"/>
    </logger>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```  

- `Logback-spring.xml` 은 크게 `Appender` 와 `Logger` 로 구성된다.
- `com.example.demo.plus` 패키지에서 `DEBUG` 레벨 이하의 로깅은 `DEMO Appender` 가 설정되어 있기 때문에 콘솔에 출력된다.	
- `com.example.demo.minus` 패키지에서 `DEBUG` 레벨 이하의 로깅은 `FILE Appender` 가 설정되었기 때문에 설정된 파일 경로에 에 남겨진다.
- `com.example.demo.multiply` 패키지에서 `DEBUG` 레벨 이하의 로깅은 `ROLLING Appender` 가 설정되었기 때문에 설정된 파일 경로에 남겨지고, Rolling 규칙에 따라 파일이 생성된다.

## 서비스 구현 코드

```java
public interface Calculate {
    int calculate(int a, int b);
}
```  

```java
public class MinusCalculateImpl implements Calculate {

    private static Logger logger = LoggerFactory.getLogger(MinusCalculateImpl.class);

    @Override
    public int calculate(int a, int b) {
        int result = a - b;

        logger.debug("minus : " + a + "-" + b + "=" + result);

        return result;
    }
}
```  

```java
public class MultiplyCalculateImpl implements Calculate {

    private static Logger logger = LoggerFactory.getLogger(MultiplyCalculateImpl.class);

    @Override
    public int calculate(int a, int b) {
        int result = a * b;

        logger.debug("multiply : " + a + "*" + b + "=" + result);

        return result;
    }
}
```  

```java
public class PlusCalculateImpl implements Calculate {

    private static Logger logger = LoggerFactory.getLogger(PlusCalculateImpl.class);

    @Override
    public int calculate(int a, int b) {
        int result = a + b;

        logger.debug("plus : " + a + "+" + b + "=" + result);

        return result;
    }
}
```  

## 테스트 코드

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class CalculateTest {
    private Calculate plusCalculate;
    private Calculate minusCalculate;
    private Calculate multiplyCalculate;
    
    @Before
    public void init() {
        this.plusCalculate = new PlusCalculateImpl();
        this.minusCalculate = new MinusCalculateImpl();
        this.multiplyCalculate = new MultiplyCalculateImpl();
    }

    @Test
    public void testPlus() {
        Assert.assertEquals(5, this.plusCalculate.calculate(3, 2));
    }

    @Test
    public void testMinus() {
        Assert.assertEquals(1, this.minusCalculate.calculate(3, 2));
    }

    @Test
    public void testMultiply() {
        Assert.assertEquals(6, this.multiplyCalculate.calculate(3, 2));
    }

    @Test
    public void testRolling_Debug() {
        for(int i = 1; i <= 2000; i++) {
            Assert.assertEquals(i * 2, this.multiplyCalculate.calculate(2, i));
        }
    }
}
```  

- `com.example.demo.plus`(PlusCalculateImpl) 의 `DEBUG` 레벨 이하 로그

	```
	[2019-07-13 08:44:20:3352][main] DEBUG c.e.demo.plus.PlusCalculateImpl - plus : 3+2=5
	```  
	
- `com.example.demo.minus`(MinusCalculateImpl) 의 `DEBUG` 레벨 이하 로그

	![그림 2]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-2.png)
	
	```
	[2019-07-13 08:53:42:3134][main] DEBUG c.e.demo.minus.MinusCalculateImpl - minus : 3-2=1
	```  
	
- `com.example.demo.multiply`(MultiplyCalculateImpl) 의 `DEBUG` 레벨 이하 로그
	
	![그림 3]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-3.png)
	
	```
	[2019-07-13 11:57:31:3157][main] DEBUG c.e.d.m.MultiplyCalculateImpl - multiply : 2*9=18
	```  

- `INFO` 레벨 이하 로그는 모두 콘솔에 출력된다.

	```
	11:57:31.660 [main] DEBUG c.e.d.multiply.MultiplyCalculateImpl - multiply : 2*1962=3924
	```  
	
## Filter
- Filter 기능을 추가하기 위해 `Logback-spring.xml` 에 `ERROR Appender` 와 `com.example.demo` 의 모든 `ERROR` 레벨 이하 로그에 대한 설정을 추가한다.

```xml
<!-- 롤링 파일 로깅 -->
<appender name="ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
        <level>error</level>
        <onMatch>ACCEPT</onMatch>
        <onMismatch>DENY</onMismatch>
    </filter>
    <file>${LOG_PATH}/error/error.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${LOG_PATH}/error/error.%d{yyyy-MM-dd}.%i.txt</fileNamePattern>
        <maxFileSize>100KB</maxFileSize>
        <maxHistory>3</maxHistory>
        <totalSizeCap>100MB</totalSizeCap>
    </rollingPolicy>
    <encoder>
        <pattern>
            %d{yyyy-MM-dd HH:mm:ss.SSS}[%-5level] : %msg%n
        </pattern>
    </encoder>
</appender>

<logger name="com.example.demo" level="ERROR">
    <appender-ref ref="ERROR"/>
</logger>
```  

- 구현된 서비스에 `ERROR` 로그를 남기기 위해 코드를 추가해 준다.

```java
@Override
public int calculate(int a, int b) {
    int result = a - b;

    if(result == 0) {
    	// minus, plus, multiply
        logger.error("minus is zero : " + a + "-" + b + "=" + result);
    } else {
    	// minus, plus, multiply
        logger.debug("minus : " + a + "-" + b + "=" + result);
    }

    return result;
}
```  

- `ERROR` 로깅 관련 테스트도 추가해 준다.

```java
@Test
public void testRolling_ERROR() {
    for(int i = 1; i <= 666; i++) {
        Assert.assertEquals(0, this.plusCalculate.calculate(0, 0));
        Assert.assertEquals(0, this.minusCalculate.calculate(i , i));
        Assert.assertEquals(0, this.multiplyCalculate.calculate(0, i));
    }
}
```  

- `com.example.demo` 패키지에서 `ERROR` 레벨 이하 로그

	![그림 4]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-4.png)
	
	```
	2019-07-13 12:42:36.454[ERROR] : multiply is zero : 0*4=0
	```  
	

## Logback 설정파일 리로딩
- Logback은 특정 주기로 설정파일을 리로딩하는 Dynamic Reloading 기능을 제공한다.
- 아래와 같이 `Logback-spring.xml` 파일에 작성해주면 30초 마다 해당 설정파일을 변경이 있을 경우 반영해준다.

	```xml
	<configuration scan="true" scanPeriod="30 seconds">
		<!-- ... -->
	</configuration>	
	```  
	
## 로깅 하위 패키지 설정
- 아래와 같이 logger 의 `additivity` 의 값을 false로 설정하면 해당 로거가 해당 패키지에만 적용되고, 하위 패키지에서는 적용되지 않는다.

	```xml
	<logger name="com.example.demo.plus" level="DEBUG" additivity="false">
	    <appender-ref ref="DEMO"/>
	</logger>
	```  

## Profile 에 따라 Logback 설정 구분하기
- `Logback-spring.xml` 파일에서 Profile 별로 설정을 구분하는 방법은 아래와 같이 `appender` 와 `logger` 를 `springProfile` 로 묶어 주면된다.

	```xml	
    <springProfile name="prod">
        <!-- 파일 로깅 -->
        <appender name="PROD_FILE" class="ch.qos.logback.core.FileAppender">
            <!-- 파일 경로 -->
            <file>${LOG_PATH}/prod/app.log</file>
            <encoder>
                <pattern>
                    [%d{yyyy-MM-dd HH:mm:ss}:%-3relative][%thread] %-5level %logger{35} - %msg%n
                </pattern>
            </encoder>
        </appender>

        <logger name="com.example.demo.minus" level="DEBUG">
            <appender-ref ref="PROD_FILE"/>
        </logger>
    </springProfile>
	```  
	
- `application.propertes` 에서 `spring.profiles.active=prod` profile 을 prod로 설정하고 `MinusCalculateImpl` 테스트 코드를 돌리면 아래와 같은 결과를 확인 할 수 있다.

	![그림 5]({{site.baseurl}}/img/spring/practice-springbootslf4jlogback-5.png)

	```
	2019-07-13 12:39:44:3130][main] DEBUG c.e.demo.minus.MinusCalculateImpl - minus : 3-2=1
	```  

---
## Reference
[The logback manual](https://logback.qos.ch/manual/index.html)   
[강력한 자바 오픈소스 로깅 프레임워크, logback 사용법 with example(스프링 부트에서 logback 가이드, logback-spring.xml 설정하기)](https://jeong-pro.tistory.com/154)   
[[Spring Boot #11] 스프링 부트 로깅( Spring Boot Logging )](https://engkimbs.tistory.com/767)   
[(Spring Boot)Logging과 Profile 전략](https://meetup.toast.com/posts/149)   
[Spring Boot SLF4j Logback example](https://www.mkyong.com/spring-boot/spring-boot-slf4j-logging-example/)   
