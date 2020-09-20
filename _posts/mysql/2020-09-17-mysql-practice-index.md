--- 
layout: single
classes: wide
title: "[MySQL 실습] Index 란"
header:
  overlay_image: /img/mysql-bg.png
excerpt: 'MySQL 에서 데이터의 색인 역할을 해주는 인덱스에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
  - MySQL
  - Practice
  - Index
  - Explain
  - B-Tree
  - Docker
toc: true
use_math: true
---  

## Index
인덱스는 지정한 컬럼을 기준으로 메모리영역에 목차를 생성해 조회(`select`) 동작에 대해 성능적인 이점을 부여한다. 
조회 동작에 성능적인 이점을 부여하는 만큼 `insert`, `update,` `delete` 동작에 대해서는 성능저하가 발생할 수 있다. 
즉 테이블에 인덱스가 불필요하게 많아지게 되면 성능 향상을 기대하기는 힘들 수 있다.  

위키피디아에서는 인덱스를 아래와 같이 정의한다. 

>인덱스(영어: index)는 데이터베이스 분야에 있어서 테이블에 대한 동작의 속도를 높여주는 자료 구조를 일컫는다. 
>인덱스는 테이블 내의 1개의 컬럼, 혹은 여러 개의 컬럼을 이용하여 생성될 수 있다. 
>고속의 검색 동작뿐만 아니라 레코드 접근과 관련 효율적인 순서 매김 동작에 대한 기초를 제공한다. 
>인덱스를 저장하는 데 필요한 디스크 공간은 보통 테이블을 저장하는 데 필요한 디스크 공간보다 작다. 
>(왜냐하면 보통 인덱스는 키-필드만 갖고 있고, 테이블의 다른 세부 항목들은 갖고 있지 않기 때문이다.) 
>관계형 데이터베이스에서는 인덱스는 테이블 부분에 대한 하나의 사본이다.


## MySQL Index
MySQL 에서는 사용하는 엔진마다 인덱스 알고리즘이 다를 수 있다. 

Storage Engine|Allowed Index Types
---|---
InnoDB|B-Tree
MyISAM|B-Tree
Memory/Heap|Hash,B-Tree

`InnoDB` 를 사용하는 상태에서 테이블에 대한 인덱스를 생성하고 인덱스 정보를 조회해 보면 아래와 같다. 

```bash
mysql> desc exam;
+----------+--------------+------+-----+-------------------+-------------------+
| Field    | Type         | Null | Key | Default           | Extra             |
+----------+--------------+------+-----+-------------------+-------------------+
| id       | bigint       | NO   | PRI | NULL              | auto_increment    |
| num      | int          | YES  |     | NULL              |                   |
| str1     | varchar(255) | YES  |     | NULL              |                   |
| str2     | varchar(255) | YES  |     | NULL              |                   |
| type     | tinyint(1)   | YES  |     | NULL              |                   |
| group_no | varchar(255) | YES  |     | NULL              |                   |
| datetime | datetime     | YES  |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+----------+--------------+------+-----+-------------------+-------------------+
7 rows in set (0.00 sec)

mysql> create index idx_type on exam (type);
Query OK, 0 rows affected (8.41 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show index from  exam\G
*************************** 1. row ***************************
        Table: exam
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: id
    Collation: A
  Cardinality: 2613696
     Sub_part: NULL
       Packed: NULL
         Null:
   Index_type: BTREE
      Comment:
Index_comment:
      Visible: YES
   Expression: NULL
*************************** 2. row ***************************
        Table: exam
   Non_unique: 1
     Key_name: idx_type
 Seq_in_index: 1
  Column_name: type
    Collation: A
  Cardinality: 2
     Sub_part: NULL
       Packed: NULL
         Null: YES
   Index_type: BTREE
      Comment:
Index_comment:
      Visible: YES
   Expression: NULL
2 rows in set (0.00 sec)
```  

출력결과를 보면 `Index_type` 컬럼의 값이 `BTREE` 인 것을 확인 할 수 있다. 
그리고 `show index from <테이블명>` 을 사용해서 해당 테이블에 설정된 인덱스의 정보를 확인해볼 수 있다. 

## B-Tree Index
`B-Tree` 는 일반적으로 많이 사용되는 데이터베이스 인덱싱 알고리즘이다. 
`B-Tree` 는 `Balanced Tree` 의 약자로 균형잡힌 이진트리를 구조를 가지고 있다. 
`B-Tree` 는 컬럼의 원래 값을 변경시키지 않고 인덱스 구조체 내에서 항상 정렬된 상태로 유지한다.  

`B-Tree` 의 기본적인 구조는 `Root node, Branch node, Leaf node, Data storage` 이다. 

.. 그림 ..

인덱스 트리에서 최상위에 하나만 존재하는 노드를 `Root node` 라고 하고, 
자식이 없는 맨 끝에 있는 노드를 `Leaf node` 라고 한다. 
그리고 `Root node` 와 `Leaf node` 사이에 있는 부모와 자식이 모두 있는 노드를 `Branch node`라고 한다. 
여기서 각 노드에는 실제 테이블에 대한 데이터는 관리하지 않는다. 
테이블에 대한 데이터는 `Leaf node` 가 실제 테이블 레코드를 찾을 수 있는 주소 값을 가지고 있다.  

실제 테이블 레코드는 테이블에 정의된 `Primary` 키를 기준으로 정렬된다. 
실제 정렬은 인덱스의 구조체만 수행하고 `Leaf node` 에서 관리하는 테이블 레코드의 참조값으로 조회가 가능하다. 
그래서 테이블 레코드의 다른 컬럼을 조회할때는 테이블 레코드 참조값으로 데이터 파일을 찾는 방식으로 동작이 수행된다.  

인덱스가 한개의 컬럼이 아닌 두개 이상의 컬럼으로 다중 컬럼 인덱스라면, 
두번째 컬럼은 첫 번째 컬럼에 의존해서 정렬된 구조를 갖는다. 
이는 두번째 컬럼의 정렬순서는 첫번재 컬럼의 같은 값에서만 유요하다는 의미이다.  

`Root node, Branch node, Leaf node` 까지는 메모리 저장소에 저장된 인덱스 정보이다. 
그리고 실제 데이터 레코드를 참조할 수 있는 `Data storage` 는 디스크 저장소 이기때문에, 
인덱스 성능을 향상시킨다는 것은 디스크 저장소 접근을 얼마나 줄이느냐로 볼 수 있다.  

[여기](https://dev.mysql.com/doc/refman/8.0/en/innodb-physical-structure.html)
를 참고하면 인덱스 페이지의 크기는 `16KB` 이다. 
이는 인덱스의 키가 페이지 하나에 몇개까지 들어갈 수 있다로 생각할 수 있다. 
만약 인덱스 키한 개의 크기를 페이지 크기로 나눈 수가 하나의 페이지에 저장될 수 있는 인덱스 크기일 것이다. 
만약 조회 쿼리로 조회해야 할 인덱스의 총 크기가 `16KB` 를 넘어가게 되면, 
인덱스 탐색을 위해서 페이지를 여러번 접근해애 하고 그로 인해 성능저하가 발생할 수 있다.  

## 인덱스 컬럼
테이블에서 인덱스 컬럼을 정하는 기준 중 가장 중요한 것은 카디널리티(`Cadinality`) 이다. 
카디널리티는 컬럼의 중복된 수치를 나타내는 값으로, 
성별은 카디널리티가 낮고, 주민번호나 핸드폰 번호 등은 카디널리티가 높다고 한다. 
즉 인덱스 컬럼을 정할때는 카디널리티가 높은 컬럼을 우선적으로 고려해봐야 한다. 
카디널리티가 낮은 컬럼의 경우 데이터의 중복도가 높은 컬럼이고, 

하나의 컬럼으로 인덱스를 생성하는 경우도 있겠지만, 
다중 컬럼을 인덱스로 생성하는 경우도 많다. 
이 경우 위에서 언급했던 것 처럼 다중 컬럼 인덱스에서 두번째 인덱스는 첫번째 인덱스의 정렬에 크게 의존한다. 
그러므로 다중 컬럼 인덱스는 카디널리티가 높은 순에서 낮은 순으로 컬럼을 배치하는 것이 더욱 효율적이다.  

또한 단일 인덱스를 여러개 두는 것보다는 분리된 단일 인덱스들을 조합해서 다중 컬럼 인덱스를 고려하는 것도 성능을 향상 시킬 수 있는 방법이다.  

추가적으로 업데이트가 빈번하게 일어나지 않는 컬럼, 
`join` 으로 자주 사용되는 컬럼, 
`where` 절에서 자주 사용되는 컬럼은 인덱스에 대해 검토가 필요한 컬림이 될 수 있다.  

그리고 인덱스 컬럼의 크기는 가능한 작게할 수록 성능적인 이점이 있다.  

## 인덱스 조건
최종적으로 인덱스는 조건에 대한 조회를 통해 사용하게 된다. 
하지만 조건을 어떻게 사용하냐에 따라 성능을 위해 설정한 인덱스가 사용될 수도 있고 사용되지 않을 수도 있다.  

먼저 조건절인 `where` 에서 인덱스를 사용할 수 있는 연산자는 아래와 같다. 
- `=`
- `<`, `>`, `>=` ,`<=`
- `between`
- `in`
- `like`
- `is null`

추가적으로 `from` 절의 `join` 이나 그룹 함수인 `min`, `max` 도 인덱스를 사용한다.  

위에서 나열한 연산자를 사용하는 모든 경우에 대해서 인덱스가 사용되는 것이 아닌, 
아래에 대한 부분을 주의해서 사용해야 한다. 
- `like` 연산자를 와일드카드를 포함해서 사용하는 경우 (`str like '%str%`)
- DBMS 에서 풀 스캔이 빠르다고 판단하는 경우
- 조건 절에서 필드를 가공해서 사용하는 경우 (`num * 1000 > 12800`)
- `or` 을 사용하게 되면 비교적 풀 스캔을 사용할 확률이 높아 질 수 있다. 


## 테스트
위에서는 인덱스의 기본 개념과 특징, 주의할 점에 대해 알아봤다. 
알아본 내용을 바탕으로 `Docker` 를 사용해서 환경을 구성하고 몇가지 테스트를 진행해 본다. 
`MySQL 8` 을 사용해서 진행한다. 

`MySQL 8` 부터는 `Query Cache` 기능이 삭제 되었다. 
그래서 `OS` 레벨에서 메모리 캐시를 비우는 동작을 성능 측정할 쿼리 수정전에 실행해 주는 방법으로 진행한다. 

```bash
.. 캐시 비우기 전 ..
root@index-db:/# free -m
              total        used        free      shared  buff/cache   available
Mem:          25563        3238       17906           8        4418       22257
Swap:          7168           0        7168

.. 캐시 비우기 ..
root@index-db:/# echo 3 > /proc/sys/vm/drop_caches

.. 캐시 비운 후 ..
root@index-db:/# free -m
              total        used        free      shared  buff/cache   available
Mem:          25563        3235       21540           8         787       22309
Swap:          7168           0        7168
```  

`buff/cache` 필드 값이 실제로 줄어든것을 확인할 수 있다. 
암묵적으로 성능 측정할 쿼리 수행전에 `MySQL` 호스트에서 `echo 3 > /proc/sys/vm/drop_caches` 을 수행해서 캐시를 비운다는 점을 인지해야 한다.  

테스트를 수행할때 사용하는 `CPU` 코어당 클럭은 `2.90GHz` 이다. 
인덱스 생성과 같은 성능을 측정하고자 하는 쿼리나 수행이 아닌 경우 8개 코어를 사용해서 진행하고, 
측정 대상이 되는 쿼리를 수행할 때는 1개 코어에서 10% 만 사용한다. 
데이터 수에 따른 쿼리 성능을 극명하게 보이기 위함이다.  

### 초기 테이블 및 데이터 구성
사용할 테이블인 `exam` 은 아래와 같은 구성이다. 

```bash
mysql> desc exam;
+----------+--------------+------+-----+-------------------+-------------------+
| Field    | Type         | Null | Key | Default           | Extra             |
+----------+--------------+------+-----+-------------------+-------------------+
| id       | bigint       | NO   | PRI | NULL              | auto_increment    |
| num      | int          | YES  |     | NULL              |                   |
| str1     | varchar(255) | YES  | MUL | NULL              |                   |
| str2     | varchar(255) | YES  |     | NULL              |                   |
| str3     | varchar(255) | YES  |     | NULL              |                   |
| type     | tinyint(1)   | YES  |     | NULL              |                   |
| group_no | varchar(255) | YES  |     | NULL              |                   |
| value    | varchar(255) | YES  |     | NULL              |                   |
| datetime | datetime     | YES  |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+----------+--------------+------+-----+-------------------+-------------------+
9 rows in set (0.00 sec)
```  

초기 상태에 테이블은 `id` 컬럼에 `Primary` 키 외에는 다른 인덱스는 걸려있지 않다. 

```bash
mysql> show index from exam;
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| exam  |          0 | PRIMARY  |            1 | id          | A         |     4838076 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
2 rows in set (0.01 sec)
```  

각 컬럼에 대한 카디널리티를 쿼리로 조회해보면 아래와 같다. 

```bash
mysql> SELECT
    -> COUNT(DISTINCT(id)) AS id,
    -> COUNT(DISTINCT(num)) AS num,
    -> COUNT(DISTINCT(str1)) AS str1,
    -> COUNT(DISTINCT(str2)) AS str2,
    -> COUNT(DISTINCT(str3)) AS str3,
    -> COUNT(DISTINCT(group_no)) AS group_no,
    -> COUNT(DISTINCT(TYPE)) AS TYPE,
    -> COUNT(DISTINCT(DATETIME)) AS datetime
    -> FROM exam;
+---------+-------+-------+-------+------+----------+------+----------+
| id      | num   | str1  | str2  | str3 | group_no | TYPE | datetime |
+---------+-------+-------+-------+------+----------+------+----------+
| 4800000 | 15000 | 10000 | 10000 | 9990 |       10 |    3 |      405 |
+---------+-------+-------+-------+------+----------+------+----------+
1 row in set (41.41 sec)
```  

`exam` 테이블에 현재 총 로우수는 480만개이다. 
그리고 카디널리티가 높은 순으로 컬럼을 나열하면 아래와 같다. 
1. `id`
1. `num`
1. `str1`, `str2`
1. `str3`
1. `datetime`
1. `group_no`
1. `type`

쿼리에 대한 성능 측정은 아래와 같은 과정으로 이뤄진다. 
`set profiling = 1` 으로 프로파일링을 활성화 시킨 후 쿼리를 수행한다. 
그리고 `show profile`, `show profiles` , `show status like 'last_query_cost'` 로 쿼리의 성능을 파악할 수 있다. 
`show profile` 과 `show profiles` 는 쿼리의 소요 시간과 관련된 값을 확인할 수 있고, 
`last_query_cost` 는 직전 쿼리를 수행하기 위해 사용한 비용에 대한 값을 확인 할 수 있다. 

```bash
mysql> set profiling = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> select count(1) from exam;
+----------+
| count(1) |
+----------+
|  4800000 |
+----------+
1 row in set (1.57 sec)

mysql> show profile;
+--------------------------------+----------+
| Status                         | Duration |
+--------------------------------+----------+
| starting                       | 0.000094 |
| Executing hook on transaction  | 0.000006 |
| starting                       | 0.000009 |
| checking permissions           | 0.000007 |
| Opening tables                 | 0.000038 |
| init                           | 0.000010 |
| System lock                    | 0.000011 |
| optimizing                     | 0.000004 |
| statistics                     | 0.000016 |
| preparing                      | 0.000017 |
| executing                      | 1.569709 |
| end                            | 0.000014 |
| query end                      | 0.000008 |
| waiting for handler commit     | 0.000011 |
| closing tables                 | 0.000010 |
| freeing items                  | 0.000045 |
| cleaning up                    | 0.000012 |
+--------------------------------+----------+
17 rows in set, 1 warning (0.00 sec)

mysql> show profiles;
+----------+------------+---------------------------+
| Query_ID | Duration   | Query                     |
+----------+------------+---------------------------+
|        9 | 1.57001875 | select count(1) from exam |
+----------+------------+---------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> select count(1) from exam;
+----------+
| count(1) |
+----------+
|  4800000 |
+----------+
1 row in set (1.57 sec)

mysql> show status like 'last_query_cost';
+-----------------+---------------+
| Variable_name   | Value         |
+-----------------+---------------+
| Last_query_cost | 531490.171892 |
+-----------------+---------------+
1 row in set (0.00 sec)
```  


### 카디널리티에 따른 인덱스 사용 여부 비교
인덱스 컬럼 선정에 있어 가장 중요한 부분은 카디널리티라고 했었다. 
실제로 인덱스 유무에 따라 카디널리티의 값이 얼마나 영항을 주는지 테스트를 수행해본다. 
`exam` 테이블에 있는 컬럼을 사용해서, 카디널리티와 인덱스 유무에 따른 성능을 비교해본다. 

카디널리티가 가장 높은 `num` 컬럼에 대한 테스트 결과는 아래와 같다. 

```bash
.. 인덱스를 사용하지 않은 경우 ..
mysql> select count(value) from exam where num = 1000;
+--------------+
| count(value) |
+--------------+
|          320 |
+--------------+
1 row in set (13.01 sec)

mysql> show profiles;
+----------+-------------+------------------------------------------------+
| Query_ID | Duration    | Query                                          |
+----------+-------------+------------------------------------------------+
|        1 | 13.01022150 | select count(value) from exam where num = 1000 |
+----------+-------------+------------------------------------------------+
1 row in set, 1 warning (0.00 sec)


.. 인덱스를 사용하는 경우 ..
mysql> create index idx_num on exam(num);
Query OK, 0 rows affected (12.87 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> select count(value) from exam where num = 1000;
+--------------+
| count(value) |
+--------------+
|          320 |
+--------------+
1 row in set (0.00 sec)

mysql> show profiles;
+----------+------------+------------------------------------------------+
| Query_ID | Duration   | Query                                          |
+----------+------------+------------------------------------------------+
|        1 | 0.00242000 | select count(value) from exam where num = 1000 |
+----------+------------+------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```  

카디널리티가 가장높은 `num` 은 인덱스를 사용할 경우 5000배 이상 성능이 항샹 되는 것을 확인 할 수 있다.  

다음으로는 카디널리티가 중간정도 되는 `datetime` 컬럼을 사용해서 테스트를 진행해본다. 

```bash
.. 인덱스를 사용하지 않은 경우 ..
mysql> select count(value) from exam where datetime = '2020-09-19 10:00:44';
+--------------+
| count(value) |
+--------------+
|        14424 |
+--------------+
1 row in set (14.49 sec)

mysql> show profiles;
+----------+-------------+----------------------------------------------------------------------+
| Query_ID | Duration    | Query                                                                |
+----------+-------------+----------------------------------------------------------------------+
|        1 | 14.49877025 | select count(value) from exam where datetime = '2020-09-19 10:00:44' |
+----------+-------------+----------------------------------------------------------------------+
1 row in set, 1 warning (0.00 sec)


.. 인덱스를 사용하는 경우 ..
mysql> select count(value) from exam where datetime = '2020-09-19 10:00:44';
+--------------+
| count(value) |
+--------------+
|        14424 |
+--------------+
1 row in set (0.07 sec)

mysql> show profiles;
+----------+------------+----------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                |
+----------+------------+----------------------------------------------------------------------+
|        1 | 0.06220625 | select count(value) from exam where datetime = '2020-09-19 10:00:44' |
+----------+------------+----------------------------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```  

`datetime` 컬럼 역시 인덱스를 사용할때 조회에서 더 빠른 성능을 보여주고 있다.  

마지막으로 카디널리티가 가장 낮은 `type` 컬럼을 사용해서 테스트를 진행해 본다. 

```bash
.. 인덱스를 사용하지 않은 경우 ..
mysql> select count(value) from exam where type = 1;
+--------------+
| count(value) |
+--------------+
|      1600000 |
+--------------+
1 row in set (14.38 sec)

mysql> show profiles;
+----------+-------------+----------------------------------------------------------------------+
| Query_ID | Duration    | Query                                                                |
+----------+-------------+----------------------------------------------------------------------+
|        1 | 14.28991900 | select count(value) from exam where type = 1                         |
+----------+-------------+----------------------------------------------------------------------+
3 rows in set, 1 warning (0.00 sec)


.. 인덱스를 사용하는 경우 ..
mysql> create index idx_type on exam(type);
Query OK, 0 rows affected (10.29 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> select count(value) from exam where type = 1;
+--------------+
| count(value) |
+--------------+
|      1600000 |
+--------------+
1 row in set (3 min 34.48 sec)

mysql> show profiles;
+----------+--------------+----------------------------------------------+
| Query_ID | Duration     | Query                                        |
+----------+--------------+----------------------------------------------+
|        1 | 214.48143800 | select count(value) from exam where type = 1 |
+----------+--------------+----------------------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select count(value) from exam where type = 1;
+----+-------------+-------+------------+------+---------------+----------+---------+-------+---------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows    | filtered | Extra |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+---------+----------+-------+
|  1 | SIMPLE      | exam  | NULL       | ref  | idx_type      | idx_type | 2       | const | 2419038 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+---------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```  

카디널리티가 3밖에 되지 않는 `type` 컬럼의 경우 인덱스를 사용할때 성능이 10배 이상 낮게 나온 것을 확인할 수 있다. 
`explain` 으로 실행계획을 뽑으면 실제로 `idx_type` 을 사용해서 조회가 수행되는 것을 확인할 수있다. 


### 인덱스 크기에 따른 비교
앞서 살펴본 내용준 인덱스의 컬럼 크기에 따라 인덱스 성능의 차이가 보일 수 있다고 했었다. 
이번에는 이 내용에 대해 실제 구성된 데이터를 사용해서 테스트를 진행해본다.  

`str1` 컬럼과 `str2` 컬럼의 카디널리티는 `26181` 로 동일하다. 
하지만 `str1` 과 `str2` 는 평균적으로 갖는 컬럼의 크기가 다르다. 

```bash
mysql> select avg(t.str1len), avg(t.str2len) from (select length(str1) as str1len, length(str2) as str2len from exam) t;
+----------------+----------------+
| avg(t.str1len) | avg(t.str2len) |
+----------------+----------------+
|         6.8890 |        63.8890 |
+----------------+----------------+
1 row in set (1.54 sec)
```  

수치를 뽑아보면 `str2` 컬럼이 `str1` 컬럼보다 10배 정도 큰 것을 확인할 수 있다.  

테스트를 위해 먼저 `str1` 과 `str2` 에 인덱스를 생성해 준다. 

```bash
mysql> create index idx_str1 on exam(str1);
Query OK, 0 rows affected (18.87 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> create index idx_str2 on exam(str2);
Query OK, 0 rows affected (1 min 4.14 sec)
Records: 0  Duplicates: 0  Warnings: 0
```  

인덱스 생성에서 부터 두 컬럼간의 성능적인 차이를 확인해 볼 수 있다. 
그리고 조회시 같은 개수가 나오는 조건으로 `str1`, `str2` 를 각각 조회하면 아래와 같다. 

```bash
mysql> select count(value) from exam where str1 = 'str9000';
+--------------+
| count(value) |
+--------------+
|          480 |
+--------------+
1 row in set (0.00 sec)

mysql> show profiles;
+----------+------------+------------------------------------------------------+
| Query_ID | Duration   | Query                                                |
+----------+------------+------------------------------------------------------+
|        1 | 0.00194825 | select count(value) from exam where str1 = 'str9000' |
+----------+------------+------------------------------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> select count(value) from exam where str2 = 'strstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstr9000';
+--------------+
| count(value) |
+--------------+
|          480 |
+--------------+
1 row in set (0.00 sec)

mysql> show profiles;
+----------+------------+---------------------------------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                                         |
+----------+------------+---------------------------------------------------------------------------------------------------------------+
|        1 | 0.00449425 | select count(value) from exam where str2 = 'strstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstrstr9000' |
+----------+------------+---------------------------------------------------------------------------------------------------------------+
1 row in set, 1 warning (0.00 sec)
```  

컬럼의 길이가 10배 정도 나는 두 컬럼에서 인덱스를 사용할때 성능차이는 2배 정도인 것을 확인 할 수 있다. 


### 다중 컬럼 인덱스 비교
다중 컬럼인덱스를 사용할 때 `B-Tree` 의 구조상 카디널리티가 높은 순으로 컬럼을 배치하는 것이 성능적으로 유리하다고 설명했었다. 
`num`, `group_no`, `type` 컬럼을 사용해서 이를 비교 테스트 해본다.  

테스트를 위해 아래와 같은 인덱스 2개를 만들어 준다. 

```bash
mysql> create index idx_asc on exam(type, group_no, num);
Query OK, 0 rows affected (21.86 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> create index idx_desc on exam(num, group_no, type);
Query OK, 0 rows affected (16.21 sec)
Records: 0  Duplicates: 0  Warnings: 0
```  

조건은 `num = 5780 and group_no = 'group0' and type = 2` 을 사용해서 조회를 한다. 
그리고 조회시 인덱스 지정을 위해 힌트를 사용한다. 
먼저 힌트를 사용 한 쿼리의 실행 계획을 뽑으면 아래와 같이 지정한 인덱스를 사용하는 것을 확인 할 수 있다. 

```bash
mysql> explain select count(value) from exam use index(idx_asc) where num = 5780 and group_no = 'group0' and type = 2;
+----+-------------+-------+------------+------+---------------+---------+---------+-------------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key     | key_len | ref               | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+---------+---------+-------------------+------+----------+-------+
|  1 | SIMPLE      | exam  | NULL       | ref  | idx_asc       | idx_asc | 1030    | const,const,const |  320 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+---------+---------+-------------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select count(value) from exam use index(idx_desc) where num = 5780 and group_no = 'group0' and type = 2;
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref               | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------+
|  1 | SIMPLE      | exam  | NULL       | ref  | idx_desc      | idx_desc | 1030    | const,const,const |  320 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```  

두 쿼리를 사용해서 성능 비교 결과는 아래와 같다. 

```bash
mysql> select count(value) from exam use index(idx_asc) where num = 5780 and group_no = 'group0' and type = 2;
+--------------+
| count(value) |
+--------------+
|          320 |
+--------------+
1 row in set (0.00 sec)

mysql> select count(value) from exam use index(idx_desc) where num = 5780 and group_no = 'group0' and type = 2;
+--------------+
| count(value) |
+--------------+
|          320 |
+--------------+
1 row in set (0.00 sec)

mysql> select count(value) from exam use index(idx_asc) where num = 5780 and group_no = 'group0' and type = 2;
+--------------+
| count(value) |
+--------------+
|          320 |
+--------------+
1 row in set (0.01 sec)

mysql> select count(value) from exam use index(idx_desc) where num = 5780 and group_no = 'group0' and type = 2;
+--------------+
| count(value) |
+--------------+
|          320 |
+--------------+
1 row in set (0.00 sec)

mysql> show profiles;
+----------+------------+---------------------------------------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                                   |
+----------+------------+---------------------------------------------------------------------------------------------------------+
|        1 | 0.00175575 | select count(value) from exam use index(idx_asc) where num = 5780 and group_no = 'group0' and type = 2  |
|        2 | 0.00142800 | select count(value) from exam use index(idx_desc) where num = 5780 and group_no = 'group0' and type = 2 |
|        3 | 0.00173300 | select count(value) from exam use index(idx_asc) where num = 5780 and group_no = 'group0' and type = 2  |
|        4 | 0.00155525 | select count(value) from exam use index(idx_desc) where num = 5780 and group_no = 'group0' and type = 2 |
+----------+------------+---------------------------------------------------------------------------------------------------------+
4 rows in set, 1 warning (0.00 sec)
```  

각각 2번씩 쿼리를 수행했을 때 `idx_desc` 인 카디널리티가 높은 컬럼의 순서대로 인덱스 컬럼을 배치한 인덱스가 성능이 더욱 빠르게 나온 것을 확인 할 수 있다.  


---
## Reference
[인덱스 (데이터베이스)](https://ko.wikipedia.org/wiki/%EC%9D%B8%EB%8D%B1%EC%8A%A4_(%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4))  
[8.3 Optimization and Indexes](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)  
[B-tree](https://en.wikipedia.org/wiki/B-tree)  