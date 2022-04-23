--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Data R2DBC 성능 테스트"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Data R2DBC 성능을 JDBC, Spring Web, Spring Webflux 등 다양한 케이스 구성으로 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Data R2DBc
    - Reactive Stream
    - Spring Webflux
    - R2DBC
    - MySQL
    - Reactor
    - DB
    - JDBC
    - Spring MVC
    - Spring Web
toc: true
use_math: true
---  

## Spring Data R2DBC Performance
`Spring Data R2DBC` 에 대해 관심이 있고 알아보는 사람이라면, 
대부분 `Spring Webflux` 의 성능 개선에 대한 점을 궁금해 할 것이다. 
이번 포스트에서는 평소에 궁금했던 몇가지 구성에 대해 성능 테스트를 진행해보고, 
그 결과를 통해 `Spring Data R2DBC` 의 성능을 알아보고자 한다.  

### 테스트 환경 

컴포넌트|버전
---|---
Spring Boot|2.5.0
MySQL|8.0
Docker|20.10.14

테스트는 아래와 같은 총 5가지 케이스로 구성돼 있다. 

구분|Framework|Server|DB Driver|Connection Pool|비고
---|---|---|---|---|---
Spring Web + JDBC|Spring Web|Tomcat|Spring Data JDBC(JPA)|HikariCP| 
Spring Web + R2DBC|Spring Web|Tomcat|Spring Data R2DBC|R2DBC-Pool| 
Spring Webflux + JDBC(Scheduler)|Spring Webflux|Netty|Spring Data JDBC(JPA)|HikariCP|Scheduler 사용
Spring Webflux + JDBC|Spring Webflux|Netty|Spring Data JDBC(JPA)|HikariCP|Scheduler 미사용 
Spring Webflux + R2DBC|Spring Webflux|Netty|Spring Data R2DBC|R2DBC-Pool| 

> `R2DBC` `MySQL` 드라이버는 `dev.miku:r2dbc-mysql` 을 사용했다.  

케이스별 `Thread` 관련 설정 값은 아래와 같다.  

구분|Min Server Thread|MAX Server Thread|Min DB Connection Pool|Max DB Connection Pool
---|---|---|---|---
Spring Web + JDBC|250|600|250|600 
Spring Web + R2DBC|250|600|100|150 
Spring Webflux + JDBC(Scheduler)|4|4|250|600
Spring Webflux + JDBC|4|4|250|600 
Spring Webflux + R2DBC|4|4|100|150 

`Spring Webflux + JDBC` 구성은 또 `Scheduler` 사용 유무로 2가지로 구분된다. 
이는 `Spring Webflux` 와 `JDBC` 를 사용할 때 주로 요청 스레드의 블로킹을 피하기 위해 `Scheduler` 를 사용하게 되는데, 
이를 실제로 사용했을 때와 사용하지 않았을 때의 성능 차이를 알아보고자 추가 했다.  


우선 테스트는 `SELECT` 쿼리인 조회 성능에 대해서만 진행 한다.  

> 추후에 `INSERT`, `UPDATE`, `DELETE` 도 추가할 계획이다.  

- 의존성

```groovy
//Spring Web + JDBC
dependencies {
	implementation "org.springframework.boot:spring-boot-starter-web"
	implementation "org.springframework.boot:spring-boot-starter-data-jpa"
	runtimeOnly "mysql:mysql-connector-java"
}
```  

```groovy
//Spring Web + R2DBC
dependencies {
	implementation "org.springframework.boot:spring-boot-starter-web"
	implementation "org.springframework.boot:spring-boot-starter-data-jpa"
	runtimeOnly "mysql:mysql-connector-java"
	runtimeOnly "dev.miku:r2dbc-mysql"
}
```  

```groovy
//Spring Webflux + JDBC(Scheduler), Spring Webflux + JDBC
dependencies {
	implementation "org.springframework.boot:spring-boot-starter-webflux"
	implementation "org.springframework.boot:spring-boot-starter-data-jpa"
	runtimeOnly "mysql:mysql-connector-java"
}
```  

```groovy
//Spring Webflux + R2DBC
dependencies {
	implementation "org.springframework.boot:spring-boot-starter-data-r2dbc"
	implementation "org.springframework.boot:spring-boot-starter-webflux"
	runtimeOnly "mysql:mysql-connector-java"
	runtimeOnly "dev.miku:r2dbc-mysql"
}
```  


- `Data Model`

```java
// Spring Data R2DBC
@Getter
@Setter
@Builder
@ToString
@Table("member")
@NoArgsConstructor
@AllArgsConstructor
public class Member {
    @Id
    private Long id;
    private String name;
    private int age;
    private Long teamId;
}
```  

```java
// Spring Data JPA
@Data
@Builder
@Entity
@Table(name = "member")
@NoArgsConstructor
@AllArgsConstructor
public class Member {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private int age;
    private Long teamId;
}
```

- `Table Schema`

```sql
CREATE TABLE member (
    id INT(20) AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(20) NOT NULL,
    age INT(3) NOT NULL,
    team_id INT(20)
);
```  

- `Repository`, `Query`

```java
// Spring Data JPA
@Repository
public interface MemberRepository extends JpaRepository<Member, Long> {
    @Query(value = "SELECT * FROM member WHERE SLEEP(0.001) = 0 ORDER BY RAND() LIMIT 1", nativeQuery = true)
    Member findSlow();

}
```  

```java
// Spring Data R2DBC
@Repository
public interface MemberRepository extends ReactiveCrudRepository<Member, Long> {
    @Query("SELECT * FROM member WHERE SLEEP(0.001) = 0 ORDER BY RAND() LIMIT 1")
    Mono<Member> findSlow();

}
```  

- `Controller`

```java
// Spring Web + JDBC
@GetMapping("/findSlow")
public Member findSlow() {
	return this.memberRepository.findSlowQuery();
}
```  

```java
// Spring Web + R2DBC
@GetMapping("/findSlow")
public Mono<Member> findSlow() {
	return this.memberRepository.findSlow();
}
```  

```java
// Spring Webflux + JDBC(Scheduler)

@GetMapping("/findSlow")
public Mono<Member> findSlow() {
	return Mono.defer(() -> Mono.fromCallable(() -> this.memberRepository.findSlowQuery()).subscribeOn(Schedulers.boundedElastic()));
}
```  

```java
// Spring Webflux + JDBC
@GetMapping("/findSlowBlock")
public Mono<Member> findSlowBlock() {
	return Mono.just(this.memberRepository.findSlowQuery());
}
```  

```java
// Spring Webflux + R2DBC
@GetMapping("/findSlow")
public Mono<Member> findSlow() {
	return this.memberRepository.findSlow();
}
```

테스트에서 사용할 조회 쿼리는 `MySQL` 의 `SLEEP()` 메소드를 사용해서 조회 쿼리에서 어느정도 지연이 발생 할 수 있도록 했다.  

- `JMeter` 설정

설정 이름|설정 값
---|---
Thread|600
테스트 시간|60s

- 테스트 애플리케이션 `Docker` 구성

구분|값
---|---
CPU|2
명령어|docker run --rm --name <name> --cpus="2" -p 8080:8080 -p 9010:9010 <image>



### 테스트 결과
테스트는 한가지 구성당 총 2번 연속해서 진행했다. 
첫 번째 테스트는 `WarmUp` 을 수행해주고, 
테스트 결과로 사용한건 2번째 테스트이다.  

아래는 `RPS` 가 높은 순위로 나열한 테스트 결과 이다.  

구분|CPU|Memory|Total Thread|RPS|Latency
---|---|---|---|---|---|---
Spring Webflux + R2DBC|100%|1.4G|23|3221|184ms
Spring Webflux + JDBC(Scheduler)|60%|1.2G|40|2787|212ms
Spring Web + R2DBC|100%|2.0G|625|2270|261ms
Spring Web + JDBC|90%|1.5G|622|1496|349ms
Spring Webflux + JDBC|10%|1.0G|21|548|1073ms

#### Spring Webflux + R2DBC
`HTTP` 요청 부터 `DB` 요청 까지 모두 `Reactive` 하게 처리되어서 그런지 가장 좋은 성능을 보여줬다. 
성능 뿐만 아니라 리소스 사용 측면에서도 비교적 적은 `Memory` 와 `Thread` 를 사용해서 최대의 효율을 뽑아 냈다고 할 수 있다.  

#### Spring Webflux + JDBC(Scheduler)
`HTTP` 요청은 `Netty` 를 통해 `Reactive` 하게 수행되지만 `DB` 요청은 `JDBC` 이므로 블로킹이 존재한다. 
이를 `Scheduler` 를 사용해서 비동기로 `Netty` 스레드에서 블로킹이 발생하지 않도록 회피한 결과이다. 
`Spring Webflux + R2DBC` 에 비해 더 많은 `Thread` 를 사용하고 이에 따라 `Context Switch` 도 더 빈번하게 일어난 결과로 
약간의 성능 저하가 있었던 걸로 보인다. 
그리고 `JDBC` 요청 처리를 `Scheduler` 를 통해 비동기로 전환 했지만, 
`Scheduler` 의 `Thread-Pool` 에 처리할 `Task` 가 계속 쌓이게 되면서 전체 `CPU` 도 `60%` 까지 밖에 사용 하지 못한 걸로 보인다.  

#### Spring Web + R2DBC
`HTTP` 요청은 `Tomcat` 을 통해 요청당 하나의 `Thread` 가 할당된다. 
하지만 `DB` 요청은 `R2DBC` 를 통해 `Reactive` 하게 이뤄진 결과로 
`Spring Web + JDBC` 보다는 더 좋은 성능을 보여 줬다.  

#### Spring Web + JDBC
`HTTP` 요청은 `Tomcat` 을 통해 요청당 하나의 `Thread` 가 할당 되고,
그 요청 쓰레드에서 `JDBC` 의 커넥션을 통해 `DB` 요청이 수행 된다. 
이에 따라 `DB` 처리 시간에 따라 요청 `Thread` 는 블로킹 되기 때문에 
`Spring Web + R2DBC` 보다는 더 안 좋은 성능을 보여줬다.  

#### Spring Webflux + JDBC
`HTTP` 요청은 `Netty` 를 통해 `Reactive` 하게 이뤄지지만, 
`Netty Thread` 에서 `JDBC` 커넥션을 통해 `DB` 요청이 수행된다. 
즉 `DB` 처리 시간에 따라 `Netty Thread` 는 다른 요청을 받을 수 없는 상황이 되기 때문에 
`CPU` 도 `10%` 밖에 사용하지 못하는 상황이 발생하고 이에 따라 가장 안좋은 성능을 보여줬다.  



---  
## Reference
[Performance of relational database drivers. R2DBC vs JDBC](https://technology.amis.nl/software-development/performance-and-tuning/performance-of-relational-database-drivers-r2dbc-vs-jdbc/)  
[R2DBC vs JDBC performance](https://filia-aleks.medium.com/r2dbc-vs-jdbc-19ac3c99fafa)  
[Spring: Blocking vs non-blocking: R2DBC vs JDBC and WebFlux vs Web MVC](https://javaoraclesoa.blogspot.com/2020/04/blocking-vs-non-blocking-in-spring.html)  
  