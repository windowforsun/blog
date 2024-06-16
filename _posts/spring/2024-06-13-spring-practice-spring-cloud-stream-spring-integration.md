--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Stream with Spring Integration"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring 에서 발생하는 로그를 Logback 을 사용해 Logstash 에 전송해보자'
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

## Spring Cloud Stream with Spring Integration
`Spring Cloud Stream` 은 `MSA` 비동기 메시징 통신을 쉽게 구축할 수 있도록 하는 `Spring` 에서 제공하는 `Framework` 이다. 
`Kafka`, `RabbitMQ` 와 같은 메시지 브로커와 연동을 추상화해서, 
구체적인 세부 연동에 대한 신경을 쓰지 않더라도 보다 쉽게 연동하여 메시징 기반 애플리케이션을 구현 할 수 있도록 한다.  

- 메시지 발생, 구독을 위한 `High Level` 추상화 지원
- 메시지 브로커와 연결 간소화
- 함수형 프로그래밍 모델을 통한 메시지 처리 로직 표현

그리고 `Spring Integration` 은 `EIP(Enterprise Integration Pattern)` 의 구현에 초점을 둔 프로젝트이다. 
복잡한 메시지 처리, 라우팅, 변환 등을 지원하고 다양한 외부 시스템과 통합을 용이하게 한다.  

- 다양한 외부 시스템과의 통합 지원(HTTP, FTP, Database, ..)
- 메시지 채널, 필터, 변환 등 처리 유연성 제공

정리하면 `Spring Cloud Stream` 은 메시지 브로커와 연동 추상화를 통해 메시징 기반 애플리케이션 구현을 도와주고, 
`Spring Integration` 은 다양한하고 복잡한 메시지 처리를 용이하게 하는 서로 다른 특징을 가지고 있다. 
그래서 이 2가지를 융합해서 사용하는 방법에 대해 알아보고자 한다. 

이를 함께 사용하면 `Spring Integration` 을 통해 복잡한 통합 시나리오를 구현하고, 
`Spring Cloud Stream` 을 사용해 메시지 브로커와 연결, 전송을 쉽게 구성할 수 있다. 
그리고 메시지 브로커와의 세부 사항은 `Spring Cloud Stream` 이 추상화 해주고, 
메시지를 처리하는 비지니스 로직은 `Spring Integration` 을 통해 추상화 할 수 있을 것이다.  

본 포스팅에서는 복잡한 메시지 처리에 대한 내용은 다루지 않고, 
간단한 `Spring Cloud Stream` 의 `Source`, `Processor`, `Sink` 애플리케이션을 구성하는 방법에 대해 알아볼 것이다. 

공통으로 사용하는 `build.gradle` 내용은 아래와 같다.  

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.6.4'
    id 'com.google.cloud.tools.jib' version '3.2.0'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'
sourceCompatibility = '11'
version 'v1'

ext {
    springCloudVersion = '2021.0.1'
}
repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-stream'
    implementation 'org.springframework.cloud:spring-cloud-starter-stream-kafka'
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    compileOnly "org.projectlombok:lombok"
    annotationProcessor "org.projectlombok:lombok"
    annotationProcessor "org.springframework.boot:spring-boot-configuration-processor"

    // spring cloud stream test
    testImplementation("org.springframework.cloud:spring-cloud-stream") {
        artifact {
            name = "spring-cloud-stream"
            extension = "jar"
            type ="test-jar"
            classifier = "test-binder"
        }
    }
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

test {
    useJUnitPlatform()
}

// docker 이미지 빌드 시
jib {
    from {
        image = "openjdk:11-jre-slim"
        // for mac m1
        platforms {
//            platform {
//                architecture = "arm64"
//                os = "linux"
//            }
            platform {
                architecture = "amd64"
                os = "linux"
            }
        }
    }
    to {
        image = "<이미지 이름>"
        tags = ["${project.version}".toString()]
    }
    container {
        mainClass = "<메인 클래스>"
        ports = ["8080"]
    }

}
```


### Source Application

- `ExamSourceApplication`

```java
@SpringBootApplication
public class ExamSourceApplication {
    public static void main(String... args) {
        SpringApplication.run(ExamSourceApplication.class, args);
    }
}
```  

- `ExamSource`

```java
@Slf4j
@Configuration
public class ExamSource {
    private static final AtomicInteger COUNTER = new AtomicInteger();
    
    @Bean
    public IntegrationFlow sourceFlow(StreamBridge streamBridge) {
        return IntegrationFlows.fromSupplier(
                        // 메시지 생성
                        () -> "test message " + COUNTER.incrementAndGet(),
                        // 1초 마다 생성
                        e -> e.poller(Pollers.fixedRate(1000))
                )
                .handle(String.class, (payload, headers) -> {
                    // sourceOutput 은 이후 application.yaml 설정과 매칭 필요
                    // 1초마다 생성되는 메시지를 sourceOutput 이라는 OutputBinding 으로 전송
                    streamBridge.send("sourceOutput", MessageBuilder.withPayload(payload).build());
                    return null;
                })
                .get();
    }
}
```  

- `application.yaml`

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # kafka 브로커 주소
          brokers: localhost:9092
      # 바인딩 정의
      bindings:
        # 출력 바인딩 정의
        sourceOutput:
          # 목적지는 연결되는 Processor/Sink 애플리케이션의 input 과 매칭
          destination: input
```  

- `ExamSourceTest`

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class ExamSourceTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private CompositeMessageConverter converter;

    @Test
    public void test() {
        // sourceFlow -> sourceOutput -> input (대략적으로 표현한다면..)
        // 생성되는 메시지 수신
        Message<byte[]> message = this.outputDestination.receive(5000, "input");
        String strMessage = (String) this.converter.fromMessage(message, String.class);

        assertThat(strMessage, is("test message 1"));

        message = this.outputDestination.receive(5000, "input");
        strMessage = (String) this.converter.fromMessage(message, String.class);

        assertThat(strMessage, is("test message 2"));
    }
}
```  
