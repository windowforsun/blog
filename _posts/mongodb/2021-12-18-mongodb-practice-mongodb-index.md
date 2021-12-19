--- 
layout: single
classes: wide
title: "[MongoDB 개념] MongoDB Index"
header:
  overlay_image: /img/mongodb-bg.png
excerpt: 'MongoDB 의 Index 에 대한 개념과 사용할 수 있는 Index 종류와 특징에 대해 알아보자.'
author: "window_for_sun"
header-style: text
categories :
  - MongoDB
tags:
  - MongoDB
  - Concept
  - MongoDB
  - Index
  - Compound Index
  - Hidden Index
  - TTL Index
  - Sparse Index
  - Partial Index
  - Unique Index
  - Hashed Index
  - Text Index
  - Geospatial Index
  - Multikey Index
  - Single Field Index
toc: true
use_math: true
---  

## MongoDB Indexes
여타 다른 데이터베이스와 동일하게 `MongoDB` 의 `Index` 또한 `Read` 쿼리는 수행하는데 있어 효율성을 가져다주는 방법중 하나이다. 
만약 `Index` 를 사용하지 않는다면 `Read` 쿼리가 수행될때 `Collection Scan` 을 수행하게 된다. 
여기서 `Collection Scan` 이란 `Collection` 에 있는 전체 `Document` 를 모두 하나씩 읽으면서 쿼리와 매칭시키기 때문에 성능 효율성 입장에서는 최악의 방법이다. 
만약 `Read` 쿼리에 사용될 수 있는 적절한 `Index` 가 있는 경우라면 `MongoDB` 는 해당 `Index` 를 사용해서 모든 `Document` 를 사용하지 않기 때문에 성능적인 이점이 있다.  

이러한 장점이 있는 `MongoDB Index` 의 특징과 몇가지 종류에 대해 간단하게 알아본다.  

`Index` 는 하나의 `DataSet` 에서 작은 부분을 탐색하기 좋은 구조로 저장해놓은 특수한 데이터 구조이다. 
`Index` 로 지정된 필드는 하나의 필드 혹은 여러 필드의 집합으로 구성할 수 있다. 
이렇게 `Index` 로 지정된 필드를 사용해서 쿼리를 수행하면 `Equal`, `Range` 등의 작업에서 효율적인 탐색을 수행할 수 있고, 
정렬관련 조건에도 좋은 성능을 보여준다.  

아래는 쿼리의 조건에 맞춰 `Index` 가 사용되고 정렬하는 과정을 도식화 한 그림이다.  

![그림 1]({{site.baseurl}}/img/mongodb/practice-mongodb-index-1.svg)  

앞서 설명한 내용과 위 그림에서 알수 있듯이 기본적으로 `MongoDB` 의 `Index` 는 여타 데이터베이스들이 `Index` 와 크게 다르지 않다. 
`MongoDB` 는 `Collection` 레벨에서 `Index` 를 정의하고, `Doucment` 의 필드 혹은 `Imbedded Document` 의 필드를 `Index` 로 지정할 수 있다.


### 환경 구성
`MongoDB Index` 를 사용해볼 환경을 간단하게 `Docker Container` 로 구성해본다. 
아래 명령어를 사용해서 `MongoDB` 컨테이너를 실행하고, 컨테이너를 거쳐 `MongoDB` 에 접속한다. 

```bash
$ docker run -d --rm --name mongodb-test -p 27017:27017 mongo:5.0

$ docker exec -it mongodb-test /bin/bash
root@ab6ffa5823af:/# mongo
MongoDB shell version v5.0.5
>
```  

다음 과정은 `admin` 데이터베이스에서 인증정보를 설정하는 과정이다.  

```bash
> use admin
switched to db admin
> db.createUser({user:'root',pwd:'root',roles:['root']})
Successfully added user: { "user" : "root", "roles" : [ "root" ] }
```  

생성한 인증정보를 사용해서 로그인 하는 방법은 아래 2가지가 있다.  

```bash
$ mongo admin -u root -p 'root' --authenticationDatabase admin

or

$ mongo
> use admin
switched to db admin
> db.auth('root','root')
1
```  

마지막으로 `Index` 테스트에 사용할 `test-index` 라는 `Collection` 을 생성해 준다.  

```bash
> use test-index
switched to db test-index

```  


## Default `_id` Index
`MongoDb` 는 `Collection` 에 `Document` 를 추가할때 기본으로 [_id](https://docs.mongodb.com/manual/core/document/#std-label-document-id-field) 라는 [Unique Index](https://docs.mongodb.com/manual/core/index-unique/#std-label-index-type-unique) 를 생성한다. 
`_id` 를 통해 `Client` 가 중복되는 `Document` 가 `Collection` 에 추가되는 것을 방지하고, 필요에 따라 `_id` 필드는 삭제할 수 있다.  
`_id` 필드는 `Primary Key` 와 같은 역할로 `MongoDB` 에서 자동으로 생성하는 `Unique` 한 값으로 설정된다. 

아래 명령을 사용해서 `user` 컬렉션에 10개의 데이터를 추가해 하고 조회하면 아래와 같이, `_id` 필드가 자동 생성된 것을 확인 할 수 있다.  

```bash
> for (i = 0; i< 10; i++) {
... db.user.insert({
... "user_id":i,
... "name":"name-"+i,
... "age":(i%80),
... "datetime":new Date()
... })
... }
WriteResult({ "nInserted" : 1 })
> db.user.find();
{ "_id" : ObjectId("61bda35785b01a24b6a67331"), "user_id" : 0, "name" : "name-0", "age" : 0, "datetime" : ISODate("2021-12-18T09:01:11.734Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67332"), "user_id" : 1, "name" : "name-1", "age" : 1, "datetime" : ISODate("2021-12-18T09:01:11.758Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67333"), "user_id" : 2, "name" : "name-2", "age" : 2, "datetime" : ISODate("2021-12-18T09:01:11.759Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67334"), "user_id" : 3, "name" : "name-3", "age" : 3, "datetime" : ISODate("2021-12-18T09:01:11.759Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67335"), "user_id" : 4, "name" : "name-4", "age" : 4, "datetime" : ISODate("2021-12-18T09:01:11.760Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67336"), "user_id" : 5, "name" : "name-5", "age" : 5, "datetime" : ISODate("2021-12-18T09:01:11.760Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67337"), "user_id" : 6, "name" : "name-6", "age" : 6, "datetime" : ISODate("2021-12-18T09:01:11.761Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67338"), "user_id" : 7, "name" : "name-7", "age" : 7, "datetime" : ISODate("2021-12-18T09:01:11.761Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a67339"), "user_id" : 8, "name" : "name-8", "age" : 8, "datetime" : ISODate("2021-12-18T09:01:11.761Z") }
{ "_id" : ObjectId("61bda35785b01a24b6a6733a"), "user_id" : 9, "name" : "name-9", "age" : 9, "datetime" : ISODate("2021-12-18T09:01:11.762Z") }

```  

그리고 `user` 컬렉션에 생성된 `Index` 를 조회하면 아래와같이 `_id_` 라는 이름의 `Index` 만 존재하는 것을 확인 할 수 있다.  

```bash
> db.user.getIndexes();
[ { "v" : 2, "key" : { "_id" : 1 }, "name" : "_id_" } ]
```  

## Create Index
`Collection` 에 `Index` 를 생성은 `db.<collection>.createIndex()` 를 통해 할 수 있다. 
위 에서 미리 만들어 놓은 `user` 컬렉션의 `name` 필드에 `Index` 를 생성하는 예시는 아래와 같다.  

```bash
> db.user.createIndex({name:1})
{
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
> db.user.getIndexes();
[
        {
                "v" : 2,
                "key" : {
                        "_id" : 1
                },
                "name" : "_id_"
        },
        {
                "v" : 2,
                "key" : {
                        "name" : 1
                },
                "name" : "name_1"
        }
]
```  

`createIndex({name:1})` 은 `name` 필드를 오름차순(`ASC`)을 사용한다는 의미이다. 
반대로 `createIndex({name:-1})` 는 내림차순(`DESC`)을 의미한다.  

### Index Names
`createIndex` 명령을 사용해서 생성된 인덱스의 이름은 기본적으로 아래 규칙을 따른다.  

```
<field name>_<order direction>_<field name2>_<order direction2>
```  

즉 위에서 생성한 `name` 필드 인덱스 이름이 `name_1` 인것은 `name` 필드에서 정렬 방향이 `1`(오름차순) 인 인덱스를 의미한다. 
만약 `name` 필드에 정렬 방향이 `-1`(내림차순) 인 인덱스를 만들게 되면 이름은 `name_-1` 이 될 것이다. 
그리고 `Index` 이름에 커스텀한 설정을 원하는 경우 `createIndex({name:1},{name:<index-name>})` 와 같이 사용해주면 된다.  
예를 위해 몇가지 인덱스를 `user` 컬렉션에 추가한다.  

```bash
> db.user.createIndex({name:-1, age:1});
{
        "numIndexesBefore" : 2,
        "numIndexesAfter" : 3,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
> db.user.createIndex({name:1, datetime:-1}, {name: "name-asc-datetime-desc"})
{
        "numIndexesBefore" : 3,
        "numIndexesAfter" : 4,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
> db.user.getIndexes();
[
        .. 생략 ..
        
        {
                "v" : 2,
                "key" : {
                        "name" : -1,
                        "age" : 1
                },
                "name" : "name_-1_age_1"
        },
        {
                "v" : 2,
                "key" : {
                        "name" : 1,
                        "datetime" : -1
                },
                "name" : "name-asc-datetime-desc"
        }
]
```  


## Index Types
`MongoDB` 는 다양한 유형의 데이터 및 쿼리를 지원하기 위해 몇가지 종류의 인덱스를 제공한다.  

### Single Field
[Single Field Indexes](https://docs.mongodb.com/manual/core/index-single/)
는 사용자가 원하는 하나의 `Document Field` 에 `Asc/Desc` 중 하나의 타입을 선택해서 생성하는 인덱스이다. 
대표적으로 앞서 설명한 기본으로 생성되는 `_id Index` 가 `Single Field` 인덱스에 해당한다. 

```bash
> db.user.createIndex({score:1})

> db.user.find({score: 1000})
> db.user.find({score: {$gt: 500}})
```  

![그림 1]({{site.baseurl}}/img/mongodb/practice-mongodb-index-2.svg)  

`Single Field` 인덱스에서도 `Asc/Desc` 중 하나의 타입으로 선택할 수 있다고 말했지만, 
실제로는 어느 것을 선택하든 동작에 큰 차이는 없다. 
단일 인덱스 특성상 정령이 어떤 방식으로 되었던 간에 데이터 탐색에 있어서는 동일하기 때문이다.  

#### Embedded Field Index
`Embedded Document` 를 사용하는 경우 전체 `Embedded Document` 를 포함하지 않고 특정 `Embedded Field` 만 인덱스에 포함하는 할 수 있다. 
`Embedded Field` 를 인덱스로 추가할때는 `dot notation(.)` 을 사용한다. 
아래와 같은 `user` 컬렉션의 `location` 필드에 `Embedded Doucment` 가 있다. 

```json
{
	"_id": ObjectId("570c04a4ad233577f97dc459"),
	"userid": 1034,
	"name": "aa",
	"score": 1000,
	"age": 10,
	"datetime" : ISODate("2021-12-18T09:01:11.762Z"),
	"location": { state: "NY", city: "New York" }
}
```  

아래와 같이 `Embedded Field` 인덱스를 생성할 수 있다.  

```bash
> db.user.createIndex({"location.state" : 1})
> db.user.getIndexes()
[
        .. 생략 ..
        
        {
                "v" : 2,
                "key" : {
                        "location.state" : 1
                },
                "name" : "location.state_1"
        }
]

> db.user.find({"location.state": "NY"})
```  

#### Embedded Document Index
다음으로는 `Embedded Document` 전체를 인덱스 필드로 지정하는 방법이 있다.  

```bash
> db.user.createIndex({location: 1})
{
        "numIndexesBefore" : 5,
        "numIndexesAfter" : 6,
        "createdCollectionAutomatically" : false,
        "ok" : 1
}
> db.user.getIndexes();
[
        .. 생략 ..
        
        {
                "v" : 2,
                "key" : {
                        "location" : 1
                },
                "name" : "location_1"
        }
]

> db.user.find({location: {state: "NY", state: "New York"}})
```  



### Compound Index
[Compound Index](https://docs.mongodb.com/manual/core/index-compound/)
는 번역하면 `복합 인덱스` 로 사용자가 원하는 여러개 `Document Field` 를 사용해서 구성하는 인덱스를 의미한다. 

```bash
> db.user.createIndex({userid:1,score:-1})

> db.user.find({userid:"aa1", score: 1000})
> db.user.find({userid:"aa1", score: {$gt: 1000}})
```  

`Compound Index` 는 여러개의 키로 인덱스를 생성하기 때문에 정렬 순서를 어느 것으로 구성하느냐에 따라 유의미한 차이를 보인다. 
`createIndex({name:1,age:-1})` 라는 인덱스를 생성했다면 먼저 `name` 필드 기준 오름차순으로 정렬되고, 
동일한 `name` 이 존재하는 경우 `age` 필드 기준 내림차순으로 정렬된다.  


![그림 1]({{site.baseurl}}/img/mongodb/practice-mongodb-index-3.svg)  


#### Sort Order
`Compound Index` 를 생성할때 필드의 순서와 정렬 방향은 이후 `sort()` 연산에 많은 영향을 미친다.
`db.user.createIndex({userid:1,score:1})` 와 같은 인덱스를 생성했다면 `sort()` 연산의 종류마다 특징은 아래와 같다. 
- `db.user.find().sort({userid:1, score:1})` : 가장 좋음
- `db.user.find().sort({score:1, userid:1})` : 성능에 좋지 않음
- `db.user.find().sort({userid:-1, score:1})` : 성능에 좋음
- `db.user.find().sort({userid:1, score:-1})` : 성능에 좋지 않음
- `db.user.find().sort({userid:-1, score:-1})` : 성능에 좋지 않음

#### Prefixes
`Index Prefixes` 는 인덱스를 구성하는 필드의 순서대로 시작부터 끝까지의 부분집합을 의미한다. 
아래와 같은 `Compound Index` 가 있다고 가정해 본다.  

```
db.user.createIndex({userid:1,score:1,age:1})
```  

여기서 가능한 `Index Prefixes` 를 나열하면 아래와 같다.  

- `{userid:1}`
- `{userid:1,score:1}`
- `{userid:1,score:1,age:1}`

즉 다시 설명하면 `{userid:1,score:1,age:1}` 인덱스를 생성하면 위 3가지 인덱스를 생성한 것과 동일하게 쿼리에서 인덱스가 사용될수 있다는 것이다. 
몇가지 예제로 어떤 쿼리에서 어떤 `Index Prefixes` 가 사용되는지 살펴 본다.  

쿼리|사용된 `Index Prefixes`
---|---
`db.user.find().sort({userid:1})`|`{userid:1}`
`db.user.find().sort({userid:-1})`|`{userid:1}`
`db.user.find().sort({userid:-1, score:1})`|`{userid:1, score:1}`
`db.user.find().sort({userid:-1, score:-1})`|`{userid:1, score:1}`
`db.user.find().sort({userid:1, score:1,age:1})`|`{userid:1, score:1, age:1}`
`db.user.find(userid:{$gt:"a"}).sort({userid:1, score:1})`|`{userid:1, score:1}`

만약 `sort()` 연산의 조건이 `Index Prefixes` 를 만족하지 못하는 경우라면, 
`find()` 연산 조건이 `Equality` 상태이면 `Index Prefixes` 조건이 만족된다. 
해당하는 몇가지 케이스를 나열하면 아래와 같다.  

쿼리| 사용된 `Index Prefixes`
---|---
`db.user.find({userid:"aa"}).sort({score:1,age:1})`|`{userid:1,score:1,age:1}`
`db.user.find({age:20,userid:"aa"}).sort({age:1})`|`{userid:1,score:1,age:1}`
`db.user.find({userid:"aa", score:{$gt: 1000}}).sort({score:1})`|`{userid:1,score:1}`

다음은 `find()` 조건이 `Equality` 가 아니면서 인덱스의 선행키가 아닌 경우므로 인덱스가 효과적으로 사용되지 못하는 쿼리이다.  

|쿼리|
|---|
`db.user.find({userid:{$gt:"a"}}).sort({age:1})`
`db.user.find({age:20).sort({age:1})`

#### Index Intersection
`Index Intersection`(인덱스 교차)는 `MongoDB 2.6` 부터 지원되는 기능이다. 
이는 인덱스의 종류보다는 인덱스의 작동방식에 대한 부분으로 `Collection` 에서 각 1개 이상의 `Single Field Index`, `Compound Index` 가 정의 됐다고 했을 때, 
하나의 쿼리에서 적절한 여러개의 쿼리를 내부에서 교집합처럼 동작해서 성능을 높이는 방식을 의미한다. 

아래와 같은 2개 인덱스를 생성했다고 가정해 보자.

```bash
> db.user.createIndex({userid:1})
> db.user.createIndex({score:1})
```  

그리고 사용할 쿼리는 아래와 같다.  

```bash
> db.user.find({score:1000, userid:{$gt:"a"}})
```  

위 쿼리를 사용하면 `Index Intersection` 이 적용되어 인덱스를 활용해서 쿼리가 수행된다. 
하지만 인덱스 교차의 경우 명시적인 부분이 아니기 때문에 `explain` 을 통해 확인을 해줘야 한다. 
인덱스 교차가 사용되면 `explain` 에서 `AND_SORTED` 혹은 `AND_HASH` 스테이지를 확인할 수 있다. 

인덱스 교차는 `Compound Index` 에 비해 고려해야하는(필드 순서, 필드 정렬 조건, `Index Prefixes`) 사항들이 비교적 적다는 장점이 있다. 
하지만 쿼리의 `find()` 연산 조건에 사용된 인덱스와 별개로 선언된 인덱스를 정렬 조건으로 사용할수 없고, 
전반적으로 `Compound Index` 와 비교해서 성능이 느리다는 점들이 있다.  

다음 케이스로 아래와 같은 인덱스가 생성된 상태를 가정해 본다.  

```bash
> db.user.createIndex({userid:1})
> db.user.createIndex({score:1, age:-1})
> db.user.createIndex({score:1})
> db.user.createIndex({age:-1})
```  

위 인덱스인 상태에서 아래 쿼리는 인덱스 교차가 불가능하다.  

```bash
> db.user.find({userid:{$gt:"a"}}).sort({score:1})
```  

적용되지 않는 이유는 `userid` 의 인덱스와 `score` 인덱스가 별도의 인덱스이기 때문이다.  

그리고 아래 쿼리는 정상적으로 인덱스 교차가 적용된다.  

```bash
> db.user.find({userid:{$gt:"a"},score:1000}).sort({age:-1})
```  

다른 경우로 아래와 같은 인덱스가 있다고 다시 가정해 본다.  

```bash
> db.user.createIndex({score:1, age:-1})
```  

인덱스 적용이 가능한 쿼리와 불가능한 쿼리는 아래와 같다.  

```bash
.. 인덱스가 적용되는 쿼리 ..
> db.user.find({score:{$in:[1000, 1500]}})
> db.user.find({age:{$gt:20},score:{$in:[1000, 1500]}})

.. 인덱스가 적용 되지 않는 쿼리(선행 인덱스가 정의되지 않았기 때문) ..
> db.user.find({age:{$gt:20}})
> db.user.find({}).sort({age:1})
```  

아래와 같이 인덱스를 다시 생성해주면 교차 인덱스가 적용되면서 뭐든 쿼리에 인덱스 적용이 가능하다.  

```bash
> db.user.createIndex({score:1})
> db.user.createIndex({age:-1})
```  

### Multikey Index
[Multikey Index](https://docs.mongodb.com/manual/core/index-multikey/) 
를 사용하면 `Embedded Document` 필드를 사용해서 인덱스를 생성할 수 있다. 
`Single Field Index` 에서 언급했었던 `Embedded Field Index` 가 바로 `Multikey Index` 를 의미한다. 

아래 와 같은 `Document` 가 있다고 가정해 본다.  

```json
{
	"_id": ObjectId("570c04a4ad233577f97dc459"),
	"userid": 1034,
	"name": "aa",
	"score": 1000,
	"age": 10,
	"datetime" : ISODate("2021-12-18T09:01:11.762Z"),
	"location": { 
		state: "NY", 
		city: "New York" 
	},
	"addr" : [
		{"zip": "10036"},
		{"zip": "94301"}
	],
	"hobby" : [
		"movie", "soccer", "music"
	]
}
```  

아래와 같이 `Multipkey Index` 를 생성할 수 있다. 

```bash
> db.user.createIndex({"addr.zip":1})
> db.user.createIndex({"location.state":1})
> db.user.createIndex({"hobby":1})

> db.user.find({"addr.zip":10036})
> db.user.find({"location.state":"NY"})
> db.user.find({"hobby":"movie"})
```  

![그림 1]({{site.baseurl}}/img/mongodb/practice-mongodb-index-4.svg)

`Multikey Index` 를 사용할때는 인덱스를 사용한 필드명과 동일하게 사용해야 적용이 가능하다. 

```bash
.. 불가능 ..

> db.user.find({addr:{zip: 10036}})
> db.user.find({location:{state: "NY"}})
```  

그리고 `Multikey Index` 는 아래와 같은 제약 사항들이 있다. 
- `Shard Key` 로 설정 불가능
- `Hashed Index` 불가능
- 조건부 `Covered Queries` 적용


### Geospatial Index
지리공산 좌표 데이터(위도, 경도)의 효율적인 쿼리를 위해 `MongoDB` 는 아래와 같은 2가지 특수 인덱스를 제공한다. 

- [2d Indexes](https://docs.mongodb.com/manual/core/2d/), [2d Index Internals](https://docs.mongodb.com/manual/core/geospatial-indexes/) : 평면 형상을 사용하는 좌표 인덱스
- [2dsphere Index](https://docs.mongodb.com/manual/core/2dsphere/) : 구형 형상을 사용하는 좌표 인덱스

아래와 같은 `Document` 가 있다고 해보자. 

```json
{
	"_id": ObjectId("570c04a4ad233577f97dc459"),
	"userid": 1034,
	"name": "aa",
	"score": 1000,
	"age": 10,
	"datetime" : ISODate("2021-12-18T09:01:11.762Z"),
	"location" : {
		"2d_loc" : [ 55.5, 42.3],
		"2dsphere_loc" : {
			"type" : "Point",
			"coordinates" : [55.5, 42.3]
		}
	}
}
```  

```bash
.. 2d index ..
> db.user.createIndex({"location.2d_loc":"2d"})
> db.user.find({"location.2d_loc":{$near:[11, 22]}})

.. 2d sphere index ..
> db.user.createIndex({"location.2dsphere_loc":"2dsphere"})
> db.user.createIndex({"location.2dsphere_loc":{$near:{"type":"Point", coordinates:[11, 22]}}})
```  

`Geospatial Index` 는 `Covered Quries` 가 될수 없고, `Shard Key` 로도 사용할 수 없지만 샤딩된 컬렉션에 `Geospatial Index` 를 생성해 사용할 수는 있다.  


### Text Index
[Text Index](https://docs.mongodb.com/manual/core/index-text/)
는 `MongoDB` 컬렉션에서 문자열 내용 검색을 지원하는 유형의 인덱스이다. 
문자열, 문자열 배열 필드에 적용 할 수 있고 컬렉션은 하나의 `Text Index` 만 가질 수 있다.  

```json
{
	"_id": ObjectId("570c04a4ad233577f97dc459"),
	"userid": 1034,
	"name": "aa",
	"score": 1000,
	"age": 10,
	"datetime" : ISODate("2021-12-18T09:01:11.762Z"),
	"introduction" : "Hi. Im aa",
	"keyword" : ["Seoul", "Coffee", "Movie"]
}
```  

```bash
> db.user.createIndex({introduction: "text", keyword:"text"})

> db.user.find({$text:{$search:"Hi"}})
> db.user.find({$text:{$search:"Coffee"}})
```  

### Hash Index
[Hash Index](https://docs.mongodb.com/manual/core/index-hashed/)
은 `Hash based Sharding` 을 지원하기 위해 필드 값의 해시값으 사용하는 인덱스이다. 
`Hash Index` 는 데이터 범위에 다라 값이 랜덤하게 분포된다는 장점이 있지만 `Equality` 연산만 제공하고 `Range` 연산은 제공하지 않는다. 
그리고 `Multikey Index` 를 지원하지 않기 때문에 필드 값이 배열이라면 에러가 발생한다. 

```json
{
	"_id": ObjectId("570c04a4ad233577f97dc459"),
	"userid": 1034,
	"name": "aa",
	"score": 1000,
	"age": 10,
	"datetime" : ISODate("2021-12-18T09:01:11.762Z"),
	"securitynumber" : "123456-1234567"
}
```  

```bash
> db.user.createIndex({securitynumber:"hashed"})

> db.user.find({securitynumber:"123456-1234567"})
```  





## Index Properties
### Unique Indexes
[Unique Indexes](https://docs.mongodb.com/manual/core/index-unique/)
는 `Collection` 에서 유니크한 값이 보장되는 필드 혹은 필드 조합 인덱스에 설정할 수 있는 값이다. 
만약 `Unique Index` 로 설정된 필드의 값이 비었다면 단 하나의 도큐먼트는 추가될수 있지만, 
이후에 한번더 동일한 필드값이 빈 도큐먼트를 추가하면 에러가 발생한다. 


```bash
> db.user.createIndex({userid: 1}, {unique: true})

> db.user.createIndex({name: 1, group: 1, category:1}, {unique: true})
```  


### Partial Indexes
[Partial Indexes](https://docs.mongodb.com/manual/core/index-partial/)(부분 인덱스)
는 지정된 필터 조건에 해당하는 `Collection` 의 `Document` 만 인덱싱한다. 
`Collection` 에 존재하는 전체 `Document` 를 인덱싱하지 않고 특정 조건에 맞는 것만 인덱싱 하기 때문에, 
저장소 요구사항을 좀더 낮출 수 있고 인덱스 생성 및 유지 관리에 대한 비용을 감소 할 수 있다. 

아래와 같은 `user` 도큐먼트가 있다고 가정해 본다. 

```json
{
	"_id": ObjectId("570c04a4ad233577f97dc459"),
	"userid": 1034,
	"name": "aa",
	"score": 1000,
	"age": 10,
	"datetime" : ISODate("2021-12-18T09:01:11.762Z")
}
```  

`Partial Index` 를 생성하면 아래와 같다.  

```bash
.. age 가 20 보다 큰 경우에만 partial index 생성 ..
> db.user.createIndex({name:1, score:1}, {partialFilterExpression:{age:{$gt:20}}})

.. partial index 사용 ..
> db.user.find({name:"aa", age:{$gt:20}})
> db.user.find({name:"aa", age:{$gt:60}})

.. partial index 사용 불가능 ..
> db.user.find({name:"aa", age:{$lt:80}})
> db.user.find({name:"aa"})
```  

위와 같이 `Partial Index` 는 `MongoDB` 에서 기본적으로는 사용되지 않는 인덱스이고, 
`Partial Index` 를 생성할 때 명시한 조건과 동일하거나 모두 포함되는 조건을 명시해 줘야 사용될 수 있다. 


### Sparse Indexes
[Sparse Indexes](https://docs.mongodb.com/manual/core/index-sparse/)
는 인덱스로 지정된 필드가 `null` 값을 같거나 존재하지 않더라도, 값이 있는 `Document` 만 인덱스를 구성할 수 있도록 한다. 
이는 `Unique Index` 의 필수 조건중 하나인 인덱스 필드가 존지하지 않을 때 `insert` 가 실패하는 현상을 회피할 수 있는 방법이다. 


```
> db.test_collection.insertOne( { _id: 1, y: 1 } );

.. x 필드를 인덱스 필드로 하는 sparse 인덱스 생성 ..
> db.test_collection.createIndex({x:1}, {sparse:true});

.. 전체 조회시 x필드가 없더라도 조회됨 ..
> db.test_collection.find().count();
> 1

.. 전체 조회시 hint() 로 {x:1} 인덱스를 사용하도록 강제하면 조회 되지 않음 ..
> db.test_collection.find().hint({x:1}).count();
> 0
```  


### TTL Indexes
[TTL Indexes](https://docs.mongodb.com/manual/core/index-ttl/)
는 `Collection` 에 `Document` 가 추가되고 나서 인덱스에 설정한 일정 시간 후에 자동으로 삭제하도록 설정할 수 있는 `Single Field` 인덱스이다. 
이는 제한된 시간동안만 유효한 이벤트 데이터, 로그, 세션 등에 사용될 수 있다. 
`TTL Index` 는 인덱스를 생성할 때 `expireAfterSeconds` 필드에 초값을 기입해서 생성할 수 있고, 
아래 쿼리는 `eventlog` 컬렉션에 있는 `lastModifiedDate` 필드를 기준으로 `expireAfterSeconds` 값과 비교해서 판별한다.  

만약 `TTL Index` 로 지정한 필드 값이 날짜 값의 배열이라면, 배열에 있는 가장 이른 날짜 값을 사용해서 만료 임계값을 계산한다. 
임계값은 (필드의 날짜값) + (인덱스에 설정한 초) 로 계산된다.  

그리고 인덱스 필드가 없거나, 날짜 값이 아니거나 배열에 날짜값이 존재하지 않는 경우 해당 `Document` 는 만료되지 않는다. 
더 자세한 내용은 [Expire Data from Collections by Setting TTL](https://docs.mongodb.com/manual/tutorial/expire-data/) 에서 확인 할 수 있다.  

아래와 같이 `TTL Index` 를 생성할 수 있다. 

```bash
.. insert 수행후 3600 초후 삭제된다 ..
> db.eventlog.createIndex( { "lastModifiedDate": 1 }, { expireAfterSeconds: 3600 } )
> db.eventlog.insertOne({
  "logType":"debug",
  "message":"test log",
  "lastModifiedDate": new Date()
})


.. insert 수행후 expiredDate 시점에 삭제 된다 ..
> db.eventlog.createIndex( { "expiredDate": 1 }, { expireAfterSeconds: 0 } )
> db.eventlog.insertOne({
  "logType":"debug",
  "message":"test log",
  "expiredDate": new Date('12 22, 2021 14:00:00')
})
```  

### Hidden Indexes
[Hidden Indexes](https://docs.mongodb.com/manual/core/index-hidden/)
는 `Query Planner` 에서 표시되지 않도록 하고, 쿼리에서도 설정된 인덱스는 사용되지 않도록 한다. 
이러한 인덱스가 있는 이유는 이미 존재하는 인덱스를 `Hidden Index` 로 만듬으로써 실제 인덱스를 삭제하지 않으면서, 
인덱스를 삭제 했을 때와 동일하게 발생될 수 있는 영향에 대해서 검토할 수 있다. 
해당 인덱스가 다시 필요한 경우 다시 `Hidden Index` 취소 시킬 수 있다.  

```bash
.. create hidden index ..
> db.user.createIndex({name:1}, {hidden:true})
> db.user.getIndexes()
[
        {
                "v" : 2,
                "key" : {
                        "name" : 1
                },
                "name" : "name_1",
                "hidden" : true
        }
]

.. unhide index ..
> db.user.unhideIndex({name:1});
{ "hidden_old" : true, "hidden_new" : false, "ok" : 1 }
> db.user.getIndexes();
[
        {
                "v" : 2,
                "key" : {
                        "name" : 1
                },
                "name" : "name_1"
        }
]

.. make hidden existing index
> db.user.hideIndex({name:1});
{ "hidden_old" : false, "hidden_new" : true, "ok" : 1 }
> db.user.getIndexes();
[
        {
                "v" : 2,
                "key" : {
                        "name" : 1
                },
                "name" : "name_1",
                "hidden" : true
        }
]
```  


## Index Use
지금까지 설명한 인덱스를 사용해서 데이터 타입과 구성에 맞는 읽기 작업의 효율성을 향상시킬 수 있다. 
관련해서 자세한 내용은 [Analyze Query Performance](https://docs.mongodb.com/manual/tutorial/analyze-query-plan/)
에서 다양한 정보를 확인할 수 있다. 
그리고 `MongoDB` 가 실제로 쿼리를 수행할때 어떤 인덱스를 선택해서 사용하는지는 [Query Plans](https://docs.mongodb.com/manual/core/query-plans/#std-label-read-operations-query-optimization)
를 통해 확인할 수 있다.  


### Covered Queries
`MongoDB` 의 쿼리는 아래의 그림과 같이 `<collection><Query Criteria><Projection>` 으로 이뤄져 있다. 
[Covered Queries](https://docs.mongodb.com/manual/core/query-optimization/#std-label-read-operations-covered-query)
이때 `Query Criteria`, `Projection` 이 모두 인덱스의 필드만 포함하는 경우 `MongoDB` 는 스토리지에 저장된 `Document` 를 읽거나, 
`Document` 를 메모리로 로드하는 작업을 수행하지 않고 인덱스만 사용해서 결과를 반환한다. 
위 조건이 인덱스를 사용할때 `Read` 동작의 성능을 가장 잘 끌어올릴수 있는 상황이라고 할 수 있다.  

![그림 1]({{site.baseurl}}/img/mongodb/practice-mongodb-index-5.svg)


### Index Intersection
`Single Field Index` 에서 먼저 설명했던 것처럼, 
[Index Intersection](https://docs.mongodb.com/manual/core/index-intersection/)
은 인덱스의 교집합점을 사용해서 쿼리를 수행하는 것을 말한다. 
`Compound Index` 로 지정된 몇개의 인덱스 필드 셋이 있을 때 한 인덱스가 쿼리 조건 일부를 충족하고, 
다른 인덱스가 쿼리 조건의 다른 부분을 충족하는 상황에서, 
두 인덱스의 교집합점을 사용하여 쿼리를 수행할 수 있다. 
실제로 `Compound Index` 를 사용하는지 `Index Intersection` 이 사용되는지는 쿼리의 조건, 인덱스 구성에 따라 달라질 수 있다.  


### Restrictions
인덱스를 생성할때 주의할 점들이 있는데 인덱스 키의 길이, 컬렉션당 인덱스 수등에 대한 제사한 내용은 [Index Limitations](https://docs.mongodb.com/manual/reference/limits/#std-label-index-limitations)
에서 확인 할 수 있다.  


---
## Reference
[MongoDB Indexes](https://docs.mongodb.com/manual/indexes/)  








