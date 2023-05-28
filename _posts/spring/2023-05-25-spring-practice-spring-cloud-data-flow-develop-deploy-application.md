--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Data Flow(SCDF) Kubernetes 테스트 환경 구축"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Cloud Data Flow(SCDF) 를 Kubernetes(kubectl) 환경에서 구성해보자'
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
    - Kubernetes
    - kubectl
toc: true
use_math: true
---  

## Spring Cloud Stream Application
`SCDF` 에 배포하고 사용할 수 있는 간단한 `Spring Cloud Stream Application` 을 구성해본다.
예제는 [Spring Cloud Stream Source Processor Sink Application]({{site.baseurl}}{% link _posts/spring/2023-04-29-spring-practice-spring-cloud-stream-source-processor-sink-application.md %})
에서 약간 변경해 사용한다. 
위 예제는 `Stream Application` 에서 직접 사용할 토픽을 명시해서 `Stream Application` 간 연결성을 만들었다면, 
이번 예제에서는 `SCDF` 에서 사용하는 `Stream Name` 을 기반으로 `Stream Application` 에서 사용하는 토픽이 생성될 수 수정해 준다.  

그리고 `SCDF` 에서 `Stream Application` 에서 사용할 수 있는 `Properties` 를 선언해 배포때마다 필요한 설정 값을 주입하는 방법에 대해서도 알아본다. 
위 구현에 필요한 아래 내용을 `build.gradle` 에 모두 동일하게 추가 필요하다.  

```groovy
annotationProcessor "org.springframework.boot:spring-boot-configuration-processor"
```  

`Stream Application` 에서 사용하는 2가지 모델은 아래와 같이 수정이 필요하다.  

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DataModel {
    private Long uid;
    private String name; // 추가
    private String dataLevel;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DataPriorityModel {
    private Long uid;
    private String name; // 추가
    private String dataLevel;
    private Long priority;
}

```  

### Source Application
[이전 Source Application](https://windowforsun.github.io/blog/spring/spring-practice-spring-cloud-stream-source-processor-sink-application/#source-application)
에서 먼저 `application.yaml` 을 아래와 같이 수정해 `Stream Application` 간 토픽 바인딩이 `SCDF` 기반으로 수행 할 수 있도록 한다.  

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # kafka broker 주소
          brokers: localhost:9092
      function:
        bindings:
          # source 로 선언된 빈이름을 기반으로 stream out 의 이름인 <빈이름>-<out>-<index> 에서 리턴하는 데이터는 output 으로 바인딩
          sendDataModel-out-0: output
      bindings:
        output:
          # output 과 바인딩된 곳에서 리턴하는 데이터는 data-model 목적지로 전달
          destination: data-model
```  

`MySourceProperties` 라는 클래스를 생성해 사용할 프로퍼티를 선언해준다. 
해당 프로퍼티는 `my-source.name` 이라는 이름을 사용해서 값을 설정할 수 있다.  

```java
@Data
@ConfigurationProperties("my-source")
public class MySourceProperties {
    private String name;
}
```  

`Stream Application` 의 시작점인 `DataSourceApplication` 에 
구성한 프로퍼티를 활성화하는 어노테이션을 작성해준다.  

```java
// 추가
@EnableConfigurationProperties(MySourceProperties.class)
@SpringBootApplication
public class DataSourceApplication {
    public static void main(String... args) {
        SpringApplication.run(DataSourceApplication.class, args);
    }
}
```  

메시지 생성 역할을 하는 `DataSender` 에 `MySourceProperties` 의 값을 사용해서 
`DataModel` 에 `name` 을 설정하는 코드로 수정한다.  

```java
@Slf4j
@Configuration
@RequiredArgsConstructor
public class DataSender {
    private final MySourceProperties myTestProperties;

    @Bean
    public Supplier<DataModel> sendDataModel() {
        AtomicLong atomicLong = new AtomicLong();
        Random random = new Random();
        return () -> {
            DataModel dataModel =  DataModel.builder()
                    .uid(atomicLong.getAndIncrement())
                    // 추가
                    .name(this.myTestProperties.getName())
                    .dataLevel(Character.toString('A' + random.nextInt(('Z' - 'A') + 1)))
                    .build();

            log.info("{}", dataModel);
            return dataModel;
        };
    }
}
```  

테스트 코드는 아래와 같이 작성할 수 있다.  

```java
@SpringBootTest(properties = "my-source.name=testName")
@Import(TestChannelBinderConfiguration.class)
public class DataSourceApplicationTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private CompositeMessageConverter converter;
    @Autowired
    private DataSender dataSender;

    @Test
    public void testSendDataModel() {
        this.dataSender.sendDataModel();

        Message<byte[]> result = this.outputDestination.receive(10000, "data-model");
        DataModel dataModel = (DataModel) this.converter.fromMessage(result, DataModel.class);

        assertThat(dataModel.getUid()).isGreaterThanOrEqualTo(0L);
        assertThat(dataModel.getName()).isNotEmpty().isNotBlank();
        assertThat(dataModel.getDataLevel()).isBetween("A", "Z");
    }
}
```  

### Processor Application
[이전 Processor Application](https://windowforsun.github.io/blog/spring/spring-practice-spring-cloud-stream-source-processor-sink-application/#processor-application)
의 구성을 기반으로 `application.yaml` 을 `SCDF` 기반으로 사용할 수 있도록 수정한다.  

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # kafka broker 주소
          brokers: localhost:9092
      function:
        bindings:
          # processor 로 선언된 빈이름을 기반으로 stream in 의 이름인 <빈이름>-<in>-<index> 에서 사용할 데이터는 input 으로 바인딩
          processPriority-in-0: input
          # processor 로 선언된 빈이름을 기반으로 stream out 의 이름인 <빈이름>-<out>-<index> 에서 리턴하는 데이터는 output 으로 바인딩
          processPriority-out-0: output
      bindings:
        input:
          # input 과 바인딩된 곳에 data-model 목적지에서 전달되는 데이터 전달
          destination: data-model
        output:
          # output 과 바인딩된 곳에서 리턴하는 데이터는 data-priority 목적지로 전달
          destination: data-priority
```  

`Source Application` 의 메시지를 받아 `Processor` 동작을 수행하는 `DataProcessor` 에서 
`name` 필드를 설정하는 코드만 추가한다.  

```java
@Slf4j
@Configuration
public class DataProcessor {
    @Bean
    public Function<DataModel, DataPriorityModel> processPriority() {
        return dataModel -> {
            DataPriorityModel dataPriorityModel = DataPriorityModel.builder()
                    .uid(dataModel.getUid())
                    // 추가
                    .name(dataModel.getName())
                    .dataLevel(dataModel.getDataLevel())
                    // (Z - dataLevel + 1) * uid
                    .priority(('Z' - dataModel.getDataLevel().charAt(0) + 1) * dataModel.getUid())
                    .build();

            log.info("{}", dataPriorityModel);

            return dataPriorityModel;
        };
    }
}
```  

테스트 코드는 아래와 같이 작성할 수 있다.  

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class DataProcessorApplicationTest {
    @Autowired
    private OutputDestination outputDestination;
    @Autowired
    private InputDestination inputDestination;
    @Autowired
    private CompositeMessageConverter converter;

    @Test
    public void testProcessPriority() {
        DataModel dataModel = DataModel.builder()
                .uid(10L)
                .name("testName")
                .dataLevel("E")
                .build();
        Map<String, Object> headers = new HashMap<>();
        headers.put("contentType", "application/json");
        MessageHeaders messageHeaders = new MessageHeaders(headers);
        Message<?> sourceMessage = this.converter.toMessage(dataModel, messageHeaders);

        inputDestination.send(sourceMessage, "data-model");

        Message<byte[]> processMessage = this.outputDestination.receive(10000, "data-priority");
        DataPriorityModel dataPriorityModel = (DataPriorityModel) this.converter
                .fromMessage(processMessage, DataPriorityModel.class);

        assertThat(dataPriorityModel.getUid()).isEqualTo(dataModel.getUid());
        assertThat(dataPriorityModel.getName()).isNotEmpty().isNotBlank();
        assertThat(dataPriorityModel.getDataLevel()).isEqualTo(dataModel.getDataLevel());
        assertThat(dataPriorityModel.getPriority()).isEqualTo(220L);
    }
}
```  

### Sink Application
[이전 Sink Application](https://windowforsun.github.io/blog/spring/spring-practice-spring-cloud-stream-source-processor-sink-application/#sink-application)
의 구성을 기반으로 `application.yaml` 을 `SCDF` 기반으로 사용할 수 있도록 수정한다.  

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # kafka broker 주소
          brokers: localhost:9092
      function:
        bindings:
          # sink 로 선언된 빈이름을 기반으로 stream in 의 이름인 <빈이름>-<in>-<index> 에서 사용할 데이터는 input 으로 바인딩
          logData-in-0: input
      bindings:
        input:
          # input 과 바인딩된 곳에 data-priority 목적지에서 전달되는 데이터 전달
          destination: data-priority
```  

테스트는 아래와 같이 진행할 수 있다.  

```java
@SpringBootTest
@Import(TestChannelBinderConfiguration.class)
public class DataSinkLogApplicationTest {
    @Autowired
    private InputDestination inputDestination;
    @Autowired
    private CompositeMessageConverter converter;

    @Test
    public void testSinkLog() {
        DataPriorityModel input = DataPriorityModel.builder()
                .uid(10L)
                .name("testName")
                .dataLevel("E")
                .priority(220L)
                .build();
        Map<String, Object> headers = new HashMap<>();
        headers.put("contentType", "application/json");
        MessageHeaders messageHeaders = new MessageHeaders(headers);
        Message<?> inputMessage = this.converter.toMessage(input, messageHeaders);

        inputDestination.send(inputMessage, "data-priority");
        // check output log
    }
}
```  

### SCDF 기반 스트림 배포
[Kubernetes SCDF 구성]({{site.baseurl}}{% link _posts/spring/2023-05-11-spring-practice-spring-cloud-data-flow-installation.md %})
에서 구성한 `SCDF` 를 사용해서 구현한 애플리케이션들을 `SCDF` 에 배포해서 스트림을 만들어본다.  

`SCDF` 에서 사용하기 위해서는 외부에서 접근할 수 있는 `Maven` 혹은 `Docker` 저장소에 스트림 애플리케이션이 올라가 있어야 한다. 
예제에서는 `Docker` 빌드 후 `Docker Hub` 에 업로드해서 `SCDF` 에서 사용할 수 있도록 한다.  

앞서 구현한 애플리케이션들을 `Docker Image` 로 빌드하기 위해 모든 애플리케이션 `build.gradle` 에 플러그인과 빌드 설정을 추가해 준다.  

```groovy
plugins {
    id 'com.google.cloud.tools.jib' version '3.2.0'
}

.. 생략 ..

version 'test'

jib {
    from {
        image = "openjdk:11-jre-slim"
        platforms {
            // for mac m1
            platform {
                architecture = "arm64"
                os = "linux"
            }
            // or others
            // platform {
            //     architecture = "amd64"
            //     os = "linux"
            // }
        }
    }
    to {
        image = "data-source"
        // or
        // image = "data-processor"
        // or
        // image = "data-sink-log"
        tags = ["${project.version}".toString()]
    }
    container {
        mainClass = "com.windowforsun.dataflow.datasource.DataSourceApplication"
        // or
        // mainClass = "com.windowforsun.dataflow.dataprocessor.DataProcessorApplication"
        // or
        // mainClass = "com.windowforsun.dataflow.datasinklog.DataSinkLogApplication"
        ports = ["8080"]
    }

}
```  

아래 명령을 사용해서 3개 애플리케이션을 모두 `Docker Image` 로 빌드한다.  

```bash
.. 각 애플리케이션은 한 프로젝트에 하위 모듈로 구성된 상태이다. ..

$ ./gradlew data-source:jibDockerBuild

$ ./gradlew data-processor:jibDockerBuild

$ ./gradlew data-sink-log:jibDockerBuild

```  

생성된 이미지를 확인하고, `Docker Hub` 에 푸시할 수 있도록 태그를 수정한 뒤 업로드한다.  

```bash
$ docker image ls
REPOSITORY                                      TAG                    IMAGE ID       CREATED         SIZE
data-source                                     test                   cfa69ea32b9b   53 years ago    257MB
data-processor-amd                              test                   953d433f3e1b   53 years ago    257MB
data-sink-log-amd                               test                   4999966cb050   53 years ago    264MB

$ docker image tag data-source:test windowforsun/data-source:test
$ docker image tag data-processor:test windowforsun/data-processor:test
$ docker image tag data-sink-log:test windowforsun/data-sink-log:test

$ docker push windowforsun/data-source:test
$ docker push windowforsun/data-processor:test
$ docker push windowforsun/data-sink-log:test
```  

`SCDF` 대시보드에 접속해서 업로드한 3개 애플리케이션을 추가한다.  

spring-cloud-data-flow-develop-deploy-application-1.png

추가를 완료하면 아래와 같이 3개 애플리케이션이 추가된 것을 확인 할 수 있다.  

spring-cloud-data-flow-develop-deploy-application-2.png

아래와 같이 스트림을 구성한 뒤 `data-stream-test` 라는 이름으로 생성한다. 
별도 프로퍼티 설정은 해당 단계에서는 하지 않는다. 

spring-cloud-data-flow-develop-deploy-application-3.png

spring-cloud-data-flow-develop-deploy-application-4.png

생성된 `data-stream-test` 에 들어가 `Deploy Stream` 을 눌러 배포 화면으로 이동한다. 
그리고 `Freetext` 를 사용해서 아래 프로퍼티를 사용해 배포를 수행한다.  

```properties
deployer.*.kubernetes.limits.cpu=2
deployer.*.kubernetes.limits.memory=1000Mi
deployer.*.kubernetes.image-pull-policy=Always
# 각 애플리케이션의 프로퍼티는 app.<앱 이름>.<프로퍼티이름> 을 사용해서 설정할 수 있다. 
app.data-source.my-source.name=scdf-stream-test
```  

배포가 정상적으로 완료되면 아래와 같은 화면을 확인 할 수 있다.  

spring-cloud-data-flow-develop-deploy-application-5.png

위 화면에서 `RUNTIME` 부분에 각 애플리케이션의 `VIEW LOG` 를 클릭하면, 
애플리케이션에서 출력되는 로그를 통해 전체 스트림이 정상적으로 동작하는 것을 확인 할 수 있다. 
대표적으로 `Sink` 만 확인하면 아래와 같다.  

spring-cloud-data-flow-develop-deploy-application-6.png

현재 스트림에서 사용하는 `Kafka` 에 접속해 생성된 토픽을 확인하면 아래와 같다.  

```bash
$ kubectl exec -it kafka-deployment-66556d4d99-l4s58 -- kafka-topics --bootstrap-server localhost:29092 --list
data-stream-test.data-processor
data-stream-test.data-source
```  

`Stream Application` 에서 `application.yaml` 에서 설정한 내용을 바탕으로 애플리케이션간 관계를 그려보면 아래와 같다.  

```
(DataSource) - <output> - [data-model] - <input> - (DataProcessor) - <output> - [data-priority] - <input> - (DataSinkLog)
```  

`application.yaml` 에서 설정한 내용은 실질적으로 `Stream Application` 들간의 관계만 명시하기 위함이고, 
실질적으로 사용되는 토픽이름은 아래와 같은 규칙으로 생성된다. 

```
<SCDF 에서 사용한 Stream 이름>.<스트림 애플리케이션 이름>
```  

그래서 최종적으로 실제 사용하는 토픽이름으로 관계를 그려보면 아래와 같다.  

```
(DataSource) - <output> - [data-stream-test.data-source] - <input> - (DataProcessor) - <output> - [data-stream-test.data-processor] - <input> - (DataSinkLog)
```  

---  
## Reference
[Spring Cloud Data Flow Installation Kubectl](https://dataflow.spring.io/docs/installation/kubernetes/kubectl/)
