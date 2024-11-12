--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Transforms(SMT) 3rd"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect 에서 데이터를 변환/필터링 할 수 있는 SMT 중 ExtractTopic, Filter, TimestampRouter 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - ExtractTopic
    - Filter
    - TimestampRouter
toc: true
use_math: true
---

## Kafka Connect Transforms
[Kafka Connect Transforms(SMT) 1]({{site.baseurl}}{% link _posts/kafka/2024-10-23-kafka-practice-kafka-connect-transforms-1.md %}),
[Kafka Connect Transforms(SMT) 2]({{site.baseurl}}{% link _posts/kafka/2024-10-23-kafka-practice-kafka-connect-transforms-2.md %}),
에 이어서 추가적인 `Transforms` 의 사용법에 대해 알아본다. 
테스트를 위해 구성하는 환경와 방식은 위 첫번째 포스팅과 동일하다. 
테스트 및 예제 진행에 궁금한 점이 있다면 이전 포스팅에서 관련 내용을 확인 할 수 있다.  

### ExtractTopic
[ExtractTopic](https://docs.confluent.io/platform/current/connect/transforms/extracttopic.html)
을 사용하면 메시지에서 특정 필드를 추출해 토픽이름으로 지정 할 수 있다. 
하나의 필드만 지정 할 수도 있고, [JSON Path](https://github.com/json-path/JsonPath)
을 사용해서 충첩구조의 지정도 가능하다.  

- Key : `io.confluent.connect.transforms.ExtractTopic$Key`
- Value : `io.confluent.connect.transforms.ExtractTopic$Value`
- Header : `io.confluent.connect.transforms.ExtractTopic$Header`

`extract-topic-input.txt` 파일의 내용은 아래와 같다.  

```
exam-topic-1
exam-topic-2
exam-topic-3
exam-topic-1
exam-topic-2
exam-topic-3
```  
