--- 
layout: single
classes: wide
title: "[MySQL 개념] MySQL"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'MySQL'
author: "window_for_sun"
header-style: text
categories :
  - MySQL
tags:
    - MySQL
    - Concept
---  









## Reference
[Spring REST Docs](https://docs.spring.io/spring-restdocs/docs/1.1.2.RELEASE/reference/html5/)  
[Asciidoctor User Manual](https://asciidoctor.org/docs/user-manual/)  
[AsciiDoc Syntax Quick Reference](https://asciidoctor.org/docs/asciidoc-syntax-quick-reference/)  
[Spring Rest Docs 적용](http://woowabros.github.io/experience/2018/12/28/spring-rest-docs.html)  
[Spring REST Docs를 사용한 API 문서 자동화](https://velog.io/@kingcjy/Spring-REST-Docs%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%9C-API-%EB%AC%B8%EC%84%9C-%EC%9E%90%EB%8F%99%ED%99%94)  
[Spring REST Docs API 문서를 자동화 해보자](https://www.popit.kr/spring-rest-docs/)  
[Generating documentation for your REST API with Spring REST Docs](https://dimitr.im/spring-rest-docs)  
[Introduction to Spring REST Docs](https://www.baeldung.com/spring-rest-docs)  
[[spring] REST DOcs 사용중 'urlTemplate not found. If you are using MockMvc did you use RestDocumentationRequestBuilders to build the request?' 발생시](https://java.ihoney.pe.kr/517)  





















































# MySQL8 알아보기
### SQL
- New Window function
- Common Table Expression
- NOWAIT and SKIP LOCKED
- Descending Indexes
- Grouping
- Regular Expressions
- Character Sets
- Cost Model
- Histograms

### Json
- Extended syntax
- New function sorting
- Partial updates


## SQL
### Window Functions
- MySQL 8 은 그룹 함수와 매우 비숫한 SQL Window functions 를 제공한다.
- window functions 은 한개의 row에 대해서 계산(COUNT, SUM)과 같은 연산에 사용된다.
- 그룹 함수에서 row 드을 그룹지어 단일 row 로 반환하는 것과 달리, window functions 는 모든 row 에 대해서 함수 적용해 수행한다.
- window functions 로 사용할 수 있는 일반 그룹함수 리스트는 아래와 같다.
	- COUNT
	- SUM
	- AVG
	- MIN
	- MAX
	- BIG_OR
	- BIT_XOR
	- STDDEV_POP
	- STDDEV_SAMP
	- VAR_POP
	- VAR_SAMP
- window functions 에서만 제공하는 함수 리스트는 아래와 같다.
	- RANK
	- DENSE_RANK
	- PERCENT_RANK
	- CUME_DIST
	- NTILE
	- ROW_NUMBER
	- FIRST_VALUE
	- LAST_VALUE
	- NTH_VALUE
	- LEAD
	- LAG

#### SUM 예시 1
	
```sql
CREATE TABLE t(i INT);	# Table
INSERT INTO t VALUES (1),(2),(3),(4);		# Default data
```  
	
- 일반적인 그룹 함수를 사용할 경우 아래와 같다.

	```sql
	SELECT SUM(i) AS SUM FROM t;
	
	sum     
	--------
	10      
	```  
	
- 그룹함수는 하나의 row를 반환하는데 모든 row에 대해서 그룹 함수를 적용 하고 싶을 때 window functions 를 사용하지 않으면 아래와 같다.

	```sql
	SELECT i, (SELECT SUM(i) FROM t) FROM t;
	
	     i  (SELECT SUM(i) FROM t)  
	------  ------------------------
	     1  10                      
	     2  10                      
	     3  10                      
	     4  10  
	```  
	
- window functions 를 사용하면 아래와 같이 사용할 수 있다.

	```sql
	SELECT i, SUM(i) OVER () AS SUM FROM t;
	
	     i  sum     
	------  --------
	     1  10      
	     2  10      
	     3  10      
	     4  10  
	```  
	
- 그룹 함수와 결정적인 차이는 `OVER()` 의 유무이다.
	- `OVER()` 는 window functions 의 키워드 역할을 한다.
	- `OVER` 의 괄호안에는 그룹에 대한 추가적인 문구가 필요하고, 위 예시에서는 비어있기 때문에 전체애 대해서 그룹함수를 적용한다.
	
#### PARTITION 예시
- `PARTITION` 키워드를 통해 그룹 지을 컬럼을 명시하고, window functions 의 결과를 그룹에 따라 적용한다.

```sql
CREATE TABLE sales(employee VARCHAR(50), `date` DATE, sale INT);
 
INSERT INTO sales VALUES ('odin', '2017-03-01', 200),
                         ('odin', '2017-04-01', 300),
                         ('odin', '2017-05-01', 400),
                         ('thor', '2017-03-01', 400),
                         ('thor', '2017-04-01', 300),
                         ('thor', '2017-05-01', 500);
```  

- `employee` 컬럼에 그룹 함수를 적용하면 아래와 같다.

	```sql
	SELECT employee, SUM(sale) FROM sales GROUP BY employee;
	
	employee  SUM(sale)  
	--------  -----------
	odin      900        
	thor      1200       
	```  
	
- `PARTITION` window functions 를 사용하면 아래와 같이 출력 결과를 얻을 수 있다.

	```sql
	SELECT employee, DATE, sale, SUM(sale) OVER (PARTITION BY employee) AS SUM FROM sales;
	
	employee  date          sale  sum     
	--------  ----------  ------  --------
	odin      2017-03-01     200  900     
	odin      2017-04-01     300  900     
	odin      2017-05-01     400  900     
	thor      2017-03-01     400  1200    
	thor      2017-04-01     300  1200    
	thor      2017-05-01     500  1200   
	```  
	
- 아래와 같은 방식으로도 `PARTITION` 을 사용할 수 있다.

	```sql
	SELECT employee, MONTHNAME(DATE), sale, SUM(sale) OVER (PARTITION BY MONTH(DATE)) AS SUM FROM sales;
	
	employee  MONTHNAME(date)    sale  sum     
	--------  ---------------  ------  --------
	odin      March               200  600     
	thor      March               400  600     
	odin      April               300  600     
	thor      April               300  600     
	odin      May                 400  900     
	thor      May                 500  900   
	```  
	
#### ORDER BY 예시
- 미
```sql
SELECT employee, sale, date, SUM(sale) OVER (PARTITION by employee ORDER BY date) AS cum_sales FROM sales;

employee    sale  date        cum_sales  
--------  ------  ----------  -----------
odin         200  2017-03-01  200        
odin         300  2017-04-01  500        
odin         400  2017-05-01  900        
thor         400  2017-03-01  400        
thor         300  2017-04-01  700        
thor         500  2017-05-01  1200       
```  
	
#### Ranking 예시

```sql
CREATE TABLE people (NAME VARCHAR(100), birthdate DATE, sex CHAR(1),
                     citizenship CHAR(2));
 
INSERT INTO people VALUES
("Jimmy Hendrix", "19421127", "M", "US"),
("Herbert G Wells", "18660921", "M", "UK"),
("Angela Merkel", "19540717", "F", "DE"),
("Rigoberta Menchu", "19590109", "F", "GT"),
("Georges Danton", "17591026", "M", "FR"),
("Tracy Chapman", "19640330", "F", "US");
```  

- `ROW_NUMBER()` window function 을 통해 랭킹을 조회할 수 있다. 아래는 생일 오름차순 랭킹이다.

	```sql
	SELECT ROW_NUMBER() OVER (ORDER BY birthdate) AS num,
	       NAME, birthdate
	FROM people;
	
	   num  name              birthdate   
	------  ----------------  ------------
	     1  Georges Danton    1759-10-26  
	     2  Herbert G Wells   1866-09-21  
	     3  Jimmy Hendrix     1942-11-27  
	     4  Angela Merkel     1954-07-17  
	     5  Rigoberta Menchu  1959-01-09  
	     6  Tracy Chapman     1964-03-30  
	```  
	
- 성별을 그룹으로 지정하고, 그룹마다 랭킹을 조회할 수도 있다.

	```sql
	SELECT ROW_NUMBER() OVER (PARTITION BY sex ORDER BY birthdate) AS num,
	sex, name, birthdate
	FROM people;
	
	   num  sex     name              birthdate   
	------  ------  ----------------  ------------
	     1  F       Angela Merkel     1954-07-17  
	     2  F       Rigoberta Menchu  1959-01-09  
	     3  F       Tracy Chapman     1964-03-30  
	     1  M       Georges Danton    1759-10-26  
	     2  M       Herbert G Wells   1866-09-21  
	     3  M       Jimmy Hendrix     1942-11-27  
	```  
	
### Common Table Expression
- 주로 서브쿼리로 쓰이는 파생 테이블과 비슷한 개념인 Common Table Expression 을 제공한다. 줄여서 CTE 라고 부른다.
- 기존 View 의 개념과 비슷하지만, 별도의 권한과 정의 없이 사용 가능하다.
- 복잡한 쿼리를 그룹핑해서 가독성과 재사용성을 향상 시킬 수있다.
- Recursive 특성으로 재귀 방식이나, 트리 구조로 활용 가능하다.

#### Recursive CTE 예시 1
- n 이 10 까지 1을 더하는 CTE 예시는 아래와 같다.

	```sql
	WITH RECURSIVE my_cte AS
	(
	  SELECT 1 AS n
	  UNION ALL
	  SELECT 1+n FROM my_cte WHERE n<10  # <- recursive SELECT
	)
	SELECT * FROM my_cte;
	
	     n  
	--------
	       1
	       2
	       3
	       4
	       5
	       6
	       7
	       8
	       9
	      10
	```  
	
- 피보나치의 현재 수와 다음 수를 구하는데, 현재 수가 500 보다 작을 때까지 구하는 CTE 예시는 아래와 같다.

	```sql
	WITH RECURSIVE my_cte AS
	(
	  SELECT 1 as f, 1 as next_f
	  UNION ALL
	  SELECT next_f, f+next_f FROM my_cte WHERE f < 500
	)
	SELECT * FROM my_cte;
	
	     f  next_f  
	------  --------
	     1         1
	     1         2
	     2         3
	     3         5
	     5         8
	     8        13
	    13        21
	    21        34
	    34        55
	    55        89
	    89       144
	   144       233
	   233       377
	   377       610
	   610       987
	```  
	
### NOWAIT and SKIP LOCKED

### Descending Indexes

	



































---
## Reference
[]()   