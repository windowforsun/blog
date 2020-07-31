--- 
layout: single
classes: wide
title: "[Spring 실습] JPA Multiple DataSource(Database)"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Replication, Sharding 구성에서 필요한 JPA 다중 Datasource 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - JPA
    - Replication
    - Sharding
    - DataSource
toc: true
use_math: true
---  

## JPA Multiple DataSource
보편적으로 `Database` 튜닝을 위해서 `Replication` 이나 `Sharding` 을 구성한다. 
이를 통해 시스템 하나로 몰리던 부하를 여러 시스템으로 분산시킬 수 있다. 
이런 `Database` 구조를 사용하기 위해서는 별도의 `Proxy` 를 두거나, 
애플리케이션에서 이런 구조에 대한 추가 처리가 필요하다.  

애플리케이션에서 지원 가능하도록 하는 다양한 방법이 있겠지만, 
그 중 `Spring` 애플리케이션에서 코드를 통해 적용하는 아래 3가지 방법에 대해 알아본다. 
- `Transaction` 의 `readOnly` 속성을 사용하는 방법
- 패키지를 분리해서 사용하는 방법
- `Annotation` 을 사용하는 방법

## Docker 기반 Database 구성
테스트를 위해 `Database` 구성은 `Docker` 를 사용해서 구성한다. 
`Replication` 구조로 되어있고, `Replication` 관련 자세한 설명은 [여기]({{site.baseurl}}{% link _posts/mysql/2020-07-28-mysql-practice-replication-container.md %}) 
에서 확인 할 수 있다.  

디렉토리 구조는 아래와 같다. 

```bash
.
├── docker-compose.yaml
├── master-1
│   ├── conf.d
│   │   └── custom.cnf
│   └── init
│       └── _init.sql
├── master-2
│   ├── conf.d
│   │   └── custom.cnf
│   └── init
│       └── _init.sql
├── slave-1
│   ├── conf.d
│   │   └── custom.cnf
│   └── init
│       ├── _init.sql
│       └── replication.sh
└── slave-2
    ├── conf.d
    │   └── custom.cnf
    └── init
        ├── _init.sql
        └── replication.sh
```  

전체적인 컨테이너 템플릿을 정의하는 `docker-compose.yaml` 파일내용은 아래와 같다. 

```yaml
version: '3.7'

services:
  master-db-1:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - db-net
    volumes:
      - ./master-1/conf.d/:/etc/mysql/conf.d
      - ./master-1/init/:/docker-entrypoint-initdb.d/
    ports:
      - 33000:3306

  slave-db-1:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MASTER_HOST: master-db-1
      MASTER_PORT: 3306
      MASTER_USER: root
      MASTER_PASSWD: root
      MASTER_REPL_USER: slaveuser
      MASTER_REPL_PASSWD: slavepasswd
      MASTER_LOG_BIN: mysql-bin
    networks:
      - db-net
    volumes:
      - ./slave-1/conf.d/:/etc/mysql/conf.d
      - ./slave-1/init/:/docker-entrypoint-initdb.d/
    ports:
      - 34000:3306

  master-db-2:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
    networks:
      - db-net
    volumes:
      - ./master-2/conf.d/:/etc/mysql/conf.d
      - ./master-2/init/:/docker-entrypoint-initdb.d/
    ports:
      - 33001:3306

  slave-db-2:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root
      MASTER_HOST: master-db-2
      MASTER_PORT: 3306
      MASTER_USER: root
      MASTER_PASSWD: root
      MASTER_REPL_USER: slaveuser
      MASTER_REPL_PASSWD: slavepasswd
      MASTER_LOG_BIN: mysql-bin
    networks:
      - db-net
    volumes:
      - ./slave-2/conf.d/:/etc/mysql/conf.d
      - ./slave-2/init/:/docker-entrypoint-initdb.d/
    ports:
      - 34001:3306

  adminer:
    image: adminer
    restart: always
    ports:
      - 8888:8080
    networks:
      - db-net

networks:
  db-net:
```  

- 총 4개의 `Databas` 로 구성돼있고, `Master-Slave` 한쌍씩 되어있다. 
- 실제로 구성한다면 `Database` 는 개별 호스트에 독립 컨테이너 형식이기 때문에 별도의 서비스로 만들었다. 
- `Master` 와 `Slave` 설정에 약간의 차이점이있고, 
`Master1`, `Master2` 와 `Slave1`, `Slave2` 는 포트나 참조하는 `Database` 의 값만 다르다. 
- `Master1-Slave1`, `Master2-Slave2` 의 구조로 구성돼 있다. 
- `Slave` 에서는 `Master` 관련 정보를 환경변수로 받아 설정한다. 

`Master` 의 `MySQL` 설정 파일인 `master-*/conf.d/custom.cnf` 파일의 내용은 아래와 같다. 

```
[mysqld]
server-id=1
log-bin=mysql-bin
default-authentication-plugin=mysql_native_password
```  

`Master` 의 초기화 `SQL` 파일은 `master-*/init/_init.sql` 파일은 아래와 같다.

```sql
create database test;

use test;

-- Master1 --
CREATE TABLE `exam` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----

-- Master2 --
CREATE TABLE `exam2` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----


create user 'slaveuser'@'%' identified by 'slavepasswd';
grant replication slave on *.* to 'slaveuser'@'%';

flush privileges;
```  

- `Master1` 은 `exam` 테이블을 가지고, `Master2` 는 `exam2` 테이블을 갖는다. 

다음으로 `Slave` 의 `MySQL` 설정 파일인 `slave-*/conf.d/custom.cnf` 파일 내용은 아래와 같다. 

```
[mysqld]
server-id=2
default-authentication-plugin=mysql_native_password
```  

`Slave` 의 초기화 `SQL` 파일인 `slave-*/init/_init.sql` 파일 내용은 아래와 같다. 

```sql
create database test;

use test;

-- Slave1 --
CREATE TABLE `exam` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----

-- Slave2 --
CREATE TABLE `exam2` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `value2` varchar(255) DEFAULT NULL,
  `datetime` datetime DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
----
```  

- `Master` 와 동일하게 `Slave1` 은 `exam` 테이블을 `Slave2` 은 `exam2` 테이블을 갖는다. 

`Slave` 가 구동되면서 환경변수의 `Master` 관련 정보를 사용해서 `Replication` 을 구성하는 `slave-*/init/replication.sh` 파일 내용은 아래와 같다. 

```shell script
# /bin/bash

# function for query to master
function doMasterQuery() {
  local query="$1"
  local grepStr="$2"
  local command="mysql -h${MASTER_HOST} -u ${MASTER_USER} -p${MASTER_PASSWD} -e \"${query}\""

  if [ ! -z "${grepStr}" ]; then
    command="${command} | grep ${grepStr}"
  fi

  echo `eval "$command"`
}

# function for query to slave
function doSlaveQuery() {
  local query="$1"
  local grepStr="$2"
  local command="mysql -u root -p${MYSQL_ROOT_PASSWORD} -e \"${query}\""

  if [ ! -z "${grepStr}" ]; then
    command="${command} | grep ${grepStr}"
  fi

  echo `eval "$command"`
}

master_log_file=''

# waiting master connection and get master log path
while [[ ${master_log_file} != *"${MASTER_LOG_BIN}"* ]]; do
  master_log_file=$(doMasterQuery "show master status\\G" "${MASTER_LOG_BIN}")
  sleep 3
done

# get master log position
master_log_pos=$(doMasterQuery "show master status\\G", "Position")

# parsing log file
re="${MASTER_LOG_BIN}.[0-9]*"
if [[ ${master_log_file} =~ $re ]]; then
  master_log_file=${BASH_REMATCH[0]}
fi

# parsing log position
re="[0-9]+"
if [[ ${master_log_pos} =~ $re ]]; then
  master_log_pos=${BASH_REMATCH[0]}
fi

# set master info
replication_query="change master to
master_host='${MASTER_HOST}',
master_user='${MASTER_REPL_USER}',
master_password='${MASTER_REPL_PASSWD}',
master_log_file='${master_log_file}',
master_log_pos=${master_log_pos}"
echo $(doSlaveQuery "${replication_query}")

echo $(doSlaveQuery "start slave")
```  

- `Master` 가 완전히 올라갈때 까지 대기한다.
- `show master status` 에서 현재 로그 파일 명을 파싱한다. 
- `show master status` 에서 현재 로그 포지션을 파싱한다. 
- `change master to` 명령으로 마스터 설정을 한다. 

## Spring 애플리케이션
애플리케이션은 `Spring Boot` 기반이고, 빌드 툴은 `Gradle` 을 사용했다. 
프로젝트 디렉토리 구조는 아래와 같다. 

```
.
├── README.md
├── aop
│   ├── build.gradle
│   └── src
│       └── main
│           ├── generated
│           ├── java
│           │   └── com
│           │       └── windowforsun
│           │           └── aop
│           │               ├── AopApplication.java
│           │               ├── aop
│           │               │   ├── SetDataSource.java
│           │               │   └── SetDataSourceAspect.java
│           │               ├── compoment
│           │               │   └── RoutingDatasourceManager.java
│           │               ├── config
│           │               │   ├── DataSourceConfig.java
│           │               │   └── RoutingDataSource.java
│           │               ├── constant
│           │               │   └── EnumDB.java
│           │               ├── controller
│           │               │   ├── Exam2Controller.java
│           │               │   └── ExamController.java
│           │               ├── domain
│           │               │   ├── Exam.java
│           │               │   └── Exam2.java
│           │               ├── repository
│           │               │   ├── Exam2Repository.java
│           │               │   └── ExamRepository.java
│           │               └── service
│           │                   ├── Exam2MasterService.java
│           │                   ├── Exam2SlaveService.java
│           │                   └── ExamService.java
│           └── resources
│               └── application.yaml
├── build.gradle
├── demo
│   ├── build.gradle
│   └── src
│       └── main
│           ├── generated
│           ├── java
│           │   └── com
│           │       └── windowforsun
│           │           └── demo
│           │               ├── DemoApplication.java
│           │               ├── config
│           │               │   ├── DataSourceConfig.java
│           │               │   └── ReplicationRoutingDataSource.java
│           │               ├── controller
│           │               │   └── ExamController.java
│           │               ├── domain
│           │               │   └── Exam.java
│           │               ├── repository
│           │               │   └── ExamRepository.java
│           │               └── service
│           │                   ├── ExamMasterService.java
│           │                   └── ExamSlaveService.java
│           └── resources
│               └── application.yaml
├── gradlew
├── gradlew.bat
├── package
│   ├── build.gradle
│   └── src
│       └── main
│           ├── generated
│           ├── java
│           │   └── com
│           │       └── windowforsun
│           │           └── mpackage
│           │               ├── PackageApplication.java
│           │               ├── config
│           │               │   ├── MasterDataSourceConfig.java
│           │               │   └── SlaveDataSourceConfig.java
│           │               ├── controller
│           │               │   └── ExamController.java
│           │               ├── domain
│           │               │   └── Exam.java
│           │               ├── repository
│           │               │   ├── master
│           │               │   │   └── ExamMasterRepository.java
│           │               │   └── slave
│           │               │       └── ExamSlaveRepository.java
│           │               └── service
│           │                   ├── ExamMasterService.java
│           │                   └── ExamSlaveService.java
│           └── resources
│               └── application.yaml
├── settings.gradle
├── src
│   └── main
│       ├── java
│       └── resources
└── transaction
    ├── build.gradle
    └── src
        └── main
            ├── java
            │   └── com
            │       └── windowforsun
            │           └── transaction
            │               ├── TransactionApplication.java
            │               ├── config
            │               │   ├── DataSourceConfig.java
            │               │   └── ReplicationRoutingDataSource.java
            │               ├── controller
            │               │   └── ExamController.java
            │               ├── domain
            │               │   └── Exam.java
            │               ├── repository
            │               │   └── ExamRepository.java
            │               └── service
            │                   ├── ExamMasterService.java
            │                   └── ExamSlaveService.java
            └── resources
                └── application.yaml
```  

```
.
├── aop
│   ├── build.gradle
|   .. 생략 ..
├── build.gradle
├── gradlew
├── gradlew.bat
├── package
│   ├── build.gradle
|   .. 생략 ..
├── settings.gradle
└── transaction
    ├── build.gradle
    .. 생략 ..
```  

하나의 프로젝트에 `aop`, `package`, `transaction` 이라는 이름으로 3가지 모듈로 구성했다. 
각 모듈이 앞서 설명한 `JPA Multiple DataSource` 를 구성하는 하나의 방법에 대한 구현체이다. 

먼저 `build.gradle` 파일 내용은 아래와 같다. 

```groovy
plugins {
    id 'io.spring.dependency-management' version '1.0.8.RELEASE'
    id 'org.springframework.boot' version '2.2.1.RELEASE'
    id 'java'
}

group 'com.windowforsun'
version '1.0-SNAPSHOT'

bootJar {
    enabled = false
}

sourceCompatibility = 1.8

allprojects {
    repositories {
        mavenCentral()
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'
    apply plugin: 'org.springframework.boot'

    dependencies {
        compile 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testCompile group: 'junit', name: 'junit', version: '4.12'
        testImplementation group: 'org.hamcrest', name: 'hamcrest-all', version: '1.3'
    }
}
```  

`settings.gradle` 파일은 아래와 같다. 

```groovy
rootProject.name = 'replicationexam'
include 'demo'
include 'transaction'
include 'package'
include 'aop'
```  

각 애플리케이션에 필요한 내용이 있는 `application.yaml` 내용은 아래와 같다.

```yaml
spring:
  datasource:
    masterdb1:
      jdbc-url: jdbc:mysql://localhost:33000/test?serverTimezone=Asia/Seoul
      username: root
      password: root

    slavedb1:
      jdbc-url: jdbc:mysql://localhost:34000/test?serverTimezone=Asia/Seoul
      username: root
      password: root

    masterdb2:
      jdbc-url: jdbc:mysql://localhost:33001/test?serverTimezone=Asia/Seoul
      username: root
      password: root

    slavedb2:
      jdbc-url: jdbc:mysql://localhost:34001/test?serverTimezone=Asia/Seoul
      username: root
      password: root
  jpa:
    database: mysql
    hibernate:
      use-new-id-generator-mappings: false
      ddl-auto: update
    generate-ddl: true

logging:
  level:
    com.windowforsun: DEBUG
    org.springframework.jdbc.datasource.SimpleDriverDataSource: DEBUG
    org.springframework.jdbc.datasource: DEBUG
    org.hibernate.SQL: DEBUG
```  

- `.spring.datasource` : 하위 필드에 필요한 데이터베이스 연결에 필요한 `jdbc-url`, `username`, `password` 정보를 설정한다. 
커넥션 풀로는 `hikari` 를 사용하기 때문에 `jdbc-url` 필드를 사용한다. 
- `.spring.jpa` : 하위 필드에 `jpa` 관련 설정을 한다. 
- `.logging` : 애플리케이션에서 로그를 출력할 패키지나 클래스 이름과 레벨을 명시해 준다. 

공통적인 내용은 여기까지 이고, 
이제 부턴 앞서 설명한 3가지 방법에 대해 알아본다. 
각 방법에서는 `Controller` 로 웹요청에 대한 `Endpoint` 를 정의한다. 
여기서 `REST` 방식을 따르지 않고 `HTTP GET` 메소드만 사용해서 여러 기능을 구현한다. 
또한 별도의 테스트 클래스도 구성하지 않는다. 

### Transaction
구현이 가장 간단한 방법인 `Transaction` 의 `readOnly` 필드를 사용하는 방법이다. 
데이터베이스는 `Master1`, `Slave1` 만 사용한다.  

디렉토리 구조는 아래와 같다. 

```
transaction
├── build.gradle
└── src
    └── main
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── transaction
        │               ├── TransactionApplication.java
        │               ├── config
        │               │   ├── DataSourceConfig.java
        │               │   └── ReplicationRoutingDataSource.java
        │               ├── controller
        │               │   └── ExamController.java
        │               ├── domain
        │               │   └── Exam.java
        │               ├── repository
        │               │   └── ExamRepository.java
        │               └── service
        │                   ├── ExamMasterService.java
        │                   └── ExamSlaveService.java
        └── resources
            └── application.yaml
```  

우선 별도의 설명을 하지 않는 여타 클래스 내용은 아래와 같다. 

```java
@SpringBootApplication
public class TransactionApplication {
    public static void main(String[] args) {
        SpringApplication.run(TransactionApplication.class, args);
    }
}
```  

```java
@Entity
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Table(
        name = "exam"
)
public class Exam {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private long id;
    @Column
    private String value;
    @Column
    private LocalDateTime datetime;
}
```  

```java
@RestController
@RequestMapping("/exam")
public class ExamController {
    private final ExamMasterService examMasterService;
    private final ExamSlaveService examSlaveService;

    public ExamController(ExamMasterService examMasterService, ExamSlaveService examSlaveService) {
        this.examMasterService = examMasterService;
        this.examSlaveService = examSlaveService;
    }

    @GetMapping("/create")
    public Exam create() {
        Exam exam = Exam.builder()
                .value(System.currentTimeMillis() + "")
                .datetime(LocalDateTime.now())
                .build();
        return this.examMasterService.save(exam);
    }

    @GetMapping("/delete")
    public void deleteAll() {
        this.examMasterService.deleteAll();
    }

    @GetMapping("/delete/{id}")
    public void delete(@PathVariable long id) {
        this.examMasterService.deleteById(id);
    }

    @GetMapping("/")
    public List<Exam> readAll() {
        return this.examSlaveService.readAll();
    }

    @GetMapping("/{id}")
    public Exam read(@PathVariable long id) {
        return this.examSlaveService.readById(id);
    }
}
```  










































---
## Reference
