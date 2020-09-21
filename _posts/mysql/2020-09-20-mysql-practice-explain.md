--- 
layout: single
classes: wide
title: "[MySQL 실습] "
header:
  overlay_image: /img/mysql-bg.png
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
  - MySQL
  - Practice
  - Explain
toc: true
use_math: true
---  

## 실행계획
`DBMS` 에서 쿼리에 대한 결과를 만들때 한가지 방법만 있지는 않고, 상황에 따라 다양한 방법으로 결과를 만들어 낼 수 있다. 
여기서 `DBMS` 는 옵티마이저(`Optimizer`) 를 통해 최소의 비용이 소모되는 방법을 선택하게 된다. 
그리고 이러한 선택을 하기위해 각 테이블의 데이터가 어떤 분포로 저장돼 있는지 통계 정보를 관리하고 참조한다. 
`MySQL` 에서 사용자가 작성한 쿼리를 옵티마이저가 어떻게 실행할지 실행계획을 볼 수 있는 것이 바로 `EXPLAIN` 이다. 

### 쿼리 실행 과정
`EXPLAIN` 은 옵티마이저가 수행할 계획에 대한 설명을 나타내는 데이터로 이를 이해하기 위해서는 먼저 쿼리의 실행 절차에 대하 알아본다. 
1. 사용자가 작성한 `SQL`을 `MySQL` 이 이해할 수 있는 구문으로 분리한다. 
1. `SQL` 파싱 정보인 파스 트리를 확인하며 쿼리 수행을 위해 어떤 테이블을 사용하고 어떤 인덱스를 사용할지 결정한다. 
1. 스토리지 엔진으로 부터 앞서 결정한 테이블의 읽기 순서, 인덱스를 사용해서 데이터를 가져온다. 

위 과정을 다시 정리하면 첫번째 단계는 `SQL Pasing` 단계로 `SQL` 파서 모듈이 담당한다. 
파싱 단계에서 `SQL Parse Tree` 가 생성되고, 
`MySQL` 은 이 파스 트리를 사용해서 쿼리를 수행한다.  

두번째 단계가 첫번째 단계에서 만들어진 파스 트리를 사용해서 아래와 같은 동작을 수행한다. 
- 불필요한 조건의 제거
- 복잡한 연산 단순화
- 여러 테이블을 사용할 경우, 테이블 순서 결정
- 각 테이블에서 어떤 인덱스를 사용할지 결정
- 테이블로 부터 가져온 데이터를 임시 테이블 및 데이터 가공이 필요한지 결정
위와 같은 과정을 수행하는 곳이 바로 옵티마이저이고, 이때 나오는 것이 바로 실행 계획이다.  

세번째 단계는 실행 계획을 바탕으로 스토리지 엔진에 레코드를 읽거나, 테이블을 조인, 정렬 등을 수행한다.  

첫번째와 두번째는 `MySQL` 엔진에서 처리되고, 세번째 단계는 `MySQL` 엔진과 스토리지 엔진이 함께 수행한다. 

![그림 1]({{site.baseurl}}/img/mysql/practice-explain-mysql-architecture.png)


### 옵티마이저
`MySQL` 에서 사용하는 옵티마이저는 대부분 `DBMS` 에서 사용되고 있는 비용 기반 최적화(`Cost based Optimizer`) 이다. 
비용 기반 최적화는 쿼리 수행을 위해 상황에 따른 여러 방법을 만들어 낸다. 
그리고 각 단위 작업에 대한 비용과 대상 테이블의 예측된 통계 정보를 사용해서 각 실행 계획별 비용을 산출하게 된다. 
결과적으로 산출된 실행 계획 중 최소 비용을 사용해서 쿼리를 실행한다. 

### 통계 정보
비용 기반 최적화는 각 테이블에 대한 통계 정보에 의존한다. 
통계 정보가 정확하지 않다면 옵티마이저는 최소 비용을 사용해 쿼리를 수행하기 어려움이 있어, 
비용이 더욱 큰 실행 계획을 선택해 쿼리를 수행하게 된다.  

`MySQL` 은 다른 `DBMS` 에 비해 비교적 단순한 통계 정보를 사용한다. 
대략적인 레코드의 수, 인덱스의 유니크한 값의 수 등에 대한 통계를 쌓고 쿼리 수행에 사용한다. 
`MySQL` 은 동적으로 통계 정보를 관리하므로 주기적으로 통계 정보가 갱신되게 된다. 
필요에 따라 `ANALYZE` 명령을 사용해서 개발자가 직접 통계 정보를 갱신 시킬 수도 있다. 

```sql
ANALYZE TABLE exam;
```  

만약 파티션을 사용하는 테이블의 경우 특정 파티션만 통계 정보를 수집할 수도 있다. 

```sql
ANALYZE TABLE exam ANALYZE PARTITION p2;
```  

임의로 통계정보를 수집할 때 주의할 점은 `ANALYZE` 가 수행되게 되면 `MyISAM` 엔진의 경우 읽기는 가능하지만 쓰기는 불가능하다. 
그리고 `InnoDB` 엔진은 읽기, 쓰기 모두 불가능하기 때문에 서비스 중에는 주의가 필요하다.  

`MyISAM` 엔진은 정확한 키값 분포도를 위해 인덱스 전체를 스캔하므로 많은 시간이 소요된다. 
`InnoDB` 엔진은 인덱스 페이지 중 기본값 8개 정도만 랜덤하게 선택해서 분석하게 된다. 
그리고 `InnoDB` 에서 `innodb_stats_sample_pages` 옵션을 사용해서 분석할 페이지 수를 늘리거나 줄일 수 있다.  


## EXPLAIN
`MySQL` 에서 수행할 쿼리의 실행 계획은 `EXPLAIN` 명령을 사용해서 가능하다. 
그리고 필요에 따라 더 다양한 상세한 실행 계획을 확인할 수 있는 `EXPLAIN EXTENDED`, `EXPLAIN PARTITIONS` 를 사용할 수 있다.  

`EXPLAIN` 사용법은 아주 간단하다. 
아래와 같이 `EXPLAIN` 뒤에 확인하고 싶은 쿼리를 작성해 주면 된다. 

```sql
mysql> explain select count(1) from exam;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | exam  | NULL       | index | NULL          | idx_num | 5       | NULL | 4706820 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```  

`EXPLAIN` 을 실행하면 위와 같이 1개 로우 이상의 결과가 출력된다. 
만약 서브쿼리나 조인, 임시 테이블 등을 사용하게 된 수 만큼 로우는 늘어나게 된다. 
실행 순서는 보편적으로 맨위 로우부터 맨아래 로우 즉 낮은 `id` 부터 높은 `id` 순서이다. 
하지만 `UNION`, 상관 서브 쿼리 등의 수행일 경우 순서는 다를 수 있다.  

그리고 `INSERT`, `UPDATE`, `DELETE` 의 경우 실행 계획은 뽑을 수 없다. 
만약 관련 실행 계획이 궁금하면 `WHERE` 절만 가져와 `SELECT` 쿼리로 만들어 관련 동작을 조회해 볼 수는 있다.  

그리고 아래부터는 `EXPLAIN` 결과로 출력되는 각 컬럼의 의미와 구성에 대해서 알아본다.  

### id
`id` 컬럼은 각 다른 데이터를 조회하는 `select` 쿼리 별로 부여되는 식별값이다. 

```sql
create table table_1 (id int not null auto_increment, value_1 varchar(255) default null, primary key(id));
create table table_2 (id int not null auto_increment, value_2 varchar(255) default null, primary key(id));



```



---
## Reference
[8.8 Understanding the Query Execution Plan](https://dev.mysql.com/doc/refman/8.0/en/execution-plan-information.html))  
[13.8.2 EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.0/en/explain.html)  
[16.11 Overview of MySQL Storage Engine Architecture](https://dev.mysql.com/doc/refman/8.0/en/pluggable-storage-overview.html)  
[[MySQL] MySQL 실행 계획](https://12bme.tistory.com/160)  