



 
### 인덱스가 사용되는 경우


### 조건에 따른 인덱스 사용 여부
`MySQL` 의 실행 계획인 `explain` 을 사용해서 조회 쿼리가 인덱스를 사용하는지 확인해 볼 수 있다. 
테스트를 위해 `num`, `str3` 에 인덱스를 생성해 준다. 

```bash
mysql> create index idx_num on exam(num);
Query OK, 0 rows affected (11.29 sec)
Records: 0  Duplicates: 0  Warnings: 0
```  

```bash
count(value) -> count(1) 로 바꿔서 테스트 다시 해보기 !!!!!

https://12bme.tistory.com/160
위링크에서 explain 먼저 정리하고 이거 다시 하는거 고민해 보기

explain select count(value) from exam where num = 1000; 
o

explain select count(value) from exam where num * 1 = 1000; 
x
explain select count(value) from exam where num = 1000 * 1; 
o

explain select count(value) from exam where num in (1000, 1001); 
o

explain select count(value) from exam where num <> 1000;
x
explain select count(value) from exam where num != 1000;
x

explain select count(value) from exam where num > 1000;
x
explain select count(value) from exam where num > 1000 and num < 2000;
x

explain select count(value) from exam where num between 1000 and 2000;
x

explain select max(num) from exam;
o

explain select count(value) from exam where group_no like 'group100'; 
o
explain select count(value) from exam where group_no like 'group100%';
o
explain select count(value) from exam where group_no like '%group100';
x

explain select max(group_no) from exam;
o

explain select count(value) from exam where group_no between 'group100' and 'group200';
x

explain select count(value) from exam where str3 is null;
o
explain select count(value) from exam where str3 is not null;
x

explain select count(value) from exam where datetime between '2020-09-19 09:55:00' and '2020-09-19 09:56:00';
x

explain select max(datetime) from exam;
o

```


### 인덱스가 사용되지 않는 경우



### 인덱스 동작에 따른 쿼리 성능 비교







---
## Reference
[[mysql] 인덱스 정리 및 팁](https://jojoldu.tistory.com/243)  
[[MySQL] 인덱스 종류 및 고려사항 (단일, 복합, 클러스터, 논클러스터, 커버드)](https://mozi.tistory.com/199)  
[MySQL Workbench의 VISUAL EXPLAIN으로 인덱스 동작 확인하기](https://engineering.linecorp.com/ko/blog/mysql-workbench-visual-explain-index/)  