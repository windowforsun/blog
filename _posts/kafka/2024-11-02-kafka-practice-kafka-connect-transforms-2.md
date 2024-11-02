--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Transforms(SMT) 2"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect 에서 데이터를 변환/필터링 할 수 있는 SMT 중 '
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Retry
    - Non Blocking
    - Consumer
    - ExtractField
    - ReplaceField
    - TimestampConverter
    - InsertHeader
    - DropHeader
    - HeaderFrom
    - Flatten
toc: true
use_math: true
---

## Kafka Connect Transforms
[Kafka Connect Transforms(SMT) 1]({{site.baseurl}}{% link _posts/kafka/2024-10-23-kafka-practice-kafka-connect-transforms-1.md %})
에 이어서 추가적인 `Transforms` 의 사용법에 대해 알아본다. 
테스트를 위해 구성하는 환경와 방식은 위 첫번째 포스팅과 동일하다. 
테스트 및 예제 진행에 궁금한 점이 있다면 이전 포스팅에서 관련 내용을 확인 할 수 있다.

