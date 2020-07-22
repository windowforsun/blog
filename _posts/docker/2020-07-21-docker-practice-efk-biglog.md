--- 
layout: single
classes: wide
title: "[Docker 실습] EFK 모니터링에서 큰 로그 처리하기"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'EFK 로그 모니터링을 할때 큰 로그를 처리해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
  - Docker
  - EFK
  - concat
  - Fluentd
toc: true
use_math: true
---  


## EFK 로그 모니터링 구성
먼저 간단하게 `EFK` 를 활용한 애플리케이션 로그 모니터링을 구성해본다. 
애플리케이션은 `Spring` 으로 구성하고, 전체 시스템은 `Docker` 로 구성한다. 

### 애플리케이션
애플리케이션이 수행하는 동작은 `/strlog/<크기>` 로 `HTTP GET` 요청을 보내면, 
크기에 해당하는 로그를 생성하는 간단한 애플리케이션이다. 

- 프로젝트 구조

```
.
├── build.gradle
├── gradlew
├── gradlew.bat
├── settings.gradle
└── src
    └── main
        ├── generated
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── efk
        │               ├── EfkApplication.java
        │               └── TestController.java
        └── resources
            └── application.yaml
```  

- `build.gradle`

    ```groovy
    import java.time.Instant
    
    plugins {
        id 'org.springframework.boot' version '2.3.1.RELEASE'
        id 'io.spring.dependency-management' version '1.0.9.RELEASE'
        id 'java'
        id 'com.google.cloud.tools.jib' version '1.6.0'
    }
    
    group = 'com.windowforsun'
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '1.8'
    
    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }
    
    repositories {
        mavenCentral()
    }
    
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-web'
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
            exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
    }
    
    test {
        useJUnitPlatform()
    }
    
    jib {
        from {
            image = "openjdk:8-jre-alpine"
        }
        to {
            image = "efk-spring"
        }
        container {
            mainClass = "com.windowforsun.efk.EfkApplication"
            ports = ["8080"]
            creationTime = Instant.now()
        }
    }
    ```  
  
    - 빌드 툴로는 `Gradle` 을 사용한다. 
    - `Docker` 이미지 빌드를 위해서는 `jib` 플러그인을 사용한다. 
    - 빌드된 `Docker` 이미지의 이름은 `efk-spring` 이다. 
    
- `application.yaml`

    ```yaml
    logging:
      level:
        com.windowforsun.efk: [info]
    ```  
  
    - 로그레벨은 `info` 만 사용한다. 
    
- `EfkApplication.java`

    ```java
    @SpringBootApplication
    public class EfkApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(EfkApplication.class, args);
        }
    
    }
    ```  
  
- `TestController.java`

    ```java
    @Slf4j
    @RestController
    public class TestController {
    
        @GetMapping("/strlog/{size}")
        public String strLogging(@PathVariable float size) throws Exception {
            int strSize = (int)(size * 1000);
            StringBuilder builder = new StringBuilder();
            String baseStr = "0123456789";
            strSize /= baseStr.length();
    
            builder.append("~~~start~~~");
            for(int i = 0; i < strSize; i++) {
                builder.append(baseStr);
            }
            builder.append("~~~end~~~");
    
            log.info(builder.toString());
    
            return size + "KB";
        }
    }
    ```  
  
    - `/strlog/{size}` 경로는 `HTTP GET` 요청이 오면 함께 전달된, `{size}` 의 값 만큼의 크기로 로그를 생성하고 생성된 크기를 문자열로 반환한다. 
    - `{size}` 는 1000 글자 단위로, 1 일 경우 1000자 로그가 생성된다.
    - 생성되는 로그의 시작에는 `~~start~~` 라는 글자 접두사로 붙고, 로그 끝에는 `~~end~~` 라는 글자가 접미사로 붙는다.
  

### Docker 구성

```
.
├── docker-compose.yaml
├── ela-volume
└── fluentd
    ├── Dockerfile
    └── conf
        └── fluent.conf

```  

`Docker` 를 사용해서 `EFK` 구성에 필요한 파일은 위와 같다. 

- `docker-compose.yaml`

    ```yaml
    version: '3.7'
    
    services:
      efk-spring:
        image: efk-spring:latest
        ports:
          - "8080:8080"
        networks:
          - test-efk-net
        depends_on:
          - efk-fluentd
        logging:
          driver: "fluentd"
          options:
            fluentd-address: localhost:24225
            tag: web.log
    
      efk-fluentd:
        build: ./fluentd
        networks:
          - test-efk-net
        volumes:
          - ./fluentd/conf:/fluentd/etc
        ports:
          - "24225:24225"
          - "24225:24225/udp"
    
      efk-elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.8.0
        networks:
          - test-efk-net
        ports:
          - 9200:9200
        volumes:
          - ./ela-volume/data:/usr/share/elasticsearch/data
          - ./ela-volume/logs:/usr/share/elasticsearch/logs
        environment:
          - node.name=efk-elasticsearch
          - cluster.initial_master_nodes=efk-elasticsearch
    
      efk-kibana:
        image: docker.elastic.co/kibana/kibana:7.8.0
        networks:
          - test-efk-net
        ports:
          - 5601:5601
        depends_on:
          - efk-elasticsearch
        environment:
          ELASTICSEARCH_URL: http://efk-elasticsearch:9200
          ELASTICSEARCH_HOSTS: http://efk-elasticsearch:9200
    
    networks:
      test-efk-net:
    ```  
  
    - 구성된 서비스들이 사용하는 네트워크는 `test-efk-net` 이다. 
    - `efk-spring` 에서 `logging` 필드를 사용해서 `fluentd` 로 로그를 전송하는 설정을 한다. 
    - 애플리케이션 서비스는 프로젝트에서 빌드한 이미지를 사용한다. 
    - 애플리케이션 로그는 `web.log` 라는 태그와 함께 남겨진다. 
    - `Elasticsearch` 와 `Kibana` 의 구성은 [여기](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html)
    를 참고하면 더욱 자세한 내용을 확인 할 수 있다. 
    - `fluentd` 는 `Dockerfile` 을 통해 이미지를 빌드해서 사용하고, 설정파일을 마운트한다. 
    
- `fluentd/Dockerfile`

    ```dockerfile
    FROM fluent/fluentd:v1.7
    USER root
    RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
    RUN ["gem", "install", "fluent-plugin-rewrite-tag-filter"]
    USER fluent
    ```  
  
    - `fluentd` 공식 이미지를 사용하고 필요한 플러그인을 설치해서 사용할 이미지를 구성한다. 
    
- `fluentd/conf/fluent.conf`

    ```
    <source>
        @type forward
        port 24225
        bind 0.0.0.0
    </source>
    
    <match web.**>
        @type copy
    
        # elaticsearch 로 해당 로그를 전송한다.
        <store>
          @type elasticsearch
          # elasticsearch 호스트 설정
          host efk-elasticsearch
          port 9200
          logstash_format true
          logstash_prefix fluentd
          logstash_dateformat %Y%m%d
          include_tag_key true
          type_name access_log
          tag_key @log_name
          flush_interval 1s
        </store>
    
        # 표준 출력으로 출력한다.
        <store>
            @type stdout
        </store>
    </match>
    ```  
  
    - `Spring` 애플리케이션에서 전송된 로그를 `fluentd` 에 설정된 처리에 따라 수행하고, 
    마지막에 `Elasticsearch` 로 전송과 `stdout` 으로 출력을 수행한다. 
    
`docker-compose up --build` 명령어를 사용해서 전체 구성을 한번에 올릴 수 있다. 

>만약 `Elasticsearch` 실행 과정에서 `exited with code 78` 로 종료 된다면 `vm` 옵션에 대한 설정이 필요하다. 
>먼저 `sysctl -a | grep vm.max_map_count` 명령으로 현재 설정된 값을 확인한다. 
>그리고 `sysctl -w vm.max_map_count=524288` 명령으로 값을 수정해 준다. 
>시스템에 여유가 있다면 더 큰 값도 가능하다.

```bash
$ docker-compose up --build

.. 생략 ..

Successfully built 759801b3b6f8
Successfully tagged efk-spring_efk-fluentd:latest
Starting efk-spring_efk-fluentd_1       ... done
Starting efk-spring_efk-elasticsearch_1 ... done                                                                                    Starting efk-spring_efk-spring_1        ... done
Starting efk-spring_efk-kibana_1        ... done

.. 생략 ..
```  

요청은 `curl localhost:8080/strlog/1` 명령어로 보낼 수 있다. 

```bash
curl localhost:8080/strlog/1
1.0KB
```  

요청 후 `efk-fluentd` 로그를 확인하면 아래와 같다. 
`efk-fluentd` 로그 확인은 `docker-compose.yaml` 경로에서 `docker-compose logs efk-fluentd` 명령으로 가능하다. 

```
efk-fluentd_1        | 2020-07-21 20:22:43.000000000 +0000 web.info.log: {"container_id":"e3b23c06732244
4c1a3e13739793015a5293e7779317c3ab2e5dc9a267a23273","container_name":"/efk-spring_efk-spring_1","source"
:"stdout","log":"2020-07-21 20:22:43.036  INFO 1 --- [nio-8080-exec-1] com.windowforsun.efk.TestControll
er      : ~~~start~~~01234567890123456789012345678901234567890123456789012345678901234567890123456789012
34567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456
78901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890
12345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234
56789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678
90123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012
34567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456
78901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890
12345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234
5678901234567890123456789012345678901234567890123456789012345678901234567890123456789~~~end~~~"}
```  

브라우저에서 `http://localhost:5601` 로 접속하면 `Kibana` 에 접속해서도 해당 로그 확인이 가능하다. 


## 큰 로그
크기가 크지 않는 로그는 위 구성으로 애플리케이션에서 `fluentd` 로 전송하고, 
`fluentd` 에서 로그 처리를 한 후 `Elasticsearch` 로 전송해서 저장한 후 `Kibana` 에서 모니터링이 가능하다.  

하지만 `Docker` 로그는 `16KB` 가 넘어가는 로그는 나눠서 `stdout` 으로 출력된다. 
실제로 그러한지 `/strlog/20` 으로 요청을 보내 `16KB` 가 넘는 로그를 남겨 본다.  

```bash
efk-fluentd_1        | 2020-07-21 20:34:54.000000000 +0000 web.log: {"partial_id":"cdb906ac6ceb2e2690c17
bf79c4009828425fb32d93c24e650e717a5e0d3a092","partial_ordinal":"1","partial_last":"false","container_id"
:"e3b23c067322444c1a3e13739793015a5293e7779317c3ab2e5dc9a267a23273","container_name":"/efk-spring_efk-sp
ring_1","source":"stdout","log":"2020-07-21 20:34:54.766  INFO 1 --- [nio-8080-exec-6] com.windowforsun.
efk.TestController      : ~~~start~~~0123456789012345678901234567890123456789012345678901234567890123456

.. 생략 ..

789012345678901234567890123456789012345667812345678901234567890123456789012345","partial_message":"true"}
efk-fluentd_1        | 2020-07-21 20:34:54.000000000 +0000 web.log: {"partial_id":"cdb906ac6ceb2e2690c17b
f79c4009828425fb32d93c24e650e717a5e0d3a092","partial_ordinal":"2","partial_last":"true","container_id":"e
3b23c067322444c1a3e13739793015a5293e7779317c3ab2e5dc9a267a23273","container_name":"/efk-spring_efk-spring
_1","source":"stdout","log":"6789012345678901234567890123456789012345678901234567890123456789012345678901

.. 생략 ..

123456789012345678901234567890123456789012345678901234567890123456789~~~end~~~","partial_message":"true"}
```  

`efk-fluentd` 에서 2개의 로그를 남겼는데, 
`partial_id` 의 값이 두 로그 모두 같다. 
그리고 기존에는 존재하지 않았던 `partial_message`, `partial_ordinal`, `partial_last` 등의 필드가 추가 되었다. 
위와 같은 큰 로그는 `Kibana` 에서도 동일하게 하나의 로그가 2개로 나눠져 남게 된다.  

### concat
`fluentd` 에는 많고 다양한 플러그인이 존재한다. 
그래서 이러한 문제도 플러그인을 적절하게 사용하면, 
나눠진 로그를 하나의 로그로 합쳐서 `Elasticsearch` 에 보내 `Kinaba` 에서 모니터링을 할 수 있다.  

사용할 플러그인은 [fluent-plugin-concat](https://github.com/fluent-plugins-nursery/fluent-plugin-concat) 
라는 플로그인이다. 말 그대로 여러 줄로 출력된 로그를 한줄로 합치는 플러그인이다.  

기존 `fluentd/Dockerfile` 과 설정파일(`fluent.conf`)을 수정해 해당 플러그인을 적용하고 관련설정을 추가한다.  

먼저 `Dockerfile` 에 해당 플러그인을 설치하는 스크립트를 아래와 같이 추가한다. 

```dockerfile
FROM fluent/fluentd:v1.7
USER root
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
RUN ["gem", "install", "fluent-plugin-rewrite-tag-filter"]
# 플러그인 설치 스크립트 추가
RUN ["gem", "install", "fluent-plugin-concat"]
USER fluent
```  

그리고 `fluent.conf` 파일은 앞서 공유한 `Github` 레포에서 관련 문서를 확인할 수 있다. 
현재 `Docker` 버전이 `19.03.8` 이므로 아래 설정을 사용한다. 

```
<filter>
  @type concat
  key log
  use_partial_metadata true
  separator ""
</filter>
```  

실제로 적용한 `fluent.conf` 파일의 내용은 아래와 같다. 

```
<source>
    @type forward
    port 24225
    bind 0.0.0.0
</source>

## 추가한 플러그인 관련 설정
<filter **>
    @type concat
    key log
    use_first_timestamp true
    use_partial_metadata true
    separator ""
</filter>

<match web.**>
    @type copy

    # elaticsearch 로 해당 로그를 전송한다.
    <store>
      @type elasticsearch
      # elasticsearch 호스트 설정
      host efk-elasticsearch
      port 9200
      logstash_format true
      logstash_prefix fluentd
      logstash_dateformat %Y%m%d
      include_tag_key true
      type_name access_log
      tag_key @log_name
      flush_interval 1s
    </store>

    # 표준 출력으로 출력한다.
    <store>
        @type stdout
    </store>
</match>
```  

`ctrl + c` 를 누르거나 `docker-compose.yaml` 경로에서 `docker-compose stop` 명령으로 전에 실행 시켰던 것을 중지 시킨다. 
그리고 다시 `docker-compose up --build` 명령으로 새로 추가한 플러그인과 설정을 다시 적용해 준다. 
그리고 `curl localhost:8080/strlog/30` 으로 큰 로그를 남기면 `efk-fluentd` 로그에서 아래와 같은 결과를 확인 할 수 있다. 

```
efk-fluentd_1        | 2020-07-21 20:40:26.000000000 +0000 web.log: {"log":"6789012345678901234567890123
45678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567

.. 생략 ..

89012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345678901
~~~end~~~","container_id":"e3b23c067322444c1a3e13739793015a5293e7779317c3ab2e5dc9a267a23273","container_
name":"/efk-spring_efk-spring_1","source":"stdout"}
```  

`30KB` 가 넘는 로그이지만 기존 처럼 2개의 로그로 나눠 찍히지 않고, 하나의 로그로 찍힌 것을 확인 할 수 있다. 
그리고 `Elasticsearch` 에도 `stdout` 으로 출력 된 로그가 전송되기 때문에, 
`Kibana` 에서도 나눠진 로그가 아닌 하나의 로그로 확인 할 수 있다.  

추가로 `concat` 을 수행하면서, 기존에 사용하던 로그 필드 추가나, 
리태깅이 가능한지 테스트 해본다. 
`fluent.conf` 파일에 아래의 내용을 추가한다. 

```
<source>
    @type forward
    port 24225
    bind 0.0.0.0
</source>

<filter **>
    @type concat
    key log
    use_first_timestamp true
    use_partial_metadata true
    separator ""
</filter>

# web.log 태그에서 INFO 라는 글자가 있으면, 해당 로그의 태그를 web.info.log 로 변경한다.
<match web.log>
    @type rewrite_tag_filter
    <rule>
        key log
        pattern INFO
        tag web.info.log
    </rule>
</match>

# web. 으로 시작하는 태그일 경우 해당 로그에 newfield 라는 키로 newdata 를 추가한다.
<filter web.**>
    @type record_transformer

    <record>
        newfield newdata
    </record>
</filter>

<match web.**>
    @type copy

    # elaticsearch 로 해당 로그를 전송한다.
    <store>
      @type elasticsearch
      # elasticsearch 호스트 설정
      host efk-elasticsearch
      port 9200
      logstash_format true
      logstash_prefix fluentd
      logstash_dateformat %Y%m%d
      include_tag_key true
      type_name access_log
      tag_key @log_name
      flush_interval 1s
    </store>

    # 표준 출력으로 출력한다.
    <store>
        @type stdout
    </store>
</match>
```  

`fluent.conf` 파일만 수정했으므로, `efk-fluentd` 만 재시작 해주면된다. 
`docker-compose.yaml` 파일 경로에서 `docker-compose restart efk-fluentd` 명령으로 새시작을 수행한다. 
그리고 `curl localhost:8080/strlog/30` 요청으로 로그를 남기고 `efk-fluentd` 로그를 확인하면 아래와 같다. 

```
efk-fluentd_1        | 2020-07-21 20:48:48.000000000 +0000 web.info.log: {"container_id":"e3b23c06732244
4c1a3e13739793015a5293e7779317c3ab2e5dc9a267a23273","container_name":"/efk-spring_efk-spring_1","source"
:"stdout","log":"2020-07-21 20:48:48.613  INFO 1 --- [nio-8080-exec-2] com.windowforsun.efk.TestControll
er      : ~~~start~~~01234567890123456789012345678901234567890123456789012345678901234567890123456789012
34567890123456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456

.. 생략 ..

23456789012345678901234567890123456789012345678901234567890123456789012345678901234567890123456789012345
7890123456789012345678901234567890123456789012345678901234567890123456789~~~end~~~","newfield":"newdata"}
```  

로그를 보면 앞부분에 `web.info.log` 라는 태그와 마지막 부분에 `"newfield":"newdata"` 라는 값이 추가 된 것을 확인 할 수 있다.  

이렇게 `concat` 을 수행해주는 플러그인을 사용하면 `Docker` 로그로 남겨지는 로그 중 로그의 크기가 커서 나눠 찍히던 로그들을 
하나의 로그로 합쳐 기존과 동일하게 모니터링을 수행할 수 있다. 

---
## Reference
[Running the Elastic Stack on Docker](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html)  
[fluent-plugin-concat](https://github.com/fluent-plugins-nursery/fluent-plugin-concat)  
[How to fix ‘elasticsearch exited with code 78’](https://techoverflow.net/2019/03/11/how-to-fix-elasticsearch-exited-with-code-78/)  
