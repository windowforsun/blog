--- 
layout: single
classes: wide
title: "GeoJSON Format"
header:
  overlay_image: /img/blog-bg.jpg
excerpt: 'JSON 타입으로 지리좌표 공간(위치정보)을 표현 할 수 있는 GeoJSON 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - ETC
tags:
  - ETC
  - format
  - GeoJSON
  - Point
  - LineString
  - Polygon
  - MultiPolygon
  - MultiPoint
  - MultiLineString
  - GeometryCollection
toc: true
use_math: true
---  

## GeoJSON
`GeoJSON` 은 위치정보를 `JSON` 의 파일 포맷을 사용하는 지형을 포현하기 위해 설계된 형식이다. 
지리좌표계, 지형지물(주소, 위치), 라인스트링(거리, 고속도로, 경계), 폴리라인, 다각형(국가, 도시, 토지) 등을 표현할 수 있다. 
앞서 나열한 표현들은 모두 각각의 타입으로 구분되어 진다.  

```
{ type: <GeoJSON type> , coordinates: <coordinates> }
```  

- `type` : `GeoJSON` 이 표현할 수 있는 구체적인 타입을 지정한다. 
  - 사용할 수 있는 타입은 다음과 같다. `Point`, `LineString`, `Polygon`, `MultiPoint`, `MultiLineString`, `MultiPolygon`, `GeometryCollection`
- `coordinates` : 좌표를 지정하는 필드로 위도와 경도를 사용해서 좌표를 나타낼 수 있고 경도(`longitude`), 위도(`latitude`) 순으로 작성한다. 


### Point
`Point` 는 단일 좌표(`coordinates`) 위치를 나타내는 표현이다.  

```json
{
    "type": "Point",
    "coordinates": [30, 10]
}
```  


![그림 1]({{site.baseurl}}/img/etc/concept-geojson-1.svg.png)


### LineString(PolyLine)
`LineString` 은 두개 이상으로 구성된 좌표(`coordinates`) 배열을 이어서 한 선을 표현한다. 


```json
{
    "type": "LineString",
    "coordinates": [
        [30, 10], [10, 30], [40, 40]
    ]
}
```

![그림 1]({{site.baseurl}}/img/etc/concept-geojson-2.svg.png)

### Polygon
`Polygon` 은 4개 이상으로 구성된 좌표(`coordinates`) 배열을 이어 하나의 닫힌 도형을 표현 한다. 
즉 `Polygon` 은 여러 `LineString` 데이터 형태로 구성된다. 
그리고 좌표 배열에서 첫번째 요소와 마지막 요소는 동일한 좌표여야 한다.  

#### Single Ring
`Polygon Single Ring` 은 단일 도형을 표현하는 `Polygon` 을 의미한다. 

```json
{
    "type": "Polygon",
    "coordinates": [
        [[30, 10], [40, 40], [20, 40], [10, 20], [30, 10]]
    ]
}
```  

![그림 1]({{site.baseurl}}/img/etc/concept-geojson-3.svg.png)

#### Multiple Rings
`Polygon Mutiple Rings` 는 외부 도형과 내부 도형이 구분되는 형태를 의미한다. 

- 가장 첫번째 도형은 가장 외부 도형을 나타내야 한다. 
- 외부 도형은 자체 교차할 수 없다.
- 모든 내부 도형은 외부 도형 내부에 존재해야 한다. 
- 내부 도형은 다른 도형과 교차하거나 겹칠 수 없다. 
- 내부 도형은 다른 도형과 모서리를 공유 할 수 없다. 

```json
{
    "type": "Polygon",
    "coordinates": [
        [[35, 10], [45, 45], [15, 40], [10, 20], [35, 10]],
        [[20, 30], [35, 35], [30, 20], [20, 30]]
    ]
}
```  

![그림 1]({{site.baseurl}}/img/etc/concept-geojson-4.svg.png)

### MultiPoint
`MultiPoint` 는 `Point` 타입 배열을 사용해서 여러 좌표위치를 나타내는 표현이다. 

```json
{
    "type": "MultiPoint",
    "coordinates": [
        [10, 40], [40, 30], [20, 20], [30, 10]
    ]
}
```  

![그림 1]({{site.baseurl}}/img/etc/concept-geojson-5.svg.png)


### MultiLineString
`MultiLineString` 은 여러 `LineString` 타입 배열을 사용해서 여러 선을 표현한다.  

```json
{
    "type": "MultiLineString",
    "coordinates": [
        [[10, 10], [20, 20], [10, 40]],
        [[40, 40], [30, 30], [40, 20], [30, 10]]
    ]
}
```  

![그림 1]({{site.baseurl}}/img/etc/concept-geojson-6.svg.png)


### MultiPolygon
`MultiPolygon` 은 여러 `Polygon` 타입 배열을 사용해서 여러 닫힌 도형을 표현한다.  

#### Multi Single Ling

```json
{
    "type": "MultiPolygon",
    "coordinates": [
        [
            [[30, 20], [45, 40], [10, 40], [30, 20]]
        ],
        [
            [[15, 5], [40, 10], [10, 20], [5, 10], [15, 5]]
        ]
    ]
}
```  

![그림 1]({{site.baseurl}}/img/etc/concept-geojson-7.svg.png)

#### Multi Multiple Rings

```json
{
    "type": "MultiPolygon",
    "coordinates": [
        [
            [[40, 40], [20, 45], [45, 30], [40, 40]]
        ],
        [
            [[20, 35], [10, 30], [10, 10], [30, 5], [45, 20], [20, 35]],
            [[30, 20], [20, 15], [20, 25], [30, 20]]
        ]
    ]
}
```  

![그림 1]({{site.baseurl}}/img/etc/concept-geojson-8.svg.png)

### GeometryCollection
`GeometryCollection` 은 `geometries` 라는 새로운 배열 필드를 사용한다. 
`geometires` 는 지금까지 설명한 다양한 타입을 오브젝트로 가질 수 있어서, 복합적인 타입을 표현할 수 있다. 
그리고 `geometires` 배열 필드는 비어 있을 수 있다.  

```json
{
    "type": "GeometryCollection",
    "geometries": [
        {
            "type": "LineString",
            "coordinates": [
                [30, 10], [10, 30], [40, 40]
            ]
        },
        {
            "type": "Polygon",
            "coordinates": [
                [[30, 10], [40, 40], [20, 40], [10, 20], [30, 10]]
            ]
        },
        {
            "type": "MultiPolygon",
            "coordinates": [
                [
                    [[40, 40], [20, 45], [45, 30], [40, 40]]
                ],
                [
                    [[20, 35], [10, 30], [10, 10], [30, 5], [45, 20], [20, 35]],
                    [[30, 20], [20, 15], [20, 25], [30, 20]]
                ]
            ]
        }
    ]
}
```  





---

## Reference
[The GeoJSON Format](https://datatracker.ietf.org/doc/html/rfc7946)  
[GeoJSON Objects](https://docs.mongodb.com/manual/reference/geojson)  
[GeoJSON](https://ko.wikipedia.org/wiki/GeoJSON)  








