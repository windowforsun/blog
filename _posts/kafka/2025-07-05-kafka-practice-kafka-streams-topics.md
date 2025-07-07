--- 
layout: single
classes: wide
title: "[Kafka] "
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Streams
toc: true
use_math: true
---  

## Kafka Streams Topics
`Kafka Streams` 애플리케이션은 `Kafka Broker` 의 다양한 `Topic` 들과 지속적으로 데이터를 주고 받으면 데이터 스트림을 구현한다. 
이는 사용자가 필요에 의해 정의한 명시적인 `Topic` 도 포함되지만 `Kafka Streams` 에서 필요에 의해서 생성되는 `Topic` 도 포함된다. 
이번 포스팅에서는 `Kafka Streams` 애플리케이션을 구현하고 사용할 때 어떠한 유형의 `Topic` 들이 사용되고 필요하지에 대해 알아본다. 
크게 아래 2가지의 `Topic` 으로 구분될 수 있다.  

- `User Topics`
- `Internal Topics`


### User Topics
`User Topics` 는 `Kafka Streams Application` 에서 외적으로 사용하는 `Topic` 으로, 
애플리케이션에 의해 읽히거나 기록되는 토픽을 의미한다. 
즉 사용자의 필요에 따라 생성되고 관리되는 `Topic` 이라고 할 수 있다. 
그 상세한 종류는 아래와 같다.  

- `Input Topics` : 애플리케이션이 데이터를 읽어오는 소스 토픽이다. 일반적으로 외부 시스템이나 다른 애프릴케이션에서 생성한 데이터가 포함된다. 스트림 애플리케이션의 시작점 역할을 하게 된다. (e.g. `StreamsBuilder.stream()`, `StreamsBuilder.table()`, `Topology.addSource()`)
- `Output Topics` : 애플리케이션이 처리 결과를 쓰는 목적지 토픽이다. 처리된 데이터나 분석 결과가 저장된다. 다른 시스템이나 애플리케이션에서 이 토픽 데이터를 소비할 수 있다. (e.g. `KStream.to()`, `KTable.to()`, `Topology.addSink()`)
- `Intermediate Topics` : 애플리케이션 내에서 입력/출력 모두로 사용되는 토픽이다. 복잡한 처리 파이프라인에서 중간 결과를 저장하는 데 사용된다. 다단계 처리나 여러 애플리케이션 간의 데이터 전달에 활용할 수도 있다. (e.g. `KStream.through()`)

> `Kafka Streams 3.0` 이하인 경우 `KStream.through()` 를 사용해 `Intermediate Topics` 를 생성해 사용할 수 있었다. 
> 하지만 `3.0` 버전 이후 부터는 `Depreated` 되어 관련된 처리는 `KStream.repartition()` 이 새로 생겼다. 
> 그 상세한 내용을 정리하면 아래와 같다. 
> 간략한 내용은 두 가지 모두 사용 목적은 `중간 처리 단계 데이터 저장` 이지만, 관리 방식이 좀 더 용의하게 변경됐다. 
> 
> | 특성               | `repartition()` 토픽             | `through()` 토픽                |
> |-------------------|----------------------------------|---------------------------------|
> | **관리 방식**      | 자동 관리                        | 수동 관리                       |
> | **토픽 종류**      | 내부 토픽 (internal topic)      | 사용자 토픽 (user topic)       |
> | **가시성**         | 내부적으로 숨겨짐                | 명시적으로 보임                 |
> | **네이밍**         | 자동 이름 생성                   | 개발자가 이름 지정             |
> | **역할**           | 중간 처리 단계의 데이터 저장    | 중간 처리 단계의 데이터 저장    |


`User Topics` 사용/관리에 권장하는 방법은 사전에 직접 생성하고 수동으로 관리해야 한다는 것이다. (`kafka-topics` 툴 사용 등)
즉 `Kafka Broker` 의 토픽 자동 생성 기능으로 생성하도록 하는 것은 권장하지 않는다는 의미한데, 그 이유는 아래와 같다.  

- `Kafka Broker` 에서 토픽 자동 생성이 비활성화 돼 있을 수 있다. 
- `auto.create.topics.enable=true` 로 설정된 경우 생성되는 토픽은 기본 토픽 설정을 바탕으로 구성된다. 기본 설정으로 토픽이 생성될 경우 필요한 설정과는 차이가 있으 수 있기 때문이다. 

