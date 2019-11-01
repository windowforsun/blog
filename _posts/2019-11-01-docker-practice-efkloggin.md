--- 
layout: single
classes: wide
title: "[Docker 실습] Docker EFK Logging"
header:
  overlay_image: /img/docker-bg-2.jpg
excerpt: 'Docker 에서 Elasticsearch, Fluentd, Kibana 를 이용해서 로깅을 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Docker
tags:
    - Docker
    - Docker Compose
    - Practice
    - Spring
    - SpringBoot
    - Elasticsearch
    - Fluentd
    - Kibana
    - Logging
---  

# 환경
- Docker
- Docker Compose
- Spring Boot
- Maven
- Elasticsearch
- Kibana
- Fluentd

# 목표
- Docker 를 이용해서 EFK 을 구성한다.
- docker-compose 에서 컨테이너 로깅을 설정해서 EFK 에서 확인한다.

# 방법
- Docker 에서 로깅을 하는 방법중 하나는 애플리케이션(컨테이너)에서 `stdout`, `stderr` 와 같은 표준 출력으로 에러를 출력하는 것이다.
- docker-compose 로깅 설정을 통해 해당 컨테이너에서 출력되는 로그를 `fluentd` 로 보낸다.
- `fluentd` 는 받은 로그에 태그를 달고 `elasticsearch` 로 전송한다.
- `elasticsearch` 는 `fluentd` 로 부터 받은 로그를 저장하고, `kibana` 에서 index 설정을 통해 `elasticsearch` 에 저장된 로그를 모니터링 한다.


# 예제
- EFK + Web 을 Docker 로 구성하기 위해서는 가용 램용량이 최소 5기가는 되어야 한다.

## 프로젝트 구조

![그림 1]({{site.baseurl}}/img/docker/practice-efklogging-1.png)

## Docker EFK 구성하기
- `docker/docker-compose.yml`

	```yaml
	version: '3.7'
	
	services:	
	  test-efk-web:
	    build:
	      context: ./../
	      dockerfile: docker/web/Dockerfile
	    ports:
	      - "80:8080"
	    networks:
	      - test-efk-net
	    # Spring Web Application 에서 표준출력 로그를 web.log 라는 태그로 fluentd 로 전송한다.
	    logging:
	      driver: "fluentd"
	      options:
	        fluentd-address: localhost:24225
	        tag: web.log
	    depends_on:
	      - test-efk-fluentd
	
	  test-efk-fluentd:
	    build: ./fluentd
	    networks:
	      - test-efk-net
	    # web 쪽에서 받은 로그, elasticsearch 로 보낼 로그를 설정하는 파일을 마운트 해준다.
	    volumes:
	      - ./fluentd/conf:/fluentd/etc
	    ports:
	      - "24225:24225"
	      - "24225:24225/udp"
	
	  test-efk-elasticsearch:
	    image: elasticsearch:6.8.4
	    networks:
	      - test-efk-net
	    expose:
	      - 9200
	    ports:
	      - "9200:9200"
	      - "9300:9300"
	    # elasticsearch 에서 저장되는 데이터와 발생하는 로그를 호스트에 마운트한다.
	    volumes:
	      - ./ela-volume/data:/usr/share/elasticsearch/data
	      - ./ela-volume/logs:/usr/share/elasticsearch/logs
	
	  test-efk-kibana:
	    image: kibana:6.8.4
	    networks:
	      - test-efk-net
	    ports:
	      - "5601:5601"
	    depends_on:
	      - test-efk-elasticsearch
	    # 연결한 elasticsearch 호스트 이름을 설정한다.
	    environment:
	#      ELASTICSEARCH_HOSTS: http://test-efk-elasticsearch:9200
	      ELASTICSEARCH_URL: http://test-efk-elasticsearch:9200
	
	networks:
	  test-efk-net:
	```  
	
	- 전체적인 환경을 Docker 로 구성하고 각 service 간의 연동 설정이 작성되어 있다.
	
- `docker/web/Dockerfile`

	```
	### BUILD image
	FROM maven:3-jdk-11 as builder
	# create app folder for sources
	RUN mkdir -p /build
	WORKDIR /build
	COPY pom.xml /build
	#Download all required dependencies into one layer
	RUN mvn -B dependency:resolve dependency:resolve-plugins
	#Copy source code
	COPY src /build/src
	# Build application
	#RUN mvn package
	RUN mvn package -DskipTests	
	
	FROM openjdk:11-slim as runtime
	EXPOSE 8080
	#Set app home folder
	ENV APP_HOME /app
	#Possibility to set JVM options (https://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)
	ENV JAVA_OPTS=""
	#Create base app folder
	RUN mkdir $APP_HOME
	#Create folder to save configuration files
	RUN mkdir $APP_HOME/config
	#Create folder with application logs
	RUN mkdir $APP_HOME/log
	VOLUME $APP_HOME/log
	VOLUME $APP_HOME/config
	WORKDIR $APP_HOME
	#Copy executable jar file from the builder image
	COPY --from=builder /build/target/*.jar app.jar
	ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar" ]
	#Second option using shell form:
	#ENTRYPOINT exec java $JAVA_OPTS -jar app.jar $0 $@
	```  
	
	- 프로젝트에 작성된 Spring Application 을 빌드하고 실행한다.
	
- `docker/fluentd/Dockerfile`

	```
	# fluentd/Dockerfile
	FROM fluent/fluentd:v1.7
	USER root
	RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
	RUN ["gem", "install", "fluent-plugin-rewrite-tag-filter"]
	USER fluent
	```  
	
	- `fluentd` 이미지를 사용하고, elasticsearch 플러그인, rewrite tag filter 플러그인을 설치해서 `fluentd` 환경을 구성한다.
	
- `docker/fluentd/conf/fluent.conf`

	```xml
	<source>
	  @type forward
	  port 24225
	  bind 0.0.0.0
	</source>
	
	# test-efk-web 에서 전송되는 로그에 대한 설정
	<match web.log>
		# 각 로그 레벨에 맞춰 다시 태깅한다.
	    @type rewrite_tag_filter
	    <rule>
	        key log
	        pattern INFO
	        tag web.info.log
	     </rule>
	    <rule>
	        key log
	        pattern WARN
	        tag web.warn.log
	     </rule>
	</match>
	
	# 다시 태깅된 로그에 대한 설정
	<match web.*.*>
	  @type copy
	  
	  # elaticsearch 로 해당 로그를 전송한다.
	  <store>
	    @type elasticsearch
	    # elasticsearch 호스트 설정
	    host test-efk-elasticsearch
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
	
	- 웹쪽에서 받은 로그를 로그 레벨에 맞춰 다시 태깅하고, 다시 태깅된 로그를 elasticsearch 로 전송하는 설정이 작성되어 있다.

## Spring Web Application 작성하기
- 간단한 Resutful API 를 구현하고, API 에서 발생되는 로그를 남긴다.

### application.yaml
- 애플리케이션에서 사용할 로그를 설정한다.

```yaml
logging:
  level:
    com.example.springbootefk: [info, warn]
```  

### Member
- API 에서 사용할 Member 도메인을 정의한다.

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString
@EqualsAndHashCode
public class Member {
    private long id;
    private String name;
    private String email;
    private int age;
}
```  

### MemberController
- Member 도메인에 대한 Restful API 를 구현한다.
- 별도의 저장소는 구축하지 않고 전역 변수에 저장한다.
- 각 API 별 상황에 맞춰 info, warn 레벨의 로그를 남기고 있다.

```java
@RestController
@RequestMapping("/member")
@Slf4j
public class MemberController {
    public static HashMap<Long, Member> memberHashMap = new HashMap<>();

    @GetMapping
    public Map readAll() {
        List<Member> memberList = new LinkedList<>(memberHashMap.values());

        if(memberList.size() == 0) {
            log.warn("readAll: empty");
        }

        Map<String, Object> map = new HashMap<>();
        map.put("members", memberList);

        log.info("readAll:" + Arrays.toString(memberList.toArray()));
        return map;
    }

    @GetMapping("/{id}")
    public Member readById(@PathVariable long id) {
        Member member = memberHashMap.get(id);

        if(!(member instanceof Member)) {
            log.warn("read: empty");
        }

        log.info("read:" + member);
        return member;
    }

    @PostMapping
    public Member create(@RequestBody Member member) {
        member.setId(memberHashMap.size());
        Member exist = memberHashMap.put(member.getId(), member);

        if(exist != null) {
            log.warn("create: exists");
        }

        log.info("create:" + member);
        return member;
    }

    @PutMapping("/{id}")
    public Member update(@PathVariable long id, @RequestBody Member member) {
        member.setId(id);
        Member exist = memberHashMap.put(id, member);

        if(exist == null) {
            log.warn("update: not exists");
        }

        log.info("update:" + member);
        return member;
    }

    @DeleteMapping("/{id}")
    public Member deleteMember(@PathVariable long id) {
        Member member = memberHashMap.get(id);

        if(member instanceof Member) {
            memberHashMap.remove(id);
            log.info("delete:" + member);
        } else {
            log.warn("delete: not exists");
        }

        return member;
    }
}
```  

## 실행 및 테스트
- `docker/` 경로에서 `docker-compose up --build` 명령어를 통해 빌드 및 실행한다.
	- efk 를 실행하기 위해서는 최소 5기가 이상 가용램이 필요하다.

	```
	[root@localhost docker]# docker-compose up --build
	Building test-efk-fluentd
	Step 1/5 : FROM fluent/fluentd:v1.7
	 ---> cfa9329df41a
	Step 2/5 : USER root
	 ---> Using cache
	 ---> ac5a784f52ad
	Step 3/5 : RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.5"]
	 
	 # 생략
	
	Step 19/19 : ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar app.jar" ]
	 ---> Using cache
	 ---> da4a56e29de9
	
	Successfully built da4a56e29de9
	Successfully tagged docker_test-efk-web:latest
	Starting docker_test-efk-fluentd_1       ... done
	Starting docker_test-efk-elasticsearch_1 ... done
	Starting docker_test-efk-kibana_1        ... done
	Starting docker_test-efk-web_1           ... done

	# 생략
	```  

- `http://<server-ip>:5601` 로 접속하면 아래와 같은 화면을 확인할 수 있다.

	![그림 2]({{site.baseurl}}/img/docker/practice-efklogging-2.png)
	
- 로그가 쌓이지 않은 tag 에 대해서는 index 생성이 되지 않으므로, 로그가 없다면 Spring Web Application 으로 요청을 보내 로그를 쌓아 준다.
- `fluentd-*` 로 인덱스를 생성해 준다.

	![그림 3]({{site.baseurl}}/img/docker/practice-efklogging-3.png)
	
	![그림 4]({{site.baseurl}}/img/docker/practice-efklogging-4.png)
	
- 구현된 API 로 요청을 보내게되면 아래와 같이 `@log_name` 필드가 로그 레벨로 구분되어 모니터링이 가능하다.

	![그림 5]({{site.baseurl}}/img/docker/practice-efklogging-5.png)


---
## Reference
