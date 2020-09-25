--- 
layout: single
classes: wide
title: "[MySQL 실습] Explain 실행 계획"
header:
  overlay_image: /img/mysql-bg.png
excerpt: '쿼리 성능 파악을 위한 Explain 에 대해 알아보자'
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
비용이 더욱 큰 실행 계획을 선택해 쿼리를 수행하는 등 최적의 방법으로 쿼리를 실행할 수 없게 된다.  

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
`EXPLAIN` 사용법은 아주 간단하다. 
아래와 같이 `EXPLAIN` 뒤에 확인하고 싶은 쿼리를 작성해 주면 된다. 

```sql
mysql> explain select count(1) from exam;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | exam  | NULL       | index | NULL          | idx_num | 5       | NULL | 4706820 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
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
아래와 같이 `select` 한개로 이뤄진 쿼리는 아이디가 한개만 나오게 된다. 

```sql
mysql> explain select * from tb_1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_1  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
```  

`from` 절에 사용된 서브 쿼리는 아이디가 생성되지 않는다. 

```sql
mysql> explain select * from (select * from tb_1) as t;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_1  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
```  

`select` 절에 사용된 서브 쿼리는 사용된 개수만큼 아이디가 생성된다. 

```sql
mysql> explain select (select count(*) from tb_1) + (select count(*) from tb_2) as tt;
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+----------------+
|  1 | PRIMARY     | NULL  | NULL       | NULL  | NULL          | NULL        | NULL    | NULL | NULL |     NULL | No tables used |
|  3 | SUBQUERY    | tb_2  | NULL       | index | NULL          | PRIMARY     | 4       | NULL |    3 |   100.00 | Using index    |
|  2 | SUBQUERY    | tb_1  | NULL       | index | NULL          | idx_tb_2_id | 5       | NULL |    6 |   100.00 | Using index    |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+----------------+

mysql> explain select (select count(*) from tb_1) + (select count(*) from tb_2) + (select count(*) from tb_1) as tt;
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+----------------+
|  1 | PRIMARY     | NULL  | NULL       | NULL  | NULL          | NULL        | NULL    | NULL | NULL |     NULL | No tables used |
|  4 | SUBQUERY    | tb_1  | NULL       | index | NULL          | idx_tb_2_id | 5       | NULL |    6 |   100.00 | Using index    |
|  3 | SUBQUERY    | tb_2  | NULL       | index | NULL          | PRIMARY     | 4       | NULL |    3 |   100.00 | Using index    |
|  2 | SUBQUERY    | tb_1  | NULL       | index | NULL          | idx_tb_2_id | 5       | NULL |    6 |   100.00 | Using index    |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+----------------+
```  

조인을 하게되면 같은 조인 테이블 수만큼 같은 아이디가 부여된다.  

```sql
mysql> explain select * from tb_1 t1, tb_2 t2 where t1.tb_2_id = t2.id;
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key         | key_len | ref  | rows | filtered | Extra                                      |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | index | idx_tb_2_id   | idx_tb_2_id | 5       | NULL |    6 |   100.00 | Using index                                |
|  1 | SIMPLE      | t2    | NULL       | ALL   | PRIMARY       | NULL        | NULL    | NULL |    3 |    33.33 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+-------+---------------+-------------+---------+------+------+----------+--------------------------------------------+
```  

### select_type
`id` 에 해당하는 `select` 쿼리가 어떤 타입인지 표시되는 컬럼이다.  

#### SIMPLE
`union` 이나 서브쿼리는 사용하지 않은 일반적인 `select` 쿼리에 표시되는 값이다. 

```sql
mysql> explain select * from tb_1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_1  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+

mysql> explain select * from (select * from tb_1) tb1, (select * from type_1) type1;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------------------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                         |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------------------------+
|  1 | SIMPLE      | type_1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL                          |
|  1 | SIMPLE      | tb_1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | Using join buffer (hash join) |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------------------------+

mysql> explain select * from tb_1 tb1, type_1 typ1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                         |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------+
|  1 | SIMPLE      | typ1  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL                          |
|  1 | SIMPLE      | tb1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------------+
```  

#### PRIMARY
`union` 이나 서브쿼리가 포함된 `select` 쿼리에서 가장 바깥쪽 `select` 쿼리를 표시하는 값이다. 
전체 `select` 쿼리에서 `PRIMARY` 는 단 한개만 존재한다. 

```sql
mysql> explain select (select count(*) from type_1), (select count(*) from type_2) from tb_1;
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | tb_1   | NULL       | index | NULL          | PRIMARY | 4       | NULL |   12 |   100.00 | Using index |
|  3 | SUBQUERY    | type_2 | NULL       | index | NULL          | PRIMARY | 4       | NULL |    3 |   100.00 | Using index |
|  2 | SUBQUERY    | type_1 | NULL       | index | NULL          | PRIMARY | 4       | NULL |    6 |   100.00 | Using index |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```  

#### UNION
`union` 을 수행하는 `select` 쿼리 중 첫번째를 제외한 `select` 쿼리에 표시되는 값이다. 

```sql
mysql> explain
    -> select
    ->  *
    -> from (
    ->  (select * from tb_1)
    ->  union all
    ->  (select * from type_1)
    ->  union all
    ->  (select * from type_2)
    -> ) as t;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   21 |   100.00 | NULL  |
|  2 | DERIVED     | tb_1       | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | NULL  |
|  3 | UNION       | type_1     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL  |
|  4 | UNION       | type_2     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
```  

실행 계획을 보면 `union all` 을 수행하고 서브 쿼리 중 `tb_1`, `type_1`, `type_2` 테이블에서 첫번째를 제외한 `type_1`, `type_2` 에 `UNION` 값이 표시된 것을 확인할 수 있다.  

#### DEPENDENT UNION
`union` 쿼리에서 나타내는 값으로, 
`union` 에 사용되는 쿼리가 외부에 영향을 받는 경우 사용된다.  

```sql
mysql> explain
    -> select
    ->  *
    -> from
    ->  tb_1 t1
    -> where
    ->  t1.type in (
    ->   (select id from type_1 as type1 where t1.type = type1.id)
    ->   union
    ->   (select id from type_2 as type2 where t1.type = type2.id)
    ->  );
+----+--------------------+------------+------------+--------+---------------+---------+---------+------------------------+------+----------+--------------------------+
| id | select_type        | table      | partitions | type   | possible_keys | key     | key_len | ref                    | rows | filtered | Extra                    |
+----+--------------------+------------+------------+--------+---------------+---------+---------+------------------------+------+----------+--------------------------+
|  1 | PRIMARY            | t1         | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                   |   12 |   100.00 | Using where              |
|  2 | DEPENDENT SUBQUERY | type1      | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | db_select_type.t1.type |    1 |   100.00 | Using where; Using index |
|  3 | DEPENDENT UNION    | type2      | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | db_select_type.t1.type |    1 |   100.00 | Using where; Using index |
| NULL | UNION RESULT       | <union2,3> | NULL       | ALL    | NULL          | NULL    | NULL    | NULL                   | NULL |     NULL | Using temporary          |
+----+--------------------+------------+------------+--------+---------------+---------+---------+------------------------+------+----------+--------------------------+
```  

`where` 절에서 `union` 을 수행하는 쿼리에서 외부 테이블의 값인 `tb_1.type` 을 참조하고 있다. 
이러한 형식의 쿼리에서 `DEPENDDENT UNION` 이 사용된다. 
이렇게 `DEPENDENT` 가 붙은 쿼리는 외부 쿼리에 외존하게 된다. 
그로 인해 서브 쿼리의 경우 일반적으로 외부 쿼리보다 먼저 실행되지만, 
`DEPENDENT` 값이 있는 상태라면 외부 쿼리에 의존하게 되므로 먼저 실행될 수 없고 비효율적인 경우가 많다.  

#### UNION RESULT
`union` 을 수행한 결과를 담아두는 임시 테이블을 나타내는 값이다. 
그리고 `UNION RESULT` 의 경우 실제 쿼리에 존재하지 않는 쿼리이기 때문에 `id` 는 부여되지 않는다.  

```sql
mysql> explain
    -> (select
    ->  *
    -> from type_1)
    -> union
    -> (select
    ->  *
    -> from type_2);
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
| id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
|  1 | PRIMARY      | type_1     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL            |
|  2 | UNION        | type_2     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
```  

실행 계획의 마지막 줄을 보면 `UNION RESULT` 가 있다. 
`table` 컬럼 값이 `<union1,2>` 로 나와 있는데 이는 `id` 가 1번, 2번인 쿼리를 `union` 했다는 것을 의미한다.   

#### SUBQUERY
먼저 서브 쿼리의 위치에 따른 분류는 아래와 같다. 
- `Nested Query`(중첩된 쿼리) : `select` 절에 위치하는 서브 쿼리
- `Sub Query`(서브 쿼리) : `where` 절에 사용된 서브 쿼리
- `Derived`(파생 테이블) : `from` 절에 사용된 서브 쿼리

서브 쿼리가 반환하는 값에 따른 분류는 아래와 같다. 
- `Scalar SubQery` : 하나의 값만 반환하는 서브 쿼리
- `Row Sub Query` : 컬럼수와 관계없이 로우 한개를 반환하는 서브 쿼리

`select_type` 컬럼에 `SUBQUERY` 값으로 표시되는 경우는 `Derived` 를 제외한 모든 경우이다. 
`SUBQUERY` 로 표시되는 `select` 문은 외부 쿼리에 영향을 받지 않기 때문에 처음 한번만 실행되고, 그 결과는 캐싱을 통해 사용된다. 

```sql
mysql> explain select (select count(*) from tb_1) from type_1;
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | type_1 | NULL       | index | NULL          | PRIMARY | 4       | NULL |    6 |   100.00 | Using index |
|  2 | SUBQUERY    | tb_1   | NULL       | index | NULL          | PRIMARY | 4       | NULL |   12 |   100.00 | Using index |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+

mysql> explain
    -> select
    ->  *
    -> from tb_1
    -> where
    ->  (select value_1 from type_1) = type;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | tb_1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |    10.00 | Using where |
|  2 | SUBQUERY    | type_1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL        |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
```  

#### DEPENDENT SUBQUERY
서브 쿼리가 외부 쿼리에 의존하는 경우 표시된다. 
`DEPENDENT SUBQUERY` 는 한번만 캐싱되는 것이 아닌, 외부 쿼리의 컬럼 단위로 캐싱해서 사용한다. 

```sql
mysql> explain
    -> select
    ->  (
    ->   select count(*)
    ->   from type_1 type1, type_2 type2
    ->   where type1.id = tb1.type or type2.id = tb1.type
    ->  )
    -> from tb_1 tb1;
+----+--------------------+-------+------------+-------+---------------+---------+---------+------+------+----------+---------------------------------------------------------+
| id | select_type        | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                                                   |
+----+--------------------+-------+------------+-------+---------------+---------+---------+------+------+----------+---------------------------------------------------------+
|  1 | PRIMARY            | tb1   | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |   12 |   100.00 | NULL                                                    |
|  2 | DEPENDENT SUBQUERY | type2 | NULL       | index | PRIMARY       | PRIMARY | 4       | NULL |    3 |   100.00 | Using index                                             |
|  2 | DEPENDENT SUBQUERY | type1 | NULL       | index | PRIMARY       | PRIMARY | 4       | NULL |    6 |   100.00 | Using where; Using index; Using join buffer (hash join) |
+----+--------------------+-------+------------+-------+---------------+---------+---------+------+------+----------+---------------------------------------------------------+
```  

외부 쿼리 의존으로 외부 쿼리가 먼저 수행되고 나서 서브 쿼리가 수행되기 때문에 `SUBQUERY` 보다 성능적으로 좋지 않은 경우가 많다.  

#### DERIVED
`from` 절에 서브 쿼리가 사용되면서 임시 테이블 생성이 필요한 경우 사용되는 값이다. 
`DERIVED` 테이블은 파생 테이블이라고도 불리며, 해당 테이블에서는 인덱스를 사용할 수 없다. 

```sql
mysql> explain
    -> select *
    -> from ((select * from type_1) union all (select * from type_2)) as t where id > 2;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    9 |    33.33 | Using where |
|  2 | DERIVED     | type_1     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | NULL        |
|  3 | UNION       | type_2     | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL        |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------------+
```  

`id` 1번 `table` 컬럼에 `derived2` 라는 값은 2번에서 생성된 파생 테이블을 사용한다는 의미이고, 
2번은 3번과 `union` 된 파생 테이블이다. 
결과적으로 1번에서 `where` 문은 파생 테이블에서 `id` 컬럼으로 조건 검색을 하기 때문에, 
`type_1`, `type_2` 테이블의 `id` 컬럼에 인덱스가 생성돼 있더라도 사용할 수 없다. 
이런 결과로 실행 계획에서 `DERIVED` 가 있다면 조인과 같은 다른 방법에 대해 고민이 필요하다. 

#### UNCACHEABLE SUBQUERY
앞서 서브 쿼리의 경우 캐싱을 사용된다고 설명했었다. 
`UNCACHEABLE SUBQUERY` 는 서브 쿼리 캐싱이 불가능한 경우 사용되는 값이다. 
서브 쿼리 캐싱이 불가능한 경우는 아래와 같다. 
- `NOT DETERMINISTIC` 속성의 프로시저 혹은 정의 함수가 서브 쿼리에 사용된 경우
- `UUID()`, `RAND()` 와 같이 실행마다 결과가 다른 함수가 서브 쿼리에 사용된 경우

```sql
mysql> explain
    -> select *
    -> from tb_1 tb1
    -> where
    ->  tb1.type = (select floor(1 + rand() * 5));
+----+----------------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type          | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+----------------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | PRIMARY              | tb1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |   100.00 | Using where    |
|  2 | UNCACHEABLE SUBQUERY | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+----------------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
```  

`where` 절의 서브 쿼리에서 `floor(1 + rand() * 5)` 을 통해 랜덤성 숫자를 생성해 비교하고 있다. 
이런 경우 서브 쿼리 결과를 캐싱할 수 없기 때문에 `tb_1` 의 로우 수만큼 실제로 실행하게 된다.  

#### UNCACHEABLE UNION
`union` 결과를 캐싱할 수 없을때 사용되는 값이다. 

```sql
mysql> explain
    -> select(
    ->  (select id from tb_1 where id = 1)
    ->  union
    ->  (select floor(1 + rand() * 1) as id)
    -> ) as t;
+----+-------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
| id | select_type       | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra           |
+----+-------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY           | NULL       | NULL       | NULL  | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | No tables used  |
|  2 | SUBQUERY          | tb_1       | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | Using index     |
|  3 | UNCACHEABLE UNION | NULL       | NULL       | NULL  | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | No tables used  |
| NULL | UNION RESULT      | <union2,3> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+-------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
```  

### table
실행 계획은 `select` 쿼리를 기준으로 `id` 를 부여하고 실제 로우는 테이블을 기준으로 한다. 
아래와 같이 테이블을 사용하지 않거나 `DUAL` 을 사용한 경우 `NULL` 로 표시된다. 

```sql
mysql> explain select 1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+

mysql> explain select 1 from dual;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
```  

앞서 언급했던 것처럼 `table` 컬럼에 `<derived>`, `<union>` 으로 나오는 테이블은 임시 테이블이다. 

```sql
mysql> explain
    -> select
    ->  *
    -> from
    ->  (
    ->    (select * from type_1)
    ->    union
    ->    (select * from type_2)
    ->  ) as type,
    ->  tb_1 tb1
    -> where
    ->  tb1.id = type.id;
+----+--------------+------------+------------+--------+---------------------+---------+---------+---------+------+----------+-----------------+
| id | select_type  | table      | partitions | type   | possible_keys       | key     | key_len | ref     | rows | filtered | Extra           |
+----+--------------+------------+------------+--------+---------------------+---------+---------+---------+------+----------+-----------------+
|  1 | PRIMARY      | <derived2> | NULL       | ALL    | <auto_distinct_key> | NULL    | NULL    | NULL    |    9 |   100.00 | NULL            |
|  1 | PRIMARY      | tb1        | NULL       | eq_ref | PRIMARY             | PRIMARY | 4       | type.id |    1 |   100.00 | NULL            |
|  2 | DERIVED      | type_1     | NULL       | ALL    | NULL                | NULL    | NULL    | NULL    |    6 |   100.00 | NULL            |
|  3 | UNION        | type_2     | NULL       | ALL    | NULL                | NULL    | NULL    | NULL    |    3 |   100.00 | NULL            |
| NULL | UNION RESULT | <union2,3> | NULL       | ALL    | NULL                | NULL    | NULL    | NULL    | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+--------+---------------------+---------+---------+---------+------+----------+-----------------+
```  

위 전체 쿼리의 실행 순서를 나열하면 아래와 같다.
1. 1번은 `<derived2>` 이라는 임시 테이블을 사용하는데, 이는 `id` 2번에서 만들어진다. 
1. 2번은 `type_1` 테이블과 3번 `type_2` 테이블을 `union` 해서 `DEDRVIED` 테이블을 만든다. 
1. 2번과 3번을 조인한 파생 테이블은 제일 아래 `id` 가 `NULL` 인 임시 테이블에서 저장한다. 
1. 1번은 만들어진 파생 테이블을 사용해서 두 번째 1번인 `tb_1` 과 조인을 수행한다.  

### type
`type` 은 실행 계획에서 테이블을 어떤 방법으로 읽었는지 의미하는 컬럼이다. 
테이블은 읽는 방법에는 테이블 전체를 읽거나, 인덱스를 사용하는 등 다양한 방법이 있다. 
데이터가 많을 수록 그 데이터를 읽는 방법에 따라 성능 차이가 확연하기 때문에 필수적으로 확인해야할 컬럼 중하나이다.  

[MySQL 공식 문서](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain-join-types)
를 보면 `type` 에 대한 설명이 조인 타입인 것을 확인할 수 있다. 
`MySQL` 에서는 실제로 테이블 하나를 읽는 처리도 조인처럼 수행하기 때문에 이러한 설명으로 작성한 것 같다. 
이해 측면에서 테이블 조인 혹은 탐색(조회) 방법으로 해석 하더라도 큰 문제는 없어보인다.   

`type` 에 존재하는 값을 먼저 모두 나열하면 아래와 같다. 
1. `system`
1. `const`
1. `eq_ref`
1. `ref`
1. `fulltext`
1. `ref_or_null`
1. `index_merge`
1. `unique_subquery`
1. `index_subquery`
1. `range`
1. `index`
1. `ALL`

`MySQL` 에서 사용하는 우선순위와 성능에 대한 순위도 위 순서와 동일하다. 
`ALL` 을 제외하곤 모두 인덱스를 사용하는 방법이다. 
하지만 인덱스를 사용하더라도 어떠한 방법이나 범위로 인덱스를 사용하냐에 따라 성능차이가 발생할 수 있다. 
이를 초록색, 파란색, 주황색, 빨간색을 사용해서 상태도로 나타내면 아래와 같다. 
초록색, 파란색일 경우 성능적으로 나쁘지 않다고 판단할 수 있는 상태이다.  

![그림 1]({{site.baseurl}}/img/mysql/practice_explain_type_1.png)

#### system
테이블이 비어있거나 로우가 한개만 존재하는 경우 참조 방법이다. 
`InnoDB` 에서는 사용되지 않고, `MyISAM` 이나 `Memory` 엔진에서만 사용가능하다. 

```sql
mysql> explain select * from tb_isam;
table tb+----+-------------+---------+------------+--------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table   | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+---------+------------+--------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_isam | NULL       | system | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+--------+---------------+------+---------+------+------+----------+-------+

mysql> alter table tb_isam engine=InnoDB;
Query OK, 1 row affected (0.08 sec)
Records: 1  Duplicates: 0  Warnings: 0

mysql> explain select * from tb_isam;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_isam | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
```  

#### const
`Primary` 혹은 `Unique` 키를 사용해서 조건을 사용했을때 로우 1개만 조회되는 경우 사용되는 값이다. 
이를 유니크 인덱스 스캔(`Unique Index Scan`) 이라고 부르기도 한다.  

`type_group` 이라는 테이블의 `Pirmary` 키는 `type_1` 과 `type_2` 로 구성돼 있다.  

```sql
mysql> show index from type_group where Key_name = 'PRIMARY';
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table      | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| type_group |          0 | PRIMARY  |            1 | type_1      | A         |           6 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| type_group |          0 | PRIMARY  |            2 | type_2      | A         |          18 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
```  

`const` 는 `Primary` 를 구성하는 모든 컬럼을 조건으로 사용해야 한다. 

```sql
mysql> explain select * from type_group where type_1 = 1 and type_2 = 1;
+----+-------------+------------+------------+-------+---------------+---------+---------+-------------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref         | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | type_group | NULL       | const | PRIMARY       | PRIMARY | 8       | const,const |    1 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------------+------+----------+-------+
```  

만약 하나의 컬럼이라도 제외한다면 다른 접근 방법을 사용하게 된다.

```sql
mysql> explain select * from type_group where type_1 = 1;
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys     | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | type_group | NULL       | ref  | PRIMARY,idx_value | PRIMARY | 4       | const |    3 |   100.00 | NULL  |
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
```  

서브 쿼리를 사용하더라도 서브쿼리가 상수로 치환 될수 있다. 

```sql
mysql> explain select * from type_1 where id = (select type from tb_1 where id = 1);
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | PRIMARY     | type_1 | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
|  2 | SUBQUERY    | tb_1   | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+--------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
```  

#### eq_ref
조인을 수행하는 쿼리에서 표시되는 값으로, 
조인에서 `Primary`, `Unique` 키를 사용해서 조인 조건을 수행할 때 사용된다. 
조인 동작에서 `eq_ref` 가 적용되는 조건은 `const` 와 대부분 동일하다. 
즉, 조인 대상이되는 n개의 테이블에서 2번재 테이블 부터 조인 조건으로 검색할때 1개의 로우와 매칭 되야 한다. 
또한 조인 조건으로 사용되는 컬럼에는 `not null` 제약조건이 필요하다. 

```sql
mysql> show index from type_1;
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table  | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| type_1 |          0 | PRIMARY  |            1 | id          | A         |           0 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+--------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+

mysql> explain select * from tb_1 tb1, type_1 type1 where tb1.type = type1.id;
+----+-------------+-------+------------+--------+---------------+---------+---------+------------------+------+----------+-------------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref              | rows | filtered | Extra       |
+----+-------------+-------+------------+--------+---------------+---------+---------+------------------+------+----------+-------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL    | idx_type      | NULL    | NULL    | NULL             |    1 |   100.00 | Using where |
|  1 | SIMPLE      | type1 | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | db_type.tb1.type |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+--------+---------------+---------+---------+------------------+------+----------+-------------+
```  

`type_1` 테이블의 `Pirmary` 키가 `id` 컬럼인 상태에서 첫번째 조인 테이블은 `tb_1`, 두번째는 `type_1` 이다. 
`tb_1.type` 컬럼이 `type_1.id` 컬럼의 값을 가지고 있기 때문에 조건에 의해 `type_1` 테이블에서 로우 1개씩과 매칭이 가능하므로 `eq_ref` 가 출력된다.  

#### ref
인덱스 키 종류에 상관없이 동등(`Equal`) 조건으로 검색할대 사용되는 값이다. 
조건에 대한 결과가 로우 1개라는 보장이 없기 때문에 `const`, `eq_ref` 보다는 느리지만 그래도 빠른 조회 타입이다. 

```sql
mysql> show index from type_group;
+------------+------------+-----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table      | Non_unique | Key_name  | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+------------+------------+-----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| type_group |          0 | PRIMARY   |            1 | type_1      | A         |           6 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| type_group |          0 | PRIMARY   |            2 | type_2      | A         |          18 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| type_group |          1 | idx_value |            1 | value       | A         |          10 |     NULL |   NULL | YES  | BTREE      |         |               | YES     | NULL       |
+------------+------------+-----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+

mysql> explain select * from type_group where type_1 = 1;
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys     | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | type_group | NULL       | ref  | PRIMARY,idx_value | PRIMARY | 4       | const |    3 |   100.00 | NULL  |
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+

mysql> explain select * from type_group where value = '';
+----+-------------+------------+------------+------+-----------------+-----------+---------+-------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys   | key       | key_len | ref   | rows | filtered | Extra       |
+----+-------------+------------+------------+------+-----------------+-----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | type_group | NULL       | ref  | idx_value,value | idx_value | 1023    | const |    1 |   100.00 | Using index |
+----+-------------+------------+------------+------+-----------------+-----------+---------+-------+------+----------+-------------+
```  

`idx_value` 는 `null` 을 허용하지만, 
동등 조건으로 검색했기 대문에 `ref` 가 사용된다. 
그리고 `Primary` 키에서 모든 컬럼에 대한 조건을 사용하지 않은 경우에도 `ref` 가 사용된다.  

#### fulltext
`fulltext` 인덱스 컬럼으로 조건을 구성한 경우 사용되는 값이다. 
조회 쿼리 또한 `fulltext` 인덱스를 사용할 수 있는 `match(<colunm>) against(<condition>)` 을 사용해야 한다. 
전문 검색을 사용하게 될 경우, 해당 컬럼에 더 효율적인 인덱스가 있더라도 `fulltext` 를 사용한다. 
물론 일반적인 조회 쿼리 조건(`<colunm> = ''`) 을 사용하는 경우에는 `const`, `ref` 등 사용이 가능하다.  

```sql
mysql> show index from type_group;
+------------+------------+-----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table      | Non_unique | Key_name  | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+------------+------------+-----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| type_group |          0 | PRIMARY   |            1 | type_1      | A         |           6 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| type_group |          0 | PRIMARY   |            2 | type_2      | A         |          18 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| type_group |          1 | idx_value |            1 | value       | A         |          10 |     NULL |   NULL | YES  | BTREE      |         |               | YES     | NULL       |
| type_group |          1 | value     |            1 | value       | NULL      |          18 |     NULL |   NULL | YES  | FULLTEXT   |         |               | YES     | NULL       |
+------------+------------+-----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+

mysql> explain select * from type_group where match(value) against('value1');
+----+-------------+------------+------------+----------+---------------+-------+---------+-------+------+----------+-------------------------------+
| id | select_type | table      | partitions | type     | possible_keys | key   | key_len | ref   | rows | filtered | Extra
                |
+----+-------------+------------+------------+----------+---------------+-------+---------+-------+------+----------+-------------------------------+
|  1 | SIMPLE      | type_group | NULL       | fulltext | value         | value | 0       | const |    1 |   100.00 | Using where; Ft_hints: sorted |
+----+-------------+------------+------------+----------+---------------+-------+---------+-------+------+----------+-------------------------------+

mysql> explain select * from type_group where value = 'value1';
+----+-------------+------------+------------+------+-----------------+-----------+---------+-------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys   | key       | key_len | ref   | rows | filtered | Extra       |
+----+-------------+------------+------------+------+-----------------+-----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | type_group | NULL       | ref  | idx_value,value | idx_value | 1023    | const |    2 |   100.00 | Using index |
+----+-------------+------------+------------+------+-----------------+-----------+---------+-------+------+----------+-------------+
```  

`type_group.value` 컬럼에는 일반 인덱스(`idx_value`)와 `fulltext` 인덱스가 걸려있다. 
`fulltext` 조건을 사용하게되면 `fulltext` 인덱스를 사용하고, 
동등 조건을 사용하면 일반 인덱스를 사용해서 처리된다.  

#### ref_or_null
`ref` 와 비슷한 접근 방법에서 `NULL` 에 대한 비교 요소가 추가된 값이다. 
인덱스에 `NULL` 값이 포함된 경우 `is null` 로 조회하게 되면 사용된다.  

```sql
mysql> explain select * from type_group where value = 'value1' or value is null;
+----+-------------+------------+------------+-------------+-----------------+-----------+---------+-------+------+----------+--------------------------+
| id | select_type | table      | partitions | type        | possible_keys   | key       | key_len | ref   | rows | filtered | Extra                    |
+----+-------------+------------+------------+-------------+-----------------+-----------+---------+-------+------+----------+--------------------------+
|  1 | SIMPLE      | type_group | NULL       | ref_or_null | idx_value,value | idx_value | 1023    | const |    3 |   100.00 | Using where; Using index |
+----+-------------+------------+------------+-------------+-----------------+-----------+---------+-------+------+----------+--------------------------+
```  

#### index_merge
지금까지 조회방법은 존재하는 인덱스 중 하나의 인덱스를 선택해서 사용했다. 
`index_merge` 는 2개 이상의 인덱스를 사용하는 방법으로, 
여러 인덱스로 탐색한 결과를 머지하는 방식으로 처리된다. 


```sql
mysql> explain select * from type_group where type_1 = 1 or value = 'value1';
+----+-------------+------------+------------+-------------+-------------------------+-------------------+---------+------+------+----------+---------------------------------------------+
| id | select_type | table      | partitions | type        | possible_keys           | key               | key_len | ref  | rows | filtered | Extra                                       |
+----+-------------+------------+------------+-------------+-------------------------+-------------------+---------+------+------+----------+---------------------------------------------+
|  1 | SIMPLE      | type_group | NULL       | index_merge | PRIMARY,idx_value,value | PRIMARY,idx_value | 4,1023  | NULL |    5 |   100.00 | Using union(PRIMARY,idx_value); Using where |
+----+-------------+------------+------------+-------------+-------------------------+-------------------+---------+------+------+----------+---------------------------------------------+
```  

만약 `index_merge` 가 사용된 테이블이라면 `Extra` 컬럼에 표시되는 추가적인 정보를 확인해 볼 수 있다.  

#### unique_subquery
`where` 절에 사용되는 서브 쿼리가 `in` 연산자와 함께 사용될 경우 사용하는 값이다. 
`unique_subquery` 는 `in` 연산자에 사용되는 서브 쿼리의 결과가 중복되는 값이 없을 경우 사용하는 탐색 방법이다. 
최적화가 가능한 경우 `unique_subquery` 조건이더라도 `eq_ref` 로 옵티마이저에 의해 변경될 수 있다. 

```sql
mysql> explain select * from tb_1 where type in (select id from type_1 where value_1 is null) or id > 2;
+----+--------------------+--------+------------+-----------------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type        | table  | partitions | type            | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+--------------------+--------+------------+-----------------+---------------+---------+---------+------+------+----------+-------------+
|  1 | PRIMARY            | tb_1   | NULL       | ALL             | PRIMARY       | NULL    | NULL    | NULL |    1 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | type_1 | NULL       | unique_subquery | PRIMARY       | PRIMARY | 4       | func |    1 |   100.00 | Using where |
+----+--------------------+--------+------------+-----------------+---------------+---------+---------+------+------+----------+-------------+
```  

#### index_subquery
`unique_subquery` 와 마찬가지로 `in` 연산자와 서브 쿼리가 조건으로 있는 상황에서 사용되는 값이다. 
차이점이라면 서브 쿼리의 결과에 중복되는 값이 포함되는 경우 사용하는 방법이다. 
`index_subquery` 는 서브 쿼리의 결과에 중복되는 값이 나올 수 있지만 이를 인덱스를 사용해서 제거 가능한 경우이다. 
최적화가 가능한 경우 `unique_subquery` 조건이더라도 `ref` 로 옵티마이저에 의해 변경될 수 있다. 

```sql
mysql> explain select * from type_1 where id in (select type_1 from type_group where type_1 between 1 and 3) or id > 2;
+----+--------------------+------------+------------+----------------+---------------+---------+---------+------+------+----------+--------------------------+
| id | select_type        | table      | partitions | type           | possible_keys | key     | key_len | ref  | rows | filtered | Extra                    |
+----+--------------------+------------+------------+----------------+---------------+---------+---------+------+------+----------+--------------------------+
|  1 | PRIMARY            | type_1     | NULL       | ALL            | PRIMARY       | NULL    | NULL    | NULL |    1 |   100.00 | Using where              |
|  2 | DEPENDENT SUBQUERY | type_group | NULL       | index_subquery | PRIMARY       | PRIMARY | 4       | func |    3 |    11.11 | Using where; Using index |
+----+--------------------+------------+------------+----------------+---------------+---------+---------+------+------+----------+--------------------------+
```  

#### range
특정 범위 내에서 인덱스를 사용하여 데이터 추출이 가능한 경우로, 
데이터가 크지 않다면 나쁘지 않은 조회 방법이다. 
흔히 다른 용어로 `인덱스 레인지 스캔` 이라고 불리기도 한다. 
조건시 사용 가능한 연산자로는 아래와 같은 것들이 있다. 
- `=`, `<>`
- `>`, `<`, `>=`, `<=`
- `IS NULL`, `<=>`
- `BETWEEN`
- `LIKE`
- `IN`

```sql
mysql> explain select * from tb_1 where id between 1 and 5;
explain +----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_1  | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    5 |   100.00 | Using where |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+

mysql> explain select * from type_group where value like 'u%';
+----+-------------+------------+------------+-------+-----------------+-----------+---------+------+------+----------+--------------------------+
| id | select_type | table      | partitions | type  | possible_keys   | key       | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+------------+------------+-------+-----------------+-----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | type_group | NULL       | range | idx_value,value | idx_value | 1023    | NULL |    1 |   100.00 | Using where; Using index |
+----+-------------+------------+------------+-------+-----------------+-----------+---------+------+------+----------+--------------------------+
```  

#### index
인덱스를 사용하긴 하지만 인덱스 전체를 스캔하는 방법이다. 
다른 말로 인덱스 풀 스캔이라고 한다. 
`range` 는 인덱스의 특정 범위를 사용한다면, `index` 는 인덱스 전체를 사용한다. 
인덱스 테이블이 데이터 테이블보다 작기 때문에, 
테이블 전체를 스캔하는 `ALL` 보다는 빠르지만 개선이 필요한 방법이다.  

`index` 가 사용될 수 있는 상황은 `const`, `range`, `ref`, `eq_ref` 가 사용되지 못할 때 아래 2가지 중 한가지를 만족하는 경우이다. 
1. 인덱스에 포함된 컬럼만으로 처리할 수 있는 쿼리(데이터 파일을 읽을 필요 없는 경우)
1. 인덱스를 이용해 정렬이나 그룹핑 작업이 가능한 쿼리(추가적인 정렬이나 그룹핑이 필요 없는 경우)

```sql
mysql> explain select * from tb_1 order by type limit 1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | tb_1  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+

mysql> explain select type, count(id) from tb_1 group by type order by type;
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_1  | NULL       | index | idx_type      | idx_type | 5       | NULL |    1 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+

mysql> explain select * from type_group order by type_1;
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
|  1 | SIMPLE      | type_group | NULL       | index | NULL          | PRIMARY | 8       | NULL |   18 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------+
```  

#### ALL
데이터 테이블 전체를 스캔하는 경우로 풀 테이블 스캔이라고 한다. 
앞서 설명한 방법 중 어떠한 상황에도 적용되지 않을 때 마지막에 선택되는 방법으로 가장 비효율적이다. 
하지만 잘못 설계된 인덱스를 사용하는 경우에는 인덱스 스캔보다 더 효율적일 수 있다. 

```sql
mysql> explain select * from tb_1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb_1  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+

mysql> explain select * from tb_1 order by type;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | tb_1  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
```  

### possible_keys
의미 그대로 데이터 테이블을 조회할 때 사용 할 수 있는 인덱스의 목록이다. 
옵티마이저는 해당 키 목록 중 비용을 기반으로 인덱스를 선택하게 된다. 


### key
데이터 테이블을 조회할 때 실제로 사용할 인덱스를 표시한다. 

### key_len
탐색을 위해 선택된 키에서 사용된 인덱스의 길이를 표시한다. 
많은 경우 인덱스 키를 다중 컬럼으로 구성해서 사용한다. 
실제로 하나의 컬럼으로 구성된 많은 인덱스를 만드는 것보다. 
다중 컬럼으로 구성된 몇개의 인덱스를 만드는 것이 전체적인 성능에 더 좋은 경우가 많다.  

`type_group` 테이블의 `Primary` 키는 아래와 같이 다중 컬럼으로 구성돼 있다. 

```sql
mysql> show index from type_group where Key_name = 'PRIMARY';
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table      | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| type_group |          0 | PRIMARY  |            1 | type_1      | A         |           6 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| type_group |          0 | PRIMARY  |            2 | type_2      | A         |          18 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+------------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
```  

먼저 구성된 `Primary` 키 컬럼 중 하나의 컬러만 사용해서 조회를 하는 쿼리의 실행 계획은 아래와 같다. 

```sql
mysql> explain select * from type_group where type_1 = 1;
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys     | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | type_group | NULL       | ref  | PRIMARY,idx_value | PRIMARY | 4       | const |    3 |   100.00 | NULL  |
+----+-------------+------------+------------+------+-------------------+---------+---------+-------+------+----------+-------+
```  

`key_len` 필드를 보면 4로 표시되는 것을 확인할 수 있다. 
실제로 사용된 `type_group.type_1` 컬럼은 `int` 타입으로 4바이트의 크기를 사용한다. 
이어서 `Primary` 키의 2개 컬럼을 사용한 쿼리의 실행 계획은 아래와 같다. 

```sql
mysql> explain select * from type_group where type_1 = 1 and type_2 = 1;
+----+-------------+------------+------------+-------+---------------+---------+---------+-------------+------+----------+-------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref         | rows | filtered | Extra |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | type_group | NULL       | const | PRIMARY       | PRIMARY | 8       | const,const |    1 |   100.00 | NULL  |
+----+-------------+------------+------------+-------+---------------+---------+---------+-------------+------+----------+-------+
```  

`int` 형 2개가 사용된 만큼 `key_len` 필드가 8로 늘어난 것을 확인 할 수 있다.  

#### ref
인덱스와 동등 비교하는 컬럼 혹은 상수(`const`) 를 표시한다. 
사용하는 인덱스와 상수를 사용해서 동등 비교를 하면 `const` 라는 문자가 표시되고, 
테이블의 컬럼을 지정한 경우는 테이블과 컬럼명이 표시된다.  

경우에 따라 `ref` 필드에 `func` 라는 문자열이 표시될 수 있다. 
이는 인덱스와 동등 비교하는 값을 그대로 사용하지 않고 별도의 연산을 수행하는 경우를 의미한다.  

아래와 같이 동등 비교하는 값을 그대로 사용하면 테이블명과 컬럼명이 표시된다. 

```bash
mysql> explain select * from tb_1 tb1, type_1 type1 where type1.id = tb1.type;
+----+-------------+-------+------------+------+---------------+----------+---------+-----------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref             | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+----------+---------+-----------------+------+----------+-------+
|  1 | SIMPLE      | type1 | NULL       | ALL  | PRIMARY       | NULL     | NULL    | NULL            |    6 |   100.00 | NULL  |
|  1 | SIMPLE      | tb1   | NULL       | ref  | idx_type      | idx_type | 5       | db_ref.type1.id |    2 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+----------+---------+-----------------+------+----------+-------+
```  

여기서 값에 연산을 추가해서 사용하면 아래와 같이 `ref` 필드가 `func` 로 변경되는 것을 확인 할 수 있다. 

```sql
mysql> explain select * from tb_1 tb1, type_1 type1 where type1.id + 1 = tb1.type;
+----+-------------+-------+------------+------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | type1 | NULL       | ALL  | NULL          | NULL     | NULL    | NULL |    6 |   100.00 | NULL                  |
|  1 | SIMPLE      | tb1   | NULL       | ref  | idx_type      | idx_type | 5       | func |    2 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+----------+---------+------+------+----------+-----------------------+
```  

#### rows
쿼리 수행시 사용하는 로우 수를 나타내는 값이다. 
`rows` 컬럼은 실행 계획의 효율성 판단을 예측을 위해 사용된 통계 정보를 바탕으로 보여준다. 
옵티마이저에서 예측한 값으로 실제 데이터 수와는 다를 수 있다.  

```sql
mysql> explain select * from tb_1 where type = 2;
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_1  | NULL       | ref  | idx_type      | idx_type | 5       | const |    2 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+

mysql> explain select * from tb_1 where type > 2;
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra
  |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | tb_1  | NULL       | range | idx_type      | idx_type | 5       | NULL |    8 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
```  


### Extra
`Extra` 컬럼은 쿼리 실행에 대한 추가적인 정보를 보여준다. 
해당 필드에서 나타나는 값은 실제로 성능 예측과 크게 관련있는 부분이 있다. 
전체 리스트는 [여기](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html#explain-extra-information)
에서 확인 할 수 있다. 

#### Using Index
쿼리 수행시 데이터 파일을 사용하지 않고, 인덱스 테이블에서 모두 처리가능한 경우 표시된다. 
다른 말로 `커버링 인덱스` 라고 한다. 

아래 쿼리는 `type_1.id` 는 `Primary` 키로 인덱스 테이블에서 처리가능 하지만, 
`type_1.value_1` 은 데이터 파일에 접근해야 조회가 가능하므로 `Using where` 가 표시되는 것을 확인 할 수 있다. 

```sql
mysql> explain select id, value_1 from type_1 where id > 2;
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | type_1 | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |   33 |   100.00 | Using where |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```  

위 쿼리 `select` 절에서 인덱스에 해당하지 않는 `value_1` 컬럼을 제외하면, 
데이터 조회 시에 인덱스 테이블만 사용해서 가능하기 때문에 `Using index` 가 표시된다. 

```sql
mysql> explain select id from type_1 where id > 2;
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | type_1 | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |   33 |   100.00 | Using where; Using index |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+--------------------------+
```  

인덱스를 사용한 조회에서 비용이 가장 큰 부분은 데이터 테이블을 참조해 나머지 컬럼을 조회하는 부분임을 기억해야 한다.  


#### Using Where
`Using where` 은 쿼리 수행 결과로 스토리지 엔진에서 받은 데이터를 `MySQL` 엔진이 다시 필터링 할때 표시되는 값이다. 
즉, 스토리지 엔진으로 부터 받은 데이터를 별도 필터링, 가공 없이 클라이언트로 전달 가능하다면 `Using where` 은 표시되지 않는다. 
만약 실행 계획의 `type` 컬럼이 `index`, `ALL` 일 경우 쿼리의 개선이 필요하다.  

`Using where` 의 대표적인 예로는 `where` 조건에서 인덱스 컬럼을 사용해서 데이터를 선별하지만, 
`select` 절에 인덱스에 포함되지 않은 컬럼이 포함된 경우이다. 

```sql
mysql> explain select id, value_1 from type_1 where id > 2;
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | type_1 | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    4 |   100.00 | Using where |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```  

그리고 `where` 조건에 인덱스 컬럼과 인덱스가 걸리지 않은 컬럼으로 조건을 구성하는 경우도 포함된다.

```sql
mysql> explain select id from type_1 where id > 2 and value_1 is null;
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | type_1 | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    4 |    16.67 | Using where |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
```  

#### Using Filesort
쿼리 결과를 정렬할때 가장 좋은 방법은 인덱스를 사용하는 것이다. 
하지만 `order by` 절에서 인덱스를 사용하지 못하는 경우 `Using filesort` 가 표시된다. 
정렬은 메모리 혹은 디스크 상에서 수행되는 정렬을 모두 포함한다. 
조회 결과 데이터가 많은 경우 성능에 큰 영향을 미칠 수 있다. 

```sql
mysql> explain select * from type_1 order by id;
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------+
| id | select_type | table  | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------+
|  1 | SIMPLE      | type_1 | NULL       | index | NULL          | PRIMARY | 4       | NULL |    6 |   100.00 | NULL  |
+----+-------------+--------+------------+-------+---------------+---------+---------+------+------+----------+-------+

mysql> explain select * from type_1 order by value_1;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | type_1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    6 |   100.00 | Using filesort |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+----------------+
```  

첫번째 쿼리 처럼 `Primary` 키인 `id` 컬럼으로 정렬을 수행하면 인덱스를 바탕으로 정렬이 가능하다. 
하지만 두번째 쿼리 처럼 인덱스가 걸리지 않은 컬럼으로 정렬을 수행하기 위해서는 조회 결과를 별도로 정렬 알고리즘을 사용해서 정렬을 수행해줘야 한다.  


#### Using Temporary
쿼리 수행을 위해 중간 결과를 담아두기 위한 임시 테이블(`Temporary table`)을 사용하는 경우 표시된다. 
임시 테이블은 메모리, 디스크 중 하나에 생성될 수 있다.  

```sql
mysql> explain select group_id from tb_1 group by group_id order by min(type);
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+---------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | tb_1  | NULL       | index | idx_group     | idx_group | 5       | NULL |    2 |   100.00 | Using temporary; Using filesort |
+----+-------------+-------+------------+-------+---------------+-----------+---------+------+------+----------+---------------------------------+
```  

위 쿼리는 `group by` 절과 `order by` 절에 사용되는 컬럼이 다르기 때문에 임시 테이블이 사용된다.  

하지만 `MySQL` 내부적으로 임시 테이블을 사용하지만 `Using Temporary` 가 표시되지 않는 경우도 있다. 
- `from` 절에 사용되는 서브 쿼리는 파생 테이블(`Derived table`)은 임시 테이블을 사용한다. 
- `count(distinct(<colunm))` 와 같은 쿼리도 임시 테이블을 사용한다. 
- `union`, `union all` 을 사용하는 쿼리도 임시 테이블을 사용한다.
- 인덱스를 사용하지 않는 `order by` 절의 정렬 또한 정렬을 위해 사용되는 임시 버퍼는 임시 테이블을 사용한다. 


### filter
`MySQL` 엔진에 의해 필터링이 되어 남은 비율(`%`)을 나타내는 값이다. 

```sql
mysql> explain select * from type_group where type_1 > 3 and type_2 > 2;
+----+-------------+------------+------------+-------+-------------------+---------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys     | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+-------+-------------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | type_group | NULL       | range | PRIMARY,idx_value | PRIMARY | 4       | NULL |    9 |    33.33 | Using where |
+----+-------------+------------+------------+-------+-------------------+---------+---------+------+------+----------+-------------+
```  

스토리지 엔진이 쿼리에 해당하는 `rows` 컬럼의 개수인 9개를 `MySQL` 엔진에 전달했고, 
`MySQL` 엔진은 9개 중에서 33.33% 만큼인 3개만 필터링해서 결과로 사용했다는 의미이다. 
이또한 정확한 숫자가 아닌 통계 정보를 바탕으로 산출된 값에 의존한다.  

### partition
쿼리에 사용되는 테이블이 파티션 테이블일 경우, 
사용되는 파티션을 나타내는 컬럼이다. 
아래와 같이 년도를 기준으로 파티션이 구성된 테이블이 있다. 

```sql
create table tb_partition (
    id int not null auto_increment,
    value varchar(255) default null,
    reg_date datetime not null default now(),
    primary key(id, reg_date)
)
partition by range (year(reg_date)) (
    partition p2018 values less than (2019),
    partition p2019 values less than (2020),
    partition p2020 values less than (2021),
    partition p2021 values less than (2022)
);
```  

2019 년도에 해당하는 로우를 조회하는 실행 계획은 아래와 같다. 

```sql
mysql> explain select * from tb_partition where reg_date between '2019-01-01' and '2019-12-31';
+----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table        | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_partition | p2019      | ALL  | NULL          | NULL | NULL    | NULL |   12 |    11.11 | Using where |
+----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
```  

`partition` 컬럼을 확인하면 2019 년도에 해당하는 파티션인 `p2019` 사용 된 것을 확인 할 수 있다. 
좀 더 넓은 범위의 조건을 사용하면 아래와 같다. 

```sql
mysql> explain select * from tb_partition where reg_date between '2019-01-01' and '2020-03-31';
+----+-------------+--------------+-------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table        | partitions  | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------------+-------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb_partition | p2019,p2020 | ALL  | NULL          | NULL | NULL    | NULL |   21 |    11.11 | Using where |
+----+-------------+--------------+-------------+------+---------------+------+---------+------+------+----------+-------------+
```  

`where` 절 조건에 해당하는 `p2019`, `p2020` 파티션이 사용된 것을 확인 할 수 있다. 


---
## Reference
[8.8 Understanding the Query Execution Plan](https://dev.mysql.com/doc/refman/8.0/en/execution-plan-information.html))  
[13.8.2 EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.0/en/explain.html)  
[16.11 Overview of MySQL Storage Engine Architecture](https://dev.mysql.com/doc/refman/8.0/en/pluggable-storage-overview.html)  
[[MySQL] MySQL 실행 계획](https://12bme.tistory.com/160)  