--- 
layout: single
classes: wide
title: "[Elasticsearch 실습] "
header:
  overlay_image: /img/elasticsearch-bg.png
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Elasticsearch
tags:
    - Practice
    - Elasticsearch
toc: true
use_math: true
---  

## ELK Stack
`ELK Stack` 이란 `Elasticsearch`, `Logstash`, `Kibana` 의 조합을 통칭하는 기술 스택을 의미한다. 
`Elasticsearch` 는 `JSON` 기반 분산 오픈소스 검색 및 분석 엔진이다. 
`Logstash` 는 여러 소스에서 동시에 데이터를 수집하여 변환 수행 후, 
`Elasticsearch` 같은 `stash` 로 전송하는 서버사이드 데이터 파이프라인이다. 
그리고 `Kibana` 는 `Elasticsearch` 에서 차트와 그래프를 이용해 데이터를 시각화 할 수 있게 해주는 툴이다. 
마지막으로 `ELK` 에서 보편적으로 추가되는 것이 바로 `Filebeat` 이다. 
여기서 `Filebeat` 은 서버에서 에이전트로 설치 되어 다양한 유형의 데이터를 `Elasticsearch` 혹은 `Logstash` 로 전송하는 오픈 소스 데이터 발송자를 의미힌다.  

`Logstash` 와 `Filebeat` 모두 데이터를 `Elasticsearch` 로 전송한다는 목적에서는 동일한 역할을 하지만 아래와 같은 차이가 있다.  

- `Logstash` : 비교적 많은 자원을 사용해서 다를 수 있는 `input`, `output` 유형(종류)가 많고, `filter` 를 사용해서 로그(데이터)를 분석하기 쉽게끔 구조화 된 형식으로 변환 가능하다. 
- `Filebeat` : 가벼운 대신 가능한 `input`, `output` 종류가 한정적이다. 설정에 지정된 로그 데이터를 바라보는 하나 이상의 `input` 을 가진다. 지정도니 로그 파일에서 이벤트(데이터 변경)가 바생 할때 마다 데이터 수확을 시작한다. 

구성을 하나더 추가한다면 `Kafka` 가 중간에 들어갈 수 있지만, 
이번 포스트에서는 `Kafka` 는 제외한 `ELK Stack` 에 대해서 간단한 구성 방법을 알아본다. 
여기서 `Kafka` 역할에 대해서만 간단하게 알아보면, 중간 버퍼 역할을 한다고 할 수 있다. 
순간적으로 급증하는 데이터양에 대한 버퍼도 될 수 있고, 특정 시스템이 다운 됐을 때 로그의 버퍼 역할도 할 수 있다.  

구성할 `ELK` 의 간단한 예시는 아래와 같다.  





---  
## Reference
[]()  
