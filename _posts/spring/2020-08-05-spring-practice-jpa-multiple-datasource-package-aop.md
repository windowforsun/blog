--- 
layout: single
classes: wide
title: "[Spring 실습] JPA Multiple DataSource(Database) Package, AOP"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Replication, Sharding 에서 필요한 JPA 다중 Datasource 를 Transaction 을 활용해서 구현해 보자'
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
    - Package
    - AOP
toc: true
use_math: true
---  

## JPA Multiple DataSource
[여기]({{site.baseurl}}{% link _posts/spring/2020-08-01-spring-practice-jpa-multiple-datasource-transaction.md %})
포스트에서는 `Transaction` 을 사용해서 `Replication` 구저에서 다중 `DataSource` 를 활용하는 방법에 대해 알아봤다. 
이번 글에서는 미리 소개한 3가지 방법 중 나머지인 패키지와 `AOP` 를 사용해서 이를 활용하는 방법에 대해 알아본다. 

## Package 방법
`Package` 를 사용하는 방법은 `JPA` 관련 설정에 `DataSource` 와 해당하는 `JpaRepository` 패키지를 묶어 구성하는 방법을 사용한다. 
`Master` 에 해당하는 `DataSource` 관련 설정 파일에 `Master` 연산을 수행하는 `JpaRepository` 패키지만 따로 분리해 구현 한후 해당 패키지를 등록하고, 
`Slave` 또한 `DataSource` 관련 설정 파일에 `Slave` 연산을 수행하는 `JpaRepository` 패키지를 등록해서 `DataSource` 가 분리 될 수 있도록 한다.  

데이터 베이스는 `Master1`, `Slave1` 만 사용한다.  

디렉토리 구조는 아래와 같다. 

```
package
├── build.gradle
└── src
    └── main
        ├── generated
        ├── java
        │   └── com
        │       └── windowforsun
        │           └── mpackage
        │               ├── PackageApplication.java
        │               ├── config
        │               │   ├── MasterDataSourceConfig.java
        │               │   └── SlaveDataSourceConfig.java
        │               ├── constant
        │               │   └── EnumDB.java
        │               ├── controller
        │               │   └── ExamController.java
        │               ├── domain
        │               │   └── Exam.java
        │               ├── repository
        │               │   ├── master
        │               │   │   └── ExamMasterRepository.java
        │               │   └── slave
        │               │       └── ExamSlaveRepository.java
        │               └── service
        │                   ├── ExamMasterService.java
        │                   └── ExamSlaveService.java
        └── resources
            └── application.yaml
```  

우선 별도로 설명이 필요하지 않는 클라스 내용은 아래와 같다. 

```java
@SpringBootApplication
public class PackageApplication {
    public static void main(String[] args) {
        SpringApplication.run(PackageApplication.class, args);
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

    @PreUpdate
    @PrePersist
    public void preUpdate() throws Exception{
        Thread.sleep(150);
    }
}
```  












































---
## Reference
[Use replica database for read-only transactions](https://blog.pchudzik.com/201911/read-from-replica/)  