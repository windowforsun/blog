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
toc: true
use_math: true
---  

## 테스트 환경
테스트를 위해 `MySQL 8` 버전을 `Docker` 를 사용해서 구성한다.

```bash
.. 컨테이너 시작 ..
$ docker run --name test-mysql -e MYSQL_ROOT_PASSWORD=root -d -p 3306:3306 mysql:8

.. 컨테이너 접속 ..
$ docker exec -it test-mysql /bin/bash

.. MySQL 접속
$ mysql -u root -proot
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database test;
Query OK, 1 row affected (0.02 sec)

mysql> use test;
Database changed

```  

## Spatial Data(공간 데이터)
`MySQL` 은 [MySQL 5.7.5](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-5.html#mysqld-5-7-5-spatial-support)
부터 `InnoDB Engine` 에서 `Spatial Index` 에 대한 지원을 제공한다. 
`MySQL` 에서 지원하는 `Spatial Data` 의 타입은 아래와 같다. 


Type|구분|설명|데이터 예시
---|---|---|---
Point|Single|좌표 공간의 한 지점|POINT(1 1)
LineString|Single|다수의 Point를 연결한 선분(닫히지 않은 상태)|LINESTRING(1 1, 2 2, 3 3)
Polygon|Single|다수의 Point(LineString) 를 연결해서 닫힌 다각형|POLYGON((1 1, 10 1, 10 10, 1 10, 1 1))
MultiPoint|Collection|여러 개의 Point 의 집합|MULTIPOINT(1 1, 10 10, 20 20)
MultiLineString|Colleciton|여러 개의 LineString 의 집합|MULTILINESTRING((1 1, 2 2, 3 3), (1 10, 1 20, 1 30))
MultiPolygon|Collection|여러 개의 Polygon 의 집합|MULTIPOLYGON(((1 1, 10, 1, 10 10, 1 10, 1 1), (1 7, 3 7, 3 9, 1 9, 1 7)))
GeometryCollection|Collection|모든 공간 타입 데이터의 집합|GEOMETRYCOLLECTION(POINT(1 1), LINESTRING(1 1, 2 2, 3 3), POLYGON((1 1, 10, 1, 10 10, 1 10, 1 1)))

공간 데이터 타입이 가지고 있는 데이터 형식은 `(x, y)` 이고 이는 `(경도(longitude), 위도(latitude))` 순을 의미한다. 


### Spatial Data 크기
공간 데이터 타입의 데이터 크기가 얼마나 되는지 확인해 본다. 
모든 공간 데이터 타입 데이는 `Binary` 로 변환후 저장된다. 

먼저 최소 단위인 `Point` 는 아래와 같이 25 bytes 크기를 갖는다. 

```bash
mysql> SET @point = ST_GeomFromText('Point(1 -127)');
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT LENGTH(@point);
+----------------+
| LENGTH(@point) |
+----------------+
|             25 |
+----------------+
1 row in set (0.00 sec)
```  

> 추후에 다루겠지만 공간 데이터를 추가하기 위해서는 `ST_GeomFromText` 와 같은 함수를 사용해서 좌표 표현을 `Binary` 로 변경해 주어야 한다. 

25 bytes 의 구성은 아래와 같다.  

Component|Size
---|---
Byte order| 1byte
WKB type|4 bytes
X coordinate|8 bytes
Y coordinate|8 bytes

X, Y 의 크기가 각각 8 bytes 이기 때문에 -127 ~ 127 까지의 값이 들어갈 수 있다.  

각 타입에 따라 정리하면 아래와 같다.  

Type|Data|Size
---|---|---
Point|POINT(1 -127)|25 bytes
LineString|LINESTRING(1 1, 2 2)|45 bytes
LineString|LINESTRING(1 1, 2 2, 3 3)|61 bytes
Polygon|POLYGON((1 1, 10 1, 10 10, 1 1))|81 bytes
Polygon|POLYGON((1 1, 10 1, 10 10, 1 10, 1 1))|97 bytes
MultiPoint|MULTIPOINT(1 1)|34 bytes
MultiPoint|MULTIPOINT(1 1, 2 2)|55 bytes
MultiPoint|MULTIPOINT(1 1, 2 2, 3 3)|76 bytes
MultiLineString|MULTILINESTRING((1 1, 2 2))|54 bytes
MultiLineString|MULTILINESTRING((1 1, 2 2, 3 3))|70 bytes
MultiLineString|MULTILINESTRING((1 1, 2 2), (10 10, 20 20))|95 bytes
MultiLineString|MULTILINESTRING((1 1, 2 2, 3 3), (10 10, 20 20, 30 30))|127 bytes
MultiLineString|MULTILINESTRING((1 1, 2 2), (10 10, 20 20), (11 11, 22 22))|136 bytes
MultiPolygon|MULTIPOLYGON(((1 1, 10 1, 10 10, 1 1)))|90 bytes
MultiPolygon|MULTIPOLYGON(((1 1, 10 1, 10 10, 1 10, 1 1)))|106 bytes
MultiPolygon|MULTIPOLYGON(((1 1, 10 1, 10 10, 1 1), (2 2, 20 2, 20 20, 2 2)))|158 bytes
MultiPolygon|MULTIPOLYGON(((1 1, 10 1, 10 10, 1 1), (2 2, 20 2, 20 20, 2 2), (3 3, 30 3, 30 30, 3 3)))|226 bytes



## Spatial Data 사용
간단한 공간 데이터 타입으로 구성된 테이블을 생성해서 `Insert`, `Select` 연산에 대한 기본적인 방법에 대해 알아본다. 

```sql
CREATE TABLE ex_geo_table (
  name VARCHAR(20) PRIMARY KEY,
  point POINT,
  linestring LINESTRING,
  polygon POLYGON,
  multipoint MULTIPOINT,
  multilinestring MULTILINESTRING,
  multipolygon MULTIPOLYGON,
  geometrycollection GEOMETRYCOLLECTION
);
```  

생성한 테이블의 정의는 아래와 같다.  

```bash
mysql> desc ex_geo_table;
+--------------------+-----------------+------+-----+---------+-------+
| Field              | Type            | Null | Key | Default | Extra |
+--------------------+-----------------+------+-----+---------+-------+
| name               | varchar(20)     | NO   | PRI | NULL    |       |
| point              | point           | YES  |     | NULL    |       |
| linestring         | linestring      | YES  |     | NULL    |       |
| polygon            | polygon         | YES  |     | NULL    |       |
| multipoint         | multipoint      | YES  |     | NULL    |       |
| multilinestring    | multilinestring | YES  |     | NULL    |       |
| multipolygon       | multipolygon    | YES  |     | NULL    |       |
| geometrycollection | geomcollection  | YES  |     | NULL    |       |
+--------------------+-----------------+------+-----+---------+-------+
8 rows in set (0.00 sec)

```  

### Insert Spatial Data

데이터 하나를 추가하면 아래와 같다. 

```sql
INSERT INTO ex_geo_table (`name`, `point`, `linestring`, `polygon`, `multipoint`, `multilinestring`, `multipolygon`, `geometrycollection`)
VALUES (
   'exam',
   ST_GeomFromText('POINT(1 -1)'),
   ST_GeomFromText('LINESTRING(1 1, 2 2)'),
   ST_GeomFromText('POLYGON((1 1, 10 1, 10 10, 1 1))'),
   ST_GeomFromText('MULTIPOINT(1 -1, 2 -2)'),
   ST_GeomFromText('MULTILINESTRING((1 1, 2 2), (11 11, 22 22))'),
   ST_GeomFromText('MULTIPOLYGON(((1 1, 10 1, 10 10, 1 1), (2 2, 20 2, 20 20, 2 2)))'),
   ST_GeomFromText('GEOMETRYCOLLECTION(POINT(1 1), LINESTRING(1 1, 2 2), POLYGON((1 1, 10 1, 10 10, 1 1)))')
);
```  

> GeoJSON 형태를 바로 `Spatial Type` 에 추가하고 싶은 경우 아래와 같은 방법으로 가능하다.  
> 
> ```bash
> INSERT INTO ex_geo_tabel(`name`, `point`) VALUES (`json`, ST_GeomFromGeoJSON('{ "type": "Point", "coordinates": [1, -1]}'));
> ```  

### Select Spatial Data

공간 데이터는 `Binary` 형태로 저장되기 때문에 조회할 때 텍스트로 변환이 필요하다. 
변환하는 함수는 `ST_AsText()` 를 사용할 수 있다. 
각 컬럼을 `name` 과 함께 조회 하면 아래와 같다.  

```bash
mysql> select `name`, ST_AsText(`point`), ST_AsText(`linestring`) from ex_geo_table;
+------+--------------------+-------------------------+
| name | ST_AsText(`point`) | ST_AsText(`linestring`) |
+------+--------------------+-------------------------+
| exam | POINT(1 -1)        | LINESTRING(1 1,2 2)     |
+------+--------------------+-------------------------+
1 row in set (0.00 sec)

mysql> select `name`, ST_AsText(`polygon`), ST_AsText(`multipoint`) from ex_geo_table;
+------+-------------------------------+---------------------------+
| name | ST_AsText(`polygon`)          | ST_AsText(`multipoint`)   |
+------+-------------------------------+---------------------------+
| exam | POLYGON((1 1,10 1,10 10,1 1)) | MULTIPOINT((1 -1),(2 -2)) |
+------+-------------------------------+---------------------------+
1 row in set (0.00 sec)

mysql> select `name`, ST_AsText(`multilinestring`), ST_AsText(`multipolygon`) from ex_geo_table;
+------+------------------------------------------+-----------------------------------------------------------+
| name | ST_AsText(`multilinestring`)             | ST_AsText(`multipolygon`)                                 |
+------+------------------------------------------+-----------------------------------------------------------+
| exam | MULTILINESTRING((1 1,2 2),(11 11,22 22)) | MULTIPOLYGON(((1 1,10 1,10 10,1 1),(2 2,20 2,20 20,2 2))) |
+------+------------------------------------------+-----------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select `name`, ST_AsText(`geometrycollection`) from ex_geo_table;
+------+----------------------------------------------------------------------------------+
| name | ST_AsText(`geometrycollection`)                                                  |
+------+----------------------------------------------------------------------------------+
| exam | GEOMETRYCOLLECTION(POINT(1 1),LINESTRING(1 1,2 2),POLYGON((1 1,10 1,10 10,1 1))) |
+------+----------------------------------------------------------------------------------+
1 row in set (0.00 sec)

```  

만약 조회를 `GeoJSON` 타입으로 하고 싶은 경우 `ST_AsGeoJSON()` 함수를 사용할 수 있다.  

```bash
mysql> select `name`, ST_AsGeoJSON(`point`), ST_AsGeoJSON(`linestring`) from ex_geo_table;
+------+-----------------------------------------------+-----------------------------------------------------------------+
| name | ST_AsGeoJSON(`point`)                         | ST_AsGeoJSON(`linestring`)                                      |
+------+-----------------------------------------------+-----------------------------------------------------------------+
| exam | {"type": "Point", "coordinates": [1.0, -1.0]} | {"type": "LineString", "coordinates": [[1.0, 1.0], [2.0, 2.0]]} |
+------+-----------------------------------------------+-----------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select `name`, ST_AsGeoJSON(`polygon`), ST_AsGeoJSON(`multipoint`) from ex_geo_table;
+------+-------------------------------------------------------------------------------------------+-------------------------------------------------------------------+
| name | ST_AsGeoJSON(`polygon`)                                                                   | ST_AsGeoJSON(`multipoint`)                                        |
+------+-------------------------------------------------------------------------------------------+-------------------------------------------------------------------+
| exam | {"type": "Polygon", "coordinates": [[[1.0, 1.0], [10.0, 1.0], [10.0, 10.0], [1.0, 1.0]]]} | {"type": "MultiPoint", "coordinates": [[1.0, -1.0], [2.0, -2.0]]} |
+------+-------------------------------------------------------------------------------------------+-------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select `name`, ST_AsGeoJSON(`multilinestring`), ST_AsGeoJSON(`multipolygon`) from ex_geo_table;
+------+------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------+
| name | ST_AsGeoJSON(`multilinestring`)                                                                      | ST_AsGeoJSON(`multipolygon`)                                                                                                                          |
+------+------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------+
| exam | {"type": "MultiLineString", "coordinates": [[[1.0, 1.0], [2.0, 2.0]], [[11.0, 11.0], [22.0, 22.0]]]} | {"type": "MultiPolygon", "coordinates": [[[[1.0, 1.0], [10.0, 1.0], [10.0, 10.0], [1.0, 1.0]], [[2.0, 2.0], [20.0, 2.0], [20.0, 20.0], [2.0, 2.0]]]]} |
+------+------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select `name`, ST_AsGeoJSON(`geometrycollection`) from ex_geo_table;
+------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| name | ST_AsGeoJSON(`geometrycollection`)                                                                                                                                                                                                                       |
+------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| exam | {"type": "GeometryCollection", "geometries": [{"type": "Point", "coordinates": [1.0, 1.0]}, {"type": "LineString", "coordinates": [[1.0, 1.0], [2.0, 2.0]]}, {"type": "Polygon", "coordinates": [[[1.0, 1.0], [10.0, 1.0], [10.0, 10.0], [1.0, 1.0]]]}]} |
+------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

```  



### Spatial Relation Functions
몇가지 [Spatial Relation Functions](https://dev.mysql.com/doc/refman/5.7/en/spatial-relation-functions-object-shapes.html)
에 대해 다뤄 본다. 
`Spatial Relation Functions` 는 직역하면 공간 관계 함수로 두 `Spatial Data` 를 인자로 사용해서 리턴 값이 `Spatial Data` 가 아닌, 
일반(`Boolean`, `Double`, ..) 을 리턴하는 함수를 의미한다.  

Function|Desc
---|---
ST_Equals(geometry1, geometry2): boolean|geometry1, geometry2가 동일하면 True, 동일하지 않으면 False
ST_Disjoint(geometry1, geometry2)|geometry1, geometry2가 겹치는 곳이 없으면 True, 겹치는 곳이 있다면 False
ST_Contains(geometry1, geometry2)|geometry2가 geometry1 영역안에 포함된다면 True, 포함되지 않는다면 False
ST_Within(geometry1, geometry2)|geometry1이 geometry2 영역안에 포함된다면 True, 포함되지 않는다면 False
ST_Overlaps(geometry1, geometry2)|geometry1, geometry2 영역 간 교집합이 존재한다면 True, 존재하지 않는다면 False
ST_Intersects(geometry1, geometry2)|geometry1, geometry2 영역 간 교집합이 존재한다면 True, 존재하지 않는다면 False
ST_Touches(geometry1, geometry2)|geometry1, geometry2의 경계 부분만 겹친다면 True, 경계 외 영역이 겹치거나 겹치지 않는다면 False
ST_Distance(geometry1, geometry2)|geometry1과 geometry2 사이의 거리를 반환

테스트는 아래 사진과 같은 `red`, `blue`, `green`, `yellow`, `purple` `black` 과 같은 `Polygon` 데이터를 생성해서 진행 한다.  


![그림 1]({{site.baseurl}}/img/mysql/practice-spatial-data-queries-1.png)  


```bash
mysql> set @red = ST_GeomFromText('POLYGON((3 3, 6 3, 6 6, 3 6, 3 3))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @blue = ST_GeomFromText('POLYGON((2 2, 3 2, 3 3, 2 3, 2 2))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @green = ST_GeomFromText('POLYGON((6 3, 7 3, 7 4, 6 4, 6 3))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @yellow = ST_GeomFromText('POLYGON((5 5, 7 5, 7 7, 5 7, 5 5))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @purple = ST_GeomFromText('POLYGON((0 7, 1 7, 1 8, 0 8, 0 7))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @black = ST_GeomFromText('POLYGON((4 4, 5 4, 5 5, 4 5, 4 4))');
Query OK, 0 rows affected (0.00 sec)

```  


#### ST_Equal
geometry1, geometry2가 동일하면 True, 동일하지 않으면 False

```bash
mysql> SELECT ST_Equals(@red, @red), ST_Equals(@blue, @blue), ST_Equals(@green, @green);
+---------------------+---------------------+---------------------+
| ST_Equals(@g1, @g1) | ST_Equals(@g2, @g2) | ST_Equals(@g1, @g2) |
+---------------------+---------------------+---------------------+
|                   1 |                   1 |                   0 |
+---------------------+---------------------+---------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Equals(@red, @yellow), ST_Equals(@red, @green), ST_Equals(@red, @black);
+--------------------------+-------------------------+-------------------------+
| ST_Equals(@red, @yellow) | ST_Equals(@red, @green) | ST_Equals(@red, @black) |
+--------------------------+-------------------------+-------------------------+
|                        0 |                       0 |                       0 |
+--------------------------+-------------------------+-------------------------+
1 row in set (0.00 sec)

```  

#### ST_Disjoint
geometry1, geometry2가 겹치는 곳이 없으면 True, 겹치는 곳이 있다면 False

```bash
mysql> SELECT ST_Disjoint(@purple, @red), ST_Disjoint(@blue, @black), ST_Disjoint(@green, @yellow);
+----------------------------+----------------------------+------------------------------+
| ST_Disjoint(@purple, @red) | ST_Disjoint(@blue, @black) | ST_Disjoint(@green, @yellow) |
+----------------------------+----------------------------+------------------------------+
|                          1 |                          1 |                            1 |
+----------------------------+----------------------------+------------------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Disjoint(@red, @blue), ST_Disjoint(@red, @yellow), ST_Disjoint(@red, @green), ST_Disjoint(@red, @black);
+--------------------------+----------------------------+---------------------------+---------------------------+
| ST_Disjoint(@red, @blue) | ST_Disjoint(@red, @yellow) | ST_Disjoint(@red, @green) | ST_Disjoint(@red, @black) |
+--------------------------+----------------------------+---------------------------+---------------------------+
|                        0 |                          0 |                         0 |                         0 |
+--------------------------+----------------------------+---------------------------+---------------------------+
1 row in set (0.00 sec)

```  

#### ST_Contains
geometry2가 geometry1 영역안에 포함된다면 True, 포함되지 않는다면 False

```bash
mysql> SELECT ST_Contains(@red, @black);
+---------------------------+
| ST_Contains(@red, @black) |
+---------------------------+
|                         1 |
+---------------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Contains(@red, @blue), ST_Contains(@red, @yellow), ST_Contains(@black, @red);
+--------------------------+----------------------------+---------------------------+
| ST_Contains(@red, @blue) | ST_Contains(@red, @yellow) | ST_Contains(@black, @red) |
+--------------------------+----------------------------+---------------------------+
|                        0 |                          0 |                         0 |
+--------------------------+----------------------------+---------------------------+
1 row in set (0.00 sec)
```  

#### ST_Within
geometry1이 geometry2 영역안에 포함된다면 True, 포함되지 않는다면 False

```bash
mysql> SELECT ST_Within(@black, @red);
+-------------------------+
| ST_Within(@black, @red) |
+-------------------------+
|                       1 |
+-------------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Within(@red, @black), ST_Within(@yellow, @red), ST_Within(@green, @red);
+-------------------------+--------------------------+-------------------------+
| ST_Within(@red, @black) | ST_Within(@yellow, @red) | ST_Within(@green, @red) |
+-------------------------+--------------------------+-------------------------+
|                       0 |                        0 |                       0 |
+-------------------------+--------------------------+-------------------------+
1 row in set (0.00 sec)

```  

#### ST_Overlaps
geometry1, geometry2 영역 간 교집합이 존재한다면 True, 존재하지 않는다면 False

```bash
mysql> SELECT ST_Overlaps(@red, @yellow);
+----------------------------+
| ST_Overlaps(@red, @yellow) |
+----------------------------+
|                          1 |
+----------------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Overlaps(@red, @black), ST_Overlaps(@black, @red), ST_Overlaps(@black, @yellow), ST_Overlaps(@red, @blue), ST_Overlaps(@red, @green);
+---------------------------+---------------------------+------------------------------+--------------------------+---------------------------+
| ST_Overlaps(@red, @black) | ST_Overlaps(@black, @red) | ST_Overlaps(@black, @yellow) | ST_Overlaps(@red, @blue) | ST_Overlaps(@red, @green) |
+---------------------------+---------------------------+------------------------------+--------------------------+---------------------------+
|                         0 |                         0 |                            0 |                        0 |                         0 |
+---------------------------+---------------------------+------------------------------+--------------------------+---------------------------+
1 row in set (0.00 sec)

```  

#### ST_Intersects
geometry1, geometry2 영역 간 교집합이 존재한다면 True, 존재하지 않는다면 False

```bash
mysql> SELECT ST_Intersects(@red, @black), ST_Intersects(@black, @red), ST_Intersects(@black, @yellow), ST_Intersects(@red, @blue), ST_Intersects(@red, @green), ST_Intersects(@red, @yellow);
+-----------------------------+-----------------------------+--------------------------------+----------------------------+-----------------------------+------------------------------+
| ST_Intersects(@red, @black) | ST_Intersects(@black, @red) | ST_Intersects(@black, @yellow) | ST_Intersects(@red, @blue) | ST_Intersects(@red, @green) | ST_Intersects(@red, @yellow) |
+-----------------------------+-----------------------------+--------------------------------+----------------------------+-----------------------------+------------------------------+
|                           1 |                           1 |                              1 |                          1 |                           1 |         1                    |
+-----------------------------+-----------------------------+--------------------------------+----------------------------+-----------------------------+------------------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Intersects(@red, @purple), ST_Intersects(@green, @yellow), ST_Intersects(@blue, @black);
+------------------------------+--------------------------------+------------------------------+
| ST_Intersects(@red, @purple) | ST_Intersects(@green, @yellow) | ST_Intersects(@blue, @black) |
+------------------------------+--------------------------------+------------------------------+
|                            0 |                              0 |                            0 |
+------------------------------+--------------------------------+------------------------------+
1 row in set (0.00 sec)

```  

#### ST_Touches
geometry1, geometry2의 경계 부분만 겹친다면 True, 경계 외 영역이 겹치거나 겹치지 않는다면 False

```bash
mysql> SELECT ST_Touches(@red, @blue), ST_Touches(@red, @green), ST_Touches(@black, @yellow);
+-------------------------+--------------------------+-----------------------------+
| ST_Touches(@red, @blue) | ST_Touches(@red, @green) | ST_Touches(@black, @yellow) |
+-------------------------+--------------------------+-----------------------------+
|                       1 |                        1 |                           1 |
+-------------------------+--------------------------+-----------------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Touches(@red, @black), ST_Touches(@red, @yellow), ST_Touches(@purple, @green);
+--------------------------+---------------------------+-----------------------------+
| ST_Touches(@red, @black) | ST_Touches(@red, @yellow) | ST_Touches(@purple, @green) |
+--------------------------+---------------------------+-----------------------------+
|                        0 |                         0 |                           0 |
+--------------------------+---------------------------+-----------------------------+
1 row in set (0.00 sec)

```  

#### ST_Distance
geometry1과 geometry2 사이의 거리를 반환

```bash
mysql> SELECT ST_Distance(@red, @black), ST_Distance(@red, @yellow), ST_Distance(@red, @green), ST_Distance(@red, @blue);
+---------------------------+----------------------------+---------------------------+--------------------------+
| ST_Distance(@red, @black) | ST_Distance(@red, @yellow) | ST_Distance(@red, @green) | ST_Distance(@red, @blue) |
+---------------------------+----------------------------+---------------------------+--------------------------+
|                         0 |                          0 |                         0 |                        0 |
+---------------------------+----------------------------+---------------------------+--------------------------+
1 row in set (0.00 sec)

mysql> SELECT ST_Distance(@red, @purple), ST_Distance(@blue, @yellow), ST_Distance(@purple, @blue), ST_Distance(@green, @black);
+----------------------------+-----------------------------+-----------------------------+-----------------------------+
| ST_Distance(@red, @purple) | ST_Distance(@blue, @yellow) | ST_Distance(@purple, @blue) | ST_Distance(@green, @black) |
+----------------------------+-----------------------------+-----------------------------+-----------------------------+
|           2.23606797749979 |          2.8284271247461903 |           4.123105625617661 |                           1 |
+----------------------------+-----------------------------+-----------------------------+-----------------------------+
1 row in set (0.01 sec)

```  


### Spatial Operator Functions
[Spatial Operator Functions](https://dev.mysql.com/doc/refman/5.7/en/spatial-operator-functions.html)
는 직역하면 공간 연산 함수로 인자의 두 공간을 연산해서 결과로 새로운 공간 객체을 반환하는 함수를 의미한다.  


Function|Desc
---|---
ST_Intersection(geometry1, geometry2): geometry|geometry1, geometry2의 교집합인 공간 객체 리턴
ST_Union(geometry1, geometry2): geometry|geometry1, geometry2의 합집합 공간 객체 리턴
ST_Difference(geometry1, geometry2): geometry|geometry1, geometry2의 차집합 공간 객체 리턴
ST_Buffer(geometry1, double): geometry|geometry1 에서 double 거리만큼 확장된 공간 객체 리턴
ST_Envelop(geometry1): Polygon|geometry1 을 포함하는 최소 MBR인 Polygon 리턴
ST_StartPoint(linestring1): Point|linestring1 의 첫 번째 Point 리턴
ST_EndPoint(linestring1): Point|linestring1 의 마지막 Point 리턴
ST_PointN(linestring1, number): Point|linestring1 에서 number 번째 Point 리턴

테스트는 아래 사진의 `Polygon` 과 `LineString` 을 사용한다. 
`green`, `red`, `yellow` 색 `Polygon`을 사용해서 `ST_Intersection()`, `ST_Union()`, `ST_Difference()` 에 대해서 알아보고, 
`blue`, `black` 색 `LineString` 과 `purple` 색 `Polygon` 을 사용해서 `ST_Envelope()`, `ST_StartPoint()`, `ST_EndPoint()`, `ST_PointN()` 에 대해서 알아본다.  

![그림 1]({{site.baseurl}}/img/mysql/practice-spatial-data-queries-2.png)

```bash
mysql> set @red = ST_GeomFromText('POLYGON((0 0, 3 0, 3 3, 0 3, 0 0))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @green = ST_GeomFromText('POLYGON((-1 2, 1 2, 1 4, -1 4, -1 2))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @yellow = ST_GeomFromText('POLYGON((3 3, 5 3, 5 5, 3 5, 3 3))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @blue_line = ST_GeomFromText('LINESTRING(6 3, 8 3, 9 5)');
Query OK, 0 rows affected (0.00 sec)

mysql> set @purple = ST_GeomFromText('POLYGON((6 3, 9 3, 9 5, 6 5, 6 3))');
Query OK, 0 rows affected (0.00 sec)

mysql> set @black_line = ST_GeomFromText('LINESTRING(11 3, 11 4, 12 5, 13 4, 14 6, 15 5)');
Query OK, 0 rows affected (0.00 sec)

```  

#### ST_Intersection
geometry1, geometry2의 교집합인 공간 객체 리턴

```bash
mysql> SELECT ST_AsText(ST_Intersection(@red, @green)), ST_AsText(ST_Intersection(@red, @yellow)), ST_AsText(ST_Intersection(@green, @yellow));
+------------------------------------------+-------------------------------------------+---------------------------------------------+
| ST_AsText(ST_Intersection(@red, @green)) | ST_AsText(ST_Intersection(@red, @yellow)) | ST_AsText(ST_Intersection(@green, @yellow)) |
+------------------------------------------+-------------------------------------------+---------------------------------------------+
| POLYGON((0 2,1 2,1 3,0 3,0 2))           | POINT(3 3)                                | GEOMETRYCOLLECTION EMPTY                    |
+------------------------------------------+-------------------------------------------+---------------------------------------------+
1 row in set (0.00 sec)
```  

#### ST_Union
geometry1, geometry2의 합집합 공간 객체 리턴

```bash
mysql> SELECT ST_AsText(ST_Union(@red, @green)), ST_AsText(ST_Union(@red, @yellow)), ST_AsText(ST_Union(@green, @yellow));
+--------------------------------------------------+---------------------------------------------------------------+------------------------------------------------------------------+
| ST_AsText(ST_Union(@red, @green))                | ST_AsText(ST_Union(@red, @yellow))                            | ST_AsText(ST_Union(@green, @yellow))                             |
+--------------------------------------------------+---------------------------------------------------------------+------------------------------------------------------------------+
| POLYGON((0 2,0 0,3 0,3 3,1 3,1 4,-1 4,-1 2,0 2)) | MULTIPOLYGON(((3 3,0 3,0 0,3 0,3 3)),((3 3,5 3,5 5,3 5,3 3))) | MULTIPOLYGON(((-1 2,1 2,1 4,-1 4,-1 2)),((3 3,5 3,5 5,3 5,3 3))) |
+--------------------------------------------------+---------------------------------------------------------------+------------------------------------------------------------------+

```  

#### ST_Difference
geometry1, geometry2의 차집합 공간 객체 리턴

```bash
mysql> SELECT ST_AsText(ST_Difference(@red, @green)), ST_AsText(ST_Difference(@green, @red)), ST_AsText(ST_Difference(@red, @yellow)), ST_AsText(ST_Difference(@green, @yellow));
+----------------------------------------+------------------------------------------+-----------------------------------------+-------------------------------------------+
| ST_AsText(ST_Difference(@red, @green)) | ST_AsText(ST_Difference(@green, @red))   | ST_AsText(ST_Difference(@red, @yellow)) | ST_AsText(ST_Difference(@green, @yellow)) |
+----------------------------------------+------------------------------------------+-----------------------------------------+-------------------------------------------+
| POLYGON((0 2,0 0,3 0,3 3,1 3,1 2,0 2)) | POLYGON((1 3,1 4,-1 4,-1 2,0 2,0 3,1 3)) | POLYGON((3 3,0 3,0 0,3 0,3 3))          | POLYGON((-1 2,1 2,1 4,-1 4,-1 2))         |
+----------------------------------------+------------------------------------------+-----------------------------------------+-------------------------------------------+
1 row in set (0.00 sec)

```  

#### ST_Buffer
geometry1 에서 double 거리만큼 확장된 공간 객체 리턴.
`ST_Buffer()` 인자에 `POINT()` 를 주면 `POINT()` 중심으로 반지름이 `n`인 원 `Polygon` 이 결과로 출력된다. 

```bash
mysql> select ST_AsText(ST_Buffer(ST_GeomFromText('POINT(1 1)'), 1));
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ST_AsText(ST_Buffer(ST_GeomFromText('POINT(1 1)'), 1))
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| POLYGON((2 1,1.9807852804032313 1.1950903220161244,1.9238795325112883 1.3826834323650863,1.8314696123025471 1.5555702330195993,1.70710678118655 1.7071067811865452,1.5555702330196048 1.8314696123025436,1.3826834323650925 1.9238795325112856,1.1950903220161309 1.98078528040323,1.0000000000000024 2,0.8049096779838739 1.9807852804032309,0.6173165676349122 1.9238795325112874,0.44442976698039927 1.8314696123025462,0.29289321881345365 1.7071067811865488,0.16853038769745554 1.5555702330196035,0.0761204674887137 1.382683432365091,0.01921471959676979 1.1950903220161293,0 1.0000000000000007,0.01921471959676946 0.8049096779838723,0.07612046748871315 0.6173165676349106,0.16853038769745465 0.4444297669803978,0.29289321881345254 0.2928932188134524,0.44442976698039804 0.16853038769745454,0.6173165676349103 0.07612046748871326,0.8049096779838718 0.01921471959676957,1 0,1.1950903220161284 0.01921471959676957,1.3826834323650898 0.07612046748871326,1.5555702330196022 0.16853038769745476,1.7071067811865475 0.29289321881345254,1.8314696123025453 0.4444297669803978,1.9238795325112867 0.6173165676349102,1.9807852804032304 0.8049096779838718,2 1)) |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

```

#### ST_Envelop
geometry1 을 포함하는 최소 MBR인 Polygon 리턴.
`LineString` 을 포함하는 최소 `Polygon` 을 리턴한다.  

```bash
mysql> SELECT ST_AsText(ST_Envelope(@blue_line)), ST_AsText(@purple);
+------------------------------------+--------------------------------+
| ST_AsText(ST_Envelope(@blue_line)) | ST_AsText(@purple)             |
+------------------------------------+--------------------------------+
| POLYGON((6 3,9 3,9 5,6 5,6 3))     | POLYGON((6 3,9 3,9 5,6 5,6 3)) |
+------------------------------------+--------------------------------+
1 row in set (0.00 sec)

```  

#### ST_StartPoint, ST_EndPoint, ST_PointN
`ST_PointN()` 은 1부터 시작한다. 

```bash
mysql> SELECT ST_AsText(ST_StartPoint(@black_line)), ST_AsText(ST_EndPoint(@black_line)), ST_AsText(ST_PointN(@black_line, 0)), ST_AsText(ST_PointN(@black_line, 3)), ST_AsText(ST_PointN(@black_line, 5));
+---------------------------------------+-------------------------------------+--------------------------------------+--------------------------------------+--------------------------------------+
| ST_AsText(ST_StartPoint(@black_line)) | ST_AsText(ST_EndPoint(@black_line)) | ST_AsText(ST_PointN(@black_line, 0)) | ST_AsText(ST_PointN(@black_line, 3)) | ST_AsText(ST_PointN(@black_line, 5)) |
+---------------------------------------+-------------------------------------+--------------------------------------+--------------------------------------+--------------------------------------+
| POINT(11 3)                           | POINT(15 5)                         | POINT(11 3)                          | POINT(12 5)                          | POINT(14 6)                          |
+---------------------------------------+-------------------------------------+--------------------------------------+--------------------------------------+--------------------------------------+
1 row in set (0.00 sec)

```  

---
## Reference
[Spatial Data Types](https://dev.mysql.com/doc/refman/8.0/en/spatial-types.html)  
[Spatial Function Reference](https://dev.mysql.com/doc/refman/8.0/en/spatial-function-reference.html)
[Geometry Property Functions](https://dev.mysql.com/doc/refman/8.0/en/gis-property-functions.html)
[MySQL 5.7.5 Spatial Data Support](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-5.html#mysqld-5-7-5-spatial-support)
