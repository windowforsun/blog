--- 
layout: single
classes: wide
title: "[MongoDB 개념] MongoDB Aggregation"
header:
  overlay_image: /img/mongodb-bg.png
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - MongoDB
tags:
  - MongoDB
  - Concept
  - MongoDB
toc: true
use_math: true
---  

## MongoDB Aggregation
`Aggregation` 작업은 여러 `Document` 를 처리하고 계산된 결과를 반환하는데 사용할 수 있는데 그 예는 아래와 같다. 

- 여러 `Document` 의 값을 그룹화 한다. 
- 그룹화한 데이터를 기반으로 처리 작업을 수행해서 단일 결과를 반환한다. 
- 시간 경과에 따른 데이터 분석을 수행할 수 있다. 


## Single Purpose Aggregation Operations
`Collection` 에서 아래와 같은 바로 사용할 수 있는 `Aggregation` 동작을 제공한다. 
아래 작업들은 단일 컬렉션에서 문서를 집계하고, 
단순한 `Aggregation` 기능만 제공한다.  

- [db.collection.estimatedDocumentCount()](https://docs.mongodb.com/manual/reference/method/db.collection.estimatedDocumentCount/#mongodb-method-db.collection.estimatedDocumentCount)
- [db.collection.count()](https://docs.mongodb.com/manual/reference/method/db.collection.count/#mongodb-method-db.collection.count)
- [db.collection.distinct()](https://docs.mongodb.com/manual/reference/method/db.collection.distinct/#mongodb-method-db.collection.distinct)


![그림 1]({{site.baseurl}}/img/mongodb/practice-mongodb-aggregation-1.svg)  

## Aggregation Pipeline
`Aggregatuib` 연산을 구성하는 `Pipeline` 은 여러개의 [Stages](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/#std-label-aggregation-pipeline-operator-reference)
로 구성될 수 있다.  

- 각 `Stage` 는 `Document` 에 대한 필터링, 그룹화, 특정 계산을 수행한다. 
- 각 `Stage` 의 출력은 다음 `Stage` 입력이 된다. 
- `Aggregation Pipeline` 은 `Documnet` 의 그룹에 대한 `total`, `avg`, `max`, `min` 등의 결과값을 반환한다. 

간단한 `Aggregation Pipeline` 의 예시는 아래와 같다. 
먼저 테스트를 위해 아래와 같은 데이터를 추가해 준다.  

```bash
> db.user.insertMany([
  {_id:0, geder:"male", age:19, asset:10000, location:"B"},
  {_id:1, geder:"female", age:20, asset:11000, location:"A"},
  {_id:2, geder:"male", age:21, asset:12000, location:"A"},
  {_id:3, geder:"female", age:22, asset:13000, location:"A"},
  {_id:4, geder:"male", age:23, asset:14000, location:"A"},
  {_id:5, geder:"female", age:24, asset:15000, location:"A"},
  {_id:6, geder:"male", age:25, asset:16000, location:"A"},
  {_id:7, geder:"female", age:26, asset:17000, location:"A"},
  {_id:8, geder:"male", age:27, asset:18000, location:"A"},
  {_id:9, geder:"female", age:28, asset:19000, location:"A"},
  {_id:10, geder:"male", age:29, asset:20000, location:"A"},
  {_id:11, geder:"female", age:30, asset:21000, location:"A"}
]);
```  

먼저 `A` 지역에 있는 성별별 평균나이를 구하는 `Aggregation Pipleline` 은 아래와 같다.  

```bash
> db.user.aggregate([
  {$match: {location: "A"}},
  {$group: {_id: "$geder", avgAge: { $avg: "$age"}}}
]);
{ "_id" : "male", "avgAge" : 25 }
{ "_id" : "female", "avgAge" : 25 }
```  

`$match` `stage` 에서 `loation:"A"` 인 지역만 필터링하고, 
`$group` `stage` 에서 `gendar` 필드값을 기준으로 그룹화를 수행하고, 
각 그룹화된 구성에서 `age` 필드의 총 평균을 구한 결과이다.  

다음으로 `A` 지역에 있는 성별별 총 자산을 구하면 아래와 같다. 


```bash
> db.user.aggregate([
  {$match: {location: "A"}},
  {$group: {_id: "$geder", totalAsset: { $sum: "$asset"}}}
]);
{ "_id" : "male", "totalAsset" : 80000 }
{ "_id" : "female", "totalAsset" : 96000 }
```  

## Aggregation Pipeline Stage
`Aggregation Pipeline` 은 하나 이상의 `Stage` 로 구성되고 `Stage` 의 특징은 아래와 같다.  

- 각 `Stage` 는 `Document` 가 `Pipeline` 을 통화할때 `Document` 를 반환한다. 
- `Stage` 는 `Document` 에 대해서 필터링 혹은 그룹화를 수행하기 때문에 매 입력 마다 출력 `Document` 가 발생하지는 않는다. 
- `$out`, `$merge`, `$geoNear` 과 같은 `Stage` 는 하나의 `Pipeline` 에서 여러번 사용될 수 있다. 
- 사용할 수 있는 모든 `Stage` 에 대한 정보는 [Aggregation Pipeline Stage](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/#std-label-aggregation-pipeline-operator-reference)
에서 확인 할 수 있다. 
  
`Aggregation Pipeline Stage` 에 대한 쿼리 구성은 아래와 같다.  

```
db.collection.aggregate([ {<stage>}, ... ])
```  


### $addFields
[$addFields](https://docs.mongodb.com/manual/reference/operator/aggregation/addFields/#mongodb-pipeline-pipe.-addFields)
는 입력 `Document` 에 새로운 필드를 추가한다. 출력 `Document` 는 기존 모든 필드와 신규 필드를 포함한다. ($project 와 유사)

```bash
> db.test.insertMany([
  {_id:1, str:"a", num1:1, num2:11, list:[1, 2, 3], embeddedDoc: {str: "aa", num11: 1}},
  {_id:2, str:"b", num1:2, num2:22, list:[4, 5, 6], embeddedDoc: {str: "bb", num11: 2}}
]);
```  

```bash
.. 배열 합 필드 추가 및 필드의 합 필드 추가 ..
> db.test.aggregate([
  {
    $addFields: {
      totalList: {$sum: "$list"}
    }
  },
  {
    $addFields: {
      totalSum: {$add: [ "$totalList", "$num1", "$num2" ]}
    }
  }
]);
{ "_id" : 1, "str" : "a", "num1" : 1, "num2" : 11, "list" : [ 1, 2, 3 ], "totalList" : 6, "totalSum" : 18 }
{ "_id" : 2, "str" : "b", "num1" : 2, "num2" : 22, "list" : [ 4, 5, 6 ], "totalList" : 15, "totalSum" : 39 }
```  

```bash
.. Embedded Document 에 필드 추가 ..
> db.test.aggregate([
  {
    $addFields: {
      "embeddedDoc.newField" : "new!"
    }
  }
]);
{ "_id" : 1, "str" : "a", "num1" : 1, "num2" : 11, "list" : [ 1, 2, 3 ], "embeddedDoc" : { "str" : "aa", "num11" : 1, "newField" : "new!" } }
{ "_id" : 2, "str" : "b", "num1" : 2, "num2" : 22, "list" : [ 4, 5, 6 ], "embeddedDoc" : { "str" : "bb", "num11" : 2, "newField" : "new!" } }
```  

```bash
.. 기존 필드 값 교체 ..
> db.test.aggregate([
  {
    $addFields : {"num2" : 1000}
  }
]);
{ "_id" : 1, "str" : "a", "num1" : 1, "num2" : 1000, "list" : [ 1, 2, 3 ], "embeddedDoc" : { "str" : "aa", "num11" : 1 } }
{ "_id" : 2, "str" : "b", "num1" : 2, "num2" : 1000, "list" : [ 4, 5, 6 ], "embeddedDoc" : { "str" : "bb", "num11" : 2 } }
```  

```bash
.. 기존 필드를 다른 필드의 값으로 교체 ..
> db.test.aggregate([
  {
    $addFields: {
      _id: "$str",
      str: "newStr"
    }
  }
]);
{ "_id" : "a", "str" : "newStr", "num1" : 1, "num2" : 11, "list" : [ 1, 2, 3 ], "embeddedDoc" : { "str" : "aa", "num11" : 1 } }
{ "_id" : "b", "str" : "newStr", "num1" : 2, "num2" : 22, "list" : [ 4, 5, 6 ], "embeddedDoc" : { "str" : "bb", "num11" : 2 } }
```  

```bash
.. 기존 배열 필드에 값 추가 ..
> db.test.aggregate([
  {
    $addFields: {
      list : {$concatArrays: ["$list", [3000]]}
    }
  }
]);
{ "_id" : 1, "str" : "a", "num1" : 1, "num2" : 11, "list" : [ 1, 2, 3, 3000 ], "embeddedDoc" : { "str" : "aa", "num11" : 1 } }
{ "_id" : 2, "str" : "b", "num1" : 2, "num2" : 22, "list" : [ 4, 5, 6, 3000 ], "embeddedDoc" : { "str" : "bb", "num11" : 2 } }
```


### $bucket
[$bucket](https://docs.mongodb.com/manual/reference/operator/aggregation/bucket/#mongodb-pipeline-pipe.-bucket)
은 입력 `Document` 에 대해 지정된 식에 따라 `bucket` 이라는 그룹으로 분류한다. 
그리고 각 `Bucket` 별로 `Document` 를 출력한다.  

버킷 `Stage` 에서 `RAM` 은 최대 `100MB` 까지만 사용 할 수 있다. 
초과하게 되면 오류를 반환하고, 더 많은 공간이 필요한 경우 `allowDiskUse` 사용해서 가능하다.  


``` json
{
  $bucket: {
      groupBy: <expression>,
      boundaries: [ <lowerbound1>, <lowerbound2>, ... ],
      default: <literal>,
      output: {
         <output1>: { <$accumulator expression> },
         ...
         <outputN>: { <$accumulator expression> }
      }
   }
}
```  

```bash
> db.test.insertMany([
  {_id: 1, type2: "a", type3: 10, type4: 100, type5: "aa"},
  {_id: 2, type2: "b", type3: 20, type4: 200, type5: "bb"},
  {_id: 3, type2: "a", type3: 30, type4: 300, type5: "cc"},
  {_id: 4, type2: "b", type3: 10, type4: 400, type5: "dd"},
  {_id: 5, type2: "a", type3: 20, type4: 100, type5: "ee"},
  {_id: 6, type2: "b", type3: 30, type4: 200, type5: "aa"},
  {_id: 7, type2: "a", type3: 10, type4: 300, type5: "bb"},
  {_id: 8, type2: "b", type3: 20, type4: 400, type5: "cc"}
]);
```  

```bash
.. type3 을 기준으로 버킷을 그룹화 하고, 그룹화 된 문서의 수에 따라 필터링 ..
> db.test.aggregate([
  {
    $bucket: {
      groupBy: "$type3",
      boundaries: [10, 20, 30],
      default: "Other",
      output: {
        "groupCount" : {$sum: 1},
        "embeddedDoc" : {
          $push: {
            "concat1" : {$concat: ["$type2", "$type5"]},
            "type4" : "$type4"
          }
        }
      }
    }
  },
  {
    $match: {groupCount: {$gt: 2}}
  }
]);
{ "_id" : 10, "groupCount" : 3, "embeddedDoc" : [ { "concat1" : "aaa", "type4" : 100 }, { "concat1" : "bdd", "type4" : 400 }, { "concat1" : "abb", "type4" : 300 } ] }
{ "_id" : 20, "groupCount" : 3, "embeddedDoc" : [ { "concat1" : "bbb", "type4" : 200 }, { "concat1" : "aee", "type4" : 100 }, { "concat1" : "bcc", "type4" : 400 } ] }
```  

```bash
.. $facet 을 이용해서 한 Stage 에서 2개 그룹(type2, type3) 수행 ..
> db.test.aggregate([
  {
    $facet: {
      "type2" : [
        {
          $bucket: {
            groupBy: "$type2",
            boundaries: ["a", "b"],
            default: "Other",
            output: {
              "type2GroupCount" : {$sum: 1},
              "embeddedDoc" : {$push: {"type3" : "$type3", "type5", "$type5"}},
              "avgType4" : {$avg: "$type4"}
            }
          }
        }
      ],
      "type3" : [
        {
          $bucket: {
            groupBy: "$type3",
            boundaries: [10, 20, 30],
            default: "Unknown",
            output: {
              "type3GroupCount" : {$sum: 1},
              "embeddedDoc" : {$push: {"type2" : "$type2", "type5", "$type5"}},
              "avgType4" : {$avg: "$type4"}
            }
          }
        }
      ]
    }
  }
]);
```


```bash
.. $facet 을 이용해서 한 Stage 에서 2개 그룹(type2, type3) 수행 ..
> db.test.aggregate([
  {
    $facet: {
      "type4" : [
        {
          $bucket: {
            groupBy: "$type4",
            boundaries: [100, 300, 500],
            default: "Other",
            output: {
              "type2GroupCount" : {$sum: 1},
              "embeddedDoc" : {$push: {"type3" : "$type3", "type5", "$type5"}},
              "avgType3" : {$avg: "$type3"}
            }
          }
        }
      ],
      "type3" : [
        {
          $bucket: {
            groupBy: "$type3",
            boundaries: [10, 20, 30],
            default: "Unknown",
            output: {
              "type3GroupCount" : {$sum: 1},
              "embeddedDoc" : {$push: {"type2" : "$type2", "type5", "$type5"}},
              "avgType4" : {$avg: "$type4"}
            }
          }
        }
      ]
    }
  }
]);
```




```bash
.. $facet 을 이용해서 한 Stage 에서 2개 그룹(type2, type3) 수행 ..
> db.test.aggregate([
  {
    $facet: {
      "type2" : [
        {
          $bucket: {
            groupBy: "$type2",
            boundaries: ["a", "b"],
            default: "Other",
            output: {
              "type2GroupCount" : {$sum: 1},
              "embeddedDoc" : {$push: {"type3" : "$type3", "type5", "$type5"}},
              "avgType4" : {$avg: "$type4"}
            }
          }
        }
      ]
    }
  }
]);
```





Stage|Desc
---|---
[$addFields](https://docs.mongodb.com/manual/reference/operator/aggregation/addFields/#mongodb-pipeline-pipe.-addFields)|입력 `Document` 에 새로운 필드를 추가한다. 출력 `Document` 는 기존 모든 필드와 신규 필드를 포함한다. ($project 와 유사)
[$bucket](https://docs.mongodb.com/manual/reference/operator/aggregation/bucket/#mongodb-pipeline-pipe.-bucket)|입력 `Document` 에 대해 지정된 식에 따라 `bucket` 이라는 그룹으로 분류한다. 





---
## Reference
[MongoDB Indexes](https://docs.mongodb.com/manual/indexes/)  








