--- 
layout: single
classes: wide
title: "[MongoDB 개념] MongoDB Aggregation"
header:
overlay_image: /img/mongodb-bg.png
excerpt: 'MongoDB 에서 여러 Document 기반으로 데이터를 처리할 수 있는 Aggregation 의 개념과 사용법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - MongoDB
tags:
  - MongoDB
  - Concept
  - MongoDB
  - Aggregation
  - GroupBy
toc: true
use_math: true
---  

## MongoDB Aggregation

`Aggregation` 작업은 여러 `Document` 를 처리하고 계산된 결과를 반환하는데 사용할 수 있는데 그 예는 아래와 같다.

- 여러 `Document` 의 값을 그룹화 한다.
- 그룹화한 데이터를 기반으로 처리 작업을 수행해서 단일 결과를 반환한다.
- 시간 경과에 따른 데이터 분석을 수행할 수 있다.

## Single Purpose Aggregation Operations

`Collection` 에서 아래와 같은 바로 사용할 수 있는 `Aggregation` 동작을 제공한다. 아래 작업들은 단일 컬렉션에서 문서를 집계하고, 단순한 `Aggregation` 기능만 제공한다.

- [db.collection.estimatedDocumentCount()](https://docs.mongodb.com/manual/reference/method/db.collection.estimatedDocumentCount/#mongodb-method-db.collection.estimatedDocumentCount)
- [db.collection.count()](https://docs.mongodb.com/manual/reference/method/db.collection.count/#mongodb-method-db.collection.count)
- [db.collection.distinct()](https://docs.mongodb.com/manual/reference/method/db.collection.distinct/#mongodb-method-db.collection.distinct)

![그림 1]({{site.baseurl}}/img/mongodb/practice-mongodb-aggregation-1.svg)

## Aggregation Pipeline

`Aggregatuib` 연산을 구성하는 `Pipeline` 은
여러개의 [Stages](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/#std-label-aggregation-pipeline-operator-reference)
로 구성될 수 있다.

- 각 `Stage` 는 `Document` 에 대한 필터링, 그룹화, 특정 계산을 수행한다.
- 각 `Stage` 의 출력은 다음 `Stage` 입력이 된다.
- `Aggregation Pipeline` 은 `Documnet` 의 그룹에 대한 `total`, `avg`, `max`, `min` 등의 결과값을 반환한다.

간단한 `Aggregation Pipeline` 의 예시는 아래와 같다. 먼저 테스트를 위해 아래와 같은 데이터를 추가해 준다.

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
`$group` `stage` 에서 `gendar` 필드값을 기준으로 그룹화를 수행하고, 각 그룹화된 구성에서 `age` 필드의 총 평균을 구한 결과이다.

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
- 사용할 수 있는 모든 `Stage` 에 대한
  정보는 [Aggregation Pipeline Stage](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/#std-label-aggregation-pipeline-operator-reference)
  에서 확인 할 수 있다.

`Aggregation Pipeline Stage` 에 대한 쿼리 구성은 아래와 같다.

```
db.collection.aggregate([ {<stage>}, ... ])
```  

### $addFields

[$addFields](https://docs.mongodb.com/manual/reference/operator/aggregation/addFields/#mongodb-pipeline-pipe.-addFields)
는 입력 `Document` 에 새로운 필드를 추가한다. 출력 `Document` 는 기존 모든 필드와 신규 필드를 포함한다. ($project 와 유사)

- format

  ```json
  { $addFields: { <newField>: <expression>, ... } }
  ```  

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
은 입력 `Document` 에 대해 지정된 식에 따라 `bucket` 이라는 그룹으로 분류한다. 그리고 각 `Bucket` 별로 `Document` 를 출력한다.

버킷 `Stage` 에서 `RAM` 은 최대 `100MB` 까지만 사용 할 수 있다. 초과하게 되면 오류를 반환하고, 더 많은 공간이 필요한 경우 `allowDiskUse` 사용해서 가능하다.

- format

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
  {_id: 1, strType: "a", numType: 10, value: 1},
  {_id: 2, strType: "b", numType: 20, value: 2},
  {_id: 3, strType: "c", numType: 30, value: 4},
  {_id: 4, strType: "d", numType: 40, value: 8},
  {_id: 5, strType: "e", numType: 50, value: 16},
  {_id: 6, strType: "f", numType: 60, value: 32},
  {_id: 7, strType: "g", numType: 70, value: 64},
  {_id: 8, strType: "h", numType: 80, value: 128}
]);
```  

```bash
.. type3 을 기준으로 버킷을 그룹화 하고, 그룹화 된 문서의 수에 따라 필터링 ..
> db.test.aggregate([
{ 
  $bucket: { 
    groupBy: "$numType", 
    boundaries: [10, 40, 60], 
    default: "Other", 
    output: {
     "groupCount" : {$sum: 1},
     "embeddedDoc" : { 
       $push: {
         "concat1" : {$concat: ["$strType", "-concat"]},
         "value" : "$value",
         "avgValue" : {$avg: "$value"} 
       } 
     }
    } 
  } 
}, 
{ 
  $match: {groupCount: {$gt: 2}} 
}
]).pretty();

.. output .. 
{
        "_id" : 10,
        "groupCount" : 3,
        "embeddedDoc" : [
                {"concat1" : "a-concat","value" : 1,"avgValue" : 1},
                {"concat1" : "b-concat","value" : 2,"avgValue" : 2},
                {"concat1" : "c-concat","value" : 4,"avgValue" : 4}
        ]
}
{
        "_id" : "Other",
        "groupCount" : 3,
        "embeddedDoc" : [
                {"concat1" : "f-concat","value" : 32,"avgValue" : 32},
                {"concat1" : "g-concat","value" : 64,"avgValue" : 64},
                {"concat1" : "h-concat","value" : 128,"avgValue" : 128}
        ]
}
```  

```bash
.. $facet 을 이용해서 한 Stage 에서 2개 그룹(strType, numType) 수행 ..
> db.test.aggregate([
{
  $facet: {
    "strTypeGroup": [
      {
        $bucket: {
          groupBy: "$strType",
          boundaries: ["b", "e", "g"],
          default: "Other",
          output: {
           "groupCount" : {$sum: 1},
           "embeddedDoc" : { 
             $push: {
               "pow" : {$pow: ["$numType", 2]},
               "value" : "$value",
               "avgValue" : {$avg: "$value"} 
             } 
           }
          } 
        }
      }
    ],
    "numTypeGroup": [
      {
        $bucket: {
          groupBy: "$numType",
          boundaries: [10, 40, 60],
          default: "Other",
          output: {
           "groupCount" : {$sum: 1},
           "embeddedDoc" : { 
             $push: {
               "concat1" : {$concat: ["$strType", "-concat"]},
               "value" : "$value",
               "avgValue" : {$avg: "$value"} 
             } 
           }
          } 
        }
      }
    ],
  }
}
]).pretty();

.. output ..
{
        "strTypeGroup" : [
                {
                        "_id" : "Other",
                        "groupCount" : 3,
                        "embeddedDoc" : [
                                {"pow" : 100,"value" : 1,"avgValue" : 1},
                                {"pow" : 4900,"value" : 64,"avgValue" : 64},
                                {"pow" : 6400,"value" : 128,"avgValue" : 128}
                        ]
                },
                {
                        "_id" : "b",
                        "groupCount" : 3,
                        "embeddedDoc" : [
                                {"pow" : 400,"value" : 2,"avgValue" : 2},
                                {"pow" : 900,"value" : 4,"avgValue" : 4},
                                {"pow" : 1600,"value" : 8,"avgValue" : 8}
                        ]
                },
                {
                        "_id" : "e",
                        "groupCount" : 2,
                        "embeddedDoc" : [
                                {"pow" : 2500,"value" : 16,"avgValue" : 16},
                                {"pow" : 3600,"value" : 32,"avgValue" : 32}
                        ]
                }
        ],
        "numTypeGroup" : [
                {
                        "_id" : 10,
                        "groupCount" : 3,
                        "embeddedDoc" : [
                                {"concat1" : "a-concat","value" : 1,"avgValue" : 1},
                                {"concat1" : "b-concat","value" : 2,"avgValue" : 2},
                                {"concat1" : "c-concat","value" : 4,"avgValue" : 4}
                        ]
                },
                {
                        "_id" : 40,
                        "groupCount" : 2,
                        "embeddedDoc" : [
                                {"concat1" : "d-concat","value" : 8,"avgValue" : 8},
                                {"concat1" : "e-concat","value" : 16,"avgValue" : 16}
                        ]
                },
                {
                        "_id" : "Other",
                        "groupCount" : 3,
                        "embeddedDoc" : [
                                {"concat1" : "f-concat","value" : 32,"avgValue" : 32},
                                {"concat1" : "g-concat","value" : 64,"avgValue" : 64},
                                {"concat1" : "h-concat","value" : 128,"avgValue" : 128}
                        ]
                }
        ]
}
```  

### $bucketAuto
[$bucketAuto](https://docs.mongodb.com/manual/reference/operator/aggregation/bucketAuto/#mongodb-pipeline-pipe.-bucketAuto)
는 지정된 식에 맞춰 `Document` 를 분류한다. 
지정된 수에 맞춰 `Document` 를 균등분배 하기위해 `boundaries` 는 자동으로 결정된다.  

`bucketAuto` 에서 출력하는 각 `bucket` 의 필드 구성은 아래와 같다.  

- `_id` : 버킷을 경계를 지정한다. 
  - `_id.min` : 버킷 하한(포함)
  - `_id.max` : 버킷 상한(포함)
- `count` : 버킷에 해당하는 문서 수

버킷 `Stage` 에서 `RAM` 은 최대 `100MB` 까지만 사용 할 수 있다. 초과하게 되면 오류를 반환하고, 더 많은 공간이 필요한 경우 `allowDiskUse` 사용해서 가능하다.


- format

  ```json
  {
    $bucketAuto: {
        groupBy: <expression>,
        buckets: <number>,
        output: {
           <output1>: { <$accumulator expression> },
           ...
        }
        granularity: <string>
    }
  }
  ```  
  
[granularity](https://docs.mongodb.com/manual/reference/operator/aggregation/bucketAuto/#granularity)
를 사용하면 자동 분류를 좀더 세밀하게 조정할 수 있다.  


```bash
> db.test.insertMany([
  {_id: 1, strType: "a", numType: 10, value: 1},
  {_id: 2, strType: "b", numType: 20, value: 2},
  {_id: 3, strType: "c", numType: 30, value: 4},
  {_id: 4, strType: "d", numType: 40, value: 8},
  {_id: 5, strType: "e", numType: 50, value: 16},
  {_id: 6, strType: "f", numType: 60, value: 32},
  {_id: 7, strType: "g", numType: 70, value: 64},
  {_id: 8, strType: "h", numType: 80, value: 128},
  {_id: 9, strType: "i", numType: 90, value: 256},
  {_id: 10, strType: "j", numType: 100, value: 512},
  {_id: 11, strType: "k", numType: 110, value: 1024},
  {_id: 12, strType: "l", numType: 120, value: 2048},
  {_id: 13, strType: "m", numType: 130, value: 4096},
  {_id: 14, strType: "n", numType: 140, value: 8192},
  {_id: 15, strType: "o", numType: 150, value: 16384},
]);
```  

```bash
.. _id 필드를 기준으로 총 5개의 버킷으로 분류 ..
> db.test.aggregate([
{
  $bucketAuto: {
    groupBy: "$_id",
    buckets: 5,
  }
}
]).pretty();

.. no granularity output ..
{ "_id" : { "min" : 1, "max" : 4 }, "count" : 3 }
{ "_id" : { "min" : 4, "max" : 7 }, "count" : 3 }
{ "_id" : { "min" : 7, "max" : 10 }, "count" : 3 }
{ "_id" : { "min" : 10, "max" : 13 }, "count" : 3 }
{ "_id" : { "min" : 13, "max" : 15 }, "count" : 3 }

.. granularity : R20 output ..
{ "_id" : { "min" : 0.9, "max" : 3.15 }, "count" : 3 }
{ "_id" : { "min" : 3.15, "max" : 6.3 }, "count" : 3 }
{ "_id" : { "min" : 6.3, "max" : 10 }, "count" : 3 }
{ "_id" : { "min" : 10, "max" : 12.5 }, "count" : 3 }
{ "_id" : { "min" : 12.5, "max" : 16 }, "count" : 3 }

.. granularity : E24 output ..
{ "_id" : { "min" : 0.91, "max" : 3.3000000000000003 }, "count" : 3 }
{ "_id" : { "min" : 3.3000000000000003, "max" : 6.2 }, "count" : 3 }
{ "_id" : { "min" : 6.2, "max" : 9.1 }, "count" : 3 }
{ "_id" : { "min" : 9.1, "max" : 13 }, "count" : 3 }
{ "_id" : { "min" : 13, "max" : 16 }, "count" : 3 }

.. granularity : 1-2-5 output(specific interval) ..
{ "_id" : { "min" : 0.5, "max" : 5 }, "count" : 4 }
{ "_id" : { "min" : 5, "max" : 10 }, "count" : 5 }
{ "_id" : { "min" : 10, "max" : 20 }, "count" : 6 }

.. granularity : POWERSOF2 output ..
{ "_id" : { "min" : 0.5, "max" : 4 }, "count" : 3 }
{ "_id" : { "min" : 4, "max" : 8 }, "count" : 4 }
{ "_id" : { "min" : 8, "max" : 16 }, "count" : 8 }
```  

```bash
.. $facet 을 이용해서 한 Stage 에서 2개 그룹(strType, numType) 수행 ..
> db.test.aggregate([
{
  $facet: {
    "strTypeGroup" : [
      {
        $bucketAuto: {
          groupBy: "$strType",
          buckets: 6
        }
      }
    ],
    "numTypeGroup" : [
      {
        $bucketAuto: {
          groupBy: "$numType",
          buckets: 4,
          output: {
            "count" :{$sum:1},
            "numArray" : {$push:"$numType"}
          }
        }
      }
    ]
  }
}
]).pretty();

.. output ..
{
        "strTypeGroup" : [
                {"_id" : {"min" : "a","max" : "d"},"count" : 3},
                {"_id" : {"min" : "d","max" : "g"},"count" : 3},
                {"_id" : {"min" : "g","max" : "j"},"count" : 3},
                {"_id" : {"min" : "j","max" : "m"},"count" : 3},
                {"_id" : {"min" : "m","max" : "o"},"count" : 3}
        ],
        "numTypeGroup" : [
                {
                        "_id" : {"min" : 10,"max" : 50},
                        "count" : 4,
                        "numArray" : [10,20,30,40]
                },
                {
                        "_id" : {"min" : 50,"max" : 90},
                        "count" : 4,
                        "numArray" : [50,60,70,80]
                },
                {
                        "_id" : {"min" : 90,"max" : 130},
                        "count" : 4,
                        "numArray" : [90,100,110,120]
                },
                {
                        "_id" : {"min" : 130,"max" : 150},
                        "count" : 3,
                        "numArray" : [130,140,150]
                }
        ]
}
```  


### $count
[$count](https://docs.mongodb.com/manual/reference/operator/aggregation/count/#mongodb-pipeline-pipe.-count)
는 `Stage` 에 입력된 `Document` 의 수를 다음 `Stage` 로 전달한다.  

- format

  ```json
  { $count: <string> }
  ```  

`<string>` 은 카운트를 값으로 가지는 출력 필드 이름이다. 
빈문자열, `$`, `.` 문자열 포함할 수 없다.  


```bash
> db.test.insertMany([
  {_id: 1, strType: "a", numType: 10, value: 1},
  {_id: 2, strType: "b", numType: 20, value: 2},
  {_id: 3, strType: "c", numType: 30, value: 4},
  {_id: 4, strType: "d", numType: 40, value: 8},
  {_id: 5, strType: "e", numType: 50, value: 16},
  {_id: 6, strType: "f", numType: 60, value: 32},
  {_id: 7, strType: "g", numType: 70, value: 64},
  {_id: 8, strType: "h", numType: 80, value: 128},
  {_id: 9, strType: "i", numType: 90, value: 256},
  {_id: 10, strType: "j", numType: 100, value: 512},
  {_id: 11, strType: "k", numType: 110, value: 1024},
  {_id: 12, strType: "l", numType: 120, value: 2048},
  {_id: 13, strType: "m", numType: 130, value: 4096},
  {_id: 14, strType: "n", numType: 140, value: 8192},
  {_id: 15, strType: "o", numType: 150, value: 16384},
]);
```  

```bash
> db.test.aggregate([
  {
    $count: "all_count"
  }
]);

.. output ..
{ "all_count" : 15 }


> db.test.aggregate([
  {
    $match: {
      value: {$gt: 1000}
    }
  },
  {
    $count: "gt_1000_count"
  }
]);

.. output ..
{ "gt_1000_count" : 5 }
```  




---

## Reference

[MongoDB Aggregation](https://docs.mongodb.com/manual/aggregation/)  








