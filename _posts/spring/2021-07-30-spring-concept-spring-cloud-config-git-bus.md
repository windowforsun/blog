--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Cloud Config with Spring Cloud Bus"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Cloud Config 의 소개와 Git 사용한 활용방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring Cloud Config
    - Spring Cloud Bus
toc: true
use_math: true
---  

## Spring Cloud Bus
[Spring Cloud Config for Git Repository]({{site.baseurl}}{% link _posts/spring/2021-07-28-spring-concept-spring-cloud-config-git.md %})
에서(이전) `Git Repository` 를 외부 설정 저장소로 활용해서 `Spring Cloud Conig` 환경을 구성하는 법에 대해서 알아보았다. 
이때 한가지 문제점이 있었는데, 
모든 애플리케이션을 최신 설정파일로 적용하기 위해서는 `Spring Cloud Config Client` 의 수만큼 `/actuator/refresh` 를 호출 해 줘야 한다는 부분이다. 
대용량 서비스에서는 애플리케이션의 수가 비교적 많을 것이다. 
설정 파일 변경마다 `N` 번의 호출을 수행하는 것뿐만 아니라, 
각 애플리케이션마다 설정파일 동기화관련 이슈까지 있다면 오히려 `Spring Cloud Config` 를 사용하는 것이 큰 부담으로 다가올 것이다.  

위와 같은 문제점을 해결할 수 있는 방법이 바로 `Spring Coud Bus` 를 사용하는 것이다. 
`Spring Cloud Bus` 를 사용하면 `N` 번 호출을 하던 구조에서 `Spring Cloud Config Client` 에는 한번만 호출 해주게 되면, 
그 외 `Spring Cloud Config` 들은 설정 변경 통지를 받고 설정 파일을 갱신하는 구조이다.  

### Spring Cloud Bus 구조
아래 그림을 보면서 `Spring Cloud Bus` 를 사용한 구조와 방식에 대해서 이해해 보자.

![그림 1]({{site.baseurl}}/img/spring/concept-spring-cloud-config-git-bus-1.png)  

`Spring Cloud Bus` 는 다수개의 `Spring Cloud Config Client` 가 있더라도, 
갱신 요청을 한개의 클라이언트에만 하면 다른 클라이언트들도 모두 함께 설정파일이 갱신된다. 
이러한 동작이 가능할 수 있도록 `Spring Cloud Bus` 는 `Message Broker` 를 사용한다. 
모든 `Spring Cloud Config Client` 는 `Message Broker` 와 커넥션을 맺게 되고, 
한개의 클라이언트가 갱신 요청을 받게되면 해당 이벤트를 `Message Broker` 를 통해 다른 클라이언트들에게도 전파하는 방식이다.  

주로 사용하는 `Message Borker` 는 아래와 같다. 
1. `RabbitMQ`
1. `Kafka`
1. `Redis`

본 포스트에서는 `RabbitMQ` 와 `Kafka` 를 사용해서 `Spring Cloud Bus` 예제를 진행한다.  

### Spring Cloud Config Server
`Spring Cloud Config Server` 의 구성은 이전 포스트와 동일한 내용으로 사용한다. 
한가지 수행해줘야 할 작업은 `Docker Image` 를 빌드하는 작업이다.  

본 예제는 `Spring Boot 2.4` 버전을 기반으로 구성됐기 때문에 아래 명령을 사용해서 간단하게 `Docker Image` 를 빌드할 수 있다.  

```bash
.. 모듈사용 시 예시 ..
$ ./gradlew <모듈이름>:bootBuildImage --imageName=<빌드 이미지 이름>

.. 모둘 사용 x 예시 ..
$ ./gradlew bootBuildImage --imageName=<빌드 이미지 이름>

$ ./gradlew gitconfigserver:bootBuildImage --imageName=gitconfigserver
```  

위 마지막 명령어와 같이 `gitconfigserver` 라는 이름으로 이미지를 빌드한다.  

### Spring Cloud Config Client
`Spring Cloud Config Client` 는 몇가지 변경사항이 있으므로 설명대로 반영해 준다.  

앞서 언급했던 것처럼 `RabbitMQ`, `Kafka` 를 `Message Broker` 로 사용하는 두가지 경우에 대해서 예제를 진행할 예정이므로 
`Profile` 을 `rabbit`, `kafka` 와 같이 다르게 주어서 각 별도의 환경으로 구성할 예정이다. 

#### build.gradle
`Message Broker` 사용을 위해 필요한 `RabbitMQ`, `Kafka` 의존성을 추가해 준다. 
에제에서는 두가지 `Message Broker` 를 테스트하기 위해 모두 추가한 상황이고, 
실제 애플리케이션에서 적용할 때는 `RabbitMQ`, `Kafka` 중 필요한 의존성만 추가해주면 된다.

```groovy
dependencies {
	
    .. 생략 ..
	
	// rabbit
    implementation ('org.springframework.cloud:spring-cloud-starter-bus-amqp')
    //    kafka
    implementation ('org.springframework.cloud:spring-cloud-starter-bus-kafka')
	
    .. 생략 ..
}
```  

#### application.yaml

```yaml
server:
    port: 8071

management:
  security:
    enabled: false
  endpoints:
    web:
      exposure:
        # 변경
        include: busrefresh
```  

아래는 `RabbitMQ` 를 `Message Broker` 로 사용하는 `application-rabbit.yaml` 설정 내용이다. 

```yaml
spring:
  application:
    name: ${HOSTNAME}
  config:
    import: "optional:configserver:http://gitconfigserver:8070"
  cloud:
    stream:
      default-binder: rabbit
    bus:
      enabled: true
      refresh:
        enabled: true
    config:
      name: lowercase
      profile: dev

  rabbitmq:
    host: rabbitmq
    port: 5672
    username: guest
    password: guest
```  


아래는 `Kafka` 를 `Message Broker` 로 사용하는 `application-kafka.yaml` 설정 내용이다.  

```yaml
spring:
  application:
    name: ${HOSTNAME}
  config:
    import: "optional:configserver:http://gitconfigserver:8070"
  cloud:
    stream:
      kafka:
        binder:
          brokers: "kafka:9092"
      default-binder: kafka
    bus:
      enabled: true
      refresh:
        enabled: true
      trace:
        enabled: true
    config:
      name: lowercase
      profile: dev
```  

> 여기서 한가지 주의할 점이 있다. 
> 지금 `RabbitMQ`, `kafka` 설정 파일을 보면 모두 `spring.application.name` 프로퍼티를 모두 설정한 것을 확인 할 수 있다. 
> 만약 해당 프로퍼티를 설정하지 않는다면, 
> `Docker Swarm`, `Kubernetes` 와 같은 구조에서 동일한 포트를 갖는 서버 컨테이너를 구성하는 환경에서 설정 전파가 제대로 이뤄지지 않을 수 있다. 
> `spring.application.name`, `spring.application.id` 에 각 하나의 애플리케이션에 대한 고유한 값을 지정해 주거나, `server.port` 를 다르게 주어야 한다. 
> [참고](https://cloud.spring.io/spring-cloud-bus/reference/html/#service-id-must-be-unique)

#### Docker Image
`Spring Cloud Config Server` 와 동일하게 `Spring Cloud Config Client` 도 `Docker Image` 빌드가 필요하다. 

```bash
.. 모듈사용 시 예시 ..
$ ./gradlew <모듈이름>:bootBuildImage --imageName=<빌드 이미지 이름>

.. 모둘 사용 x 예시 ..
$ ./gradlew bootBuildImage --imageName=<빌드 이미지 이름>

$ ./gradlew gitbusconfigclient:bootBuildImage --imageName=gitbusconfigclient
```  

### RabbitMQ
`RabbitMQ` 를 사용해서 `Spring Cloud Bus` 를 구성하는 `docker-compose-rabbit.yaml` 파일 내용은 아래와 같다.  

```yaml
version: '3.7'

services:
  gitconfigserver:
    image: gitconfigserver:latest
    ports:
      - 9000:8070
    networks:
      - my-net

  rabbitmq:
    image: rabbitmq:3.7.5-management
    ports:
      - 15672:15672
      - 5672:5672
    networks:
      - my-net

  nginx:
    image: nginx:1.19
    volumes:
      - ./default.conf:/etc/nginx/nginx.conf
    ports:
      - 9001:80
    networks:
      - my-net
    depends_on:
      - gitbusconfiglcient-1

  gitbusconfiglcient-1:
    image: gitbusconfigclient:latest
    environment:
      - "SPRING_PROFILES_ACTIVE=rabbit"
    networks:
      - my-net
    depends_on:
      - rabbitmq
      - gitconfigserver
    scale: 2
    expose:
      - 8071

networks:
  my-net:
```  




### Kafka
`Kafka` 를 사용해서 `Spring Cloud Bus` 를 구성하는 `docker-compose-kafka.yaml` 파일 내용은 아래와 같다.

```yaml
version: '3.7'

services:
  gitconfigserver:
    image: gitconfigserver:latest
    ports:
      - 9000:8070
    networks:
      - my-net

  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
    networks:
      - my-net
	
  kafka:
    image: wurstmeister/kafka:2.12-2.5.0
    ports:
      - "9094:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 127.0.0.1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENERS: INTERNAL://kafka:9092,OUTSIDE://kafka:9094
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:9092,OUTSIDE://localhost:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,OUTSIDE:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - my-net
    depends_on:
      - zookeeper

  nginx:
    image: nginx:1.19
    volumes:
      - ./default.conf:/etc/nginx/nginx.conf
    ports:
      - 9001:80
    networks:
      - my-net
    depends_on:
      - gitbusconfiglcient-1

  gitbusconfiglcient-1:
    image: gitbusconfigclient:latest
    environment:
      - "SPRING_PROFILES_ACTIVE=kafka"
    networks:
      - my-net
    depends_on:
      - kafka
      - gitconfigserver
    scale: 2
    expose:
      - 8071

networks:
  my-net:
```  







---
## Reference
[Spring Cloud Config](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_quick_start)  
[Spring Cloud Config Server — Auto Refresh using Apache Kafka in Kubernetes](https://medium.com/@athulravindran/spring-cloud-config-server-auto-refresh-using-apache-kafka-in-kubernetes-86e3c427926e)  
[Spring Cloud Bus](https://www.baeldung.com/spring-cloud-bus)  
[Kafka access inside and outside docker](https://stackoverflow.com/questions/53247553/kafka-access-inside-and-outside-docker)  
[5. Service ID Must Be Unique](https://cloud.spring.io/spring-cloud-bus/reference/html/#service-id-must-be-unique)  
