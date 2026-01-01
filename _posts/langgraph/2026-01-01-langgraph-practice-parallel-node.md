--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Agent Memory and Stream"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
toc: true
use_math: true
---  

## LangGraph Parallel Node
그래프 상에서 두 개 이상의 노드를 동시에(병렬로) 실행하여, 
전체 처리 속도를 높이거나 다양한 경로로 데이터를 처리할 수 있도록 할 수 있다. 
일반적으로 워크플로우는 직렬(순차적)로 노드가 실행되지만, 
서로 의존성이 없는 작업은 병렬 실행이 효율적일 수 있다. 
`LangGraph` 에서는 한 노드에 대한 분기(`edge`)를 여러 노드로 연결하고, 
여러 노드로 분기된 `edge` 가 다시 하나의 노드로 합쳐지는 구조로 병렬 처리를 구현할 수 있다.  

`LangGraph` 에서는 병렬 처리를 위해 `fan-out` 과 `fan-in` 이라는 구조와 개념을 사용한다. 
이는 복잡한 작업을 효율적으로 처리할 때 사용하는 전형적인 병렬 처리 패턴이다.  

- `fan-out`(확장)
  - 큰 작업을 여러 개의 더 작은, 독립적인 작업으로 나누어 동시에 처리하는 과정이다. 
  - 이는 마치 요리를 만들 때 필요한 각각의 제료 손질을 병렬로 진행하는 것과 비슷하다.  
- `fan-in`(수집)
  - 여러 경로로 나눠졌던 작업(또는 데이터)이 모두 끝나면, 이 결과를 다시 하나로 모으는 과정이다.  
  - 각각 준비한 재료를 한데 모아 요리를 완성하는 것과 같다.

병렬 처리된 결과를 다시 합칠 때 단순히 값을 덮어쓰는 것이 아니라,
여러 결과를 누적하거나 결합해야 하는 경우가 있다. 
이런 경우 그래프 상태에 `reducer` 의 `add` 연산자인 `add_message` 를 사용해, 
여러 노드의 결과가 리스트라면, 각 리스트를 이어붙이는 식으로 집계할 수 있다.  

`LangGraph` 에서는 타입 자체를 바꾸지 않고, 
타입에 `reducer` 함수를 `주석`처럼 첨부할 수 있도록 `Annotated` 를 사용한다. 
리스트 타입에 `add reducer`(`add_message`) 를 주석으로 달면, 해당 카에 값을 추가할 때마다 리스트를 자동으로 이어붙이게 동작한다. 
이렇게 하면 병렬 브랜치에서 온 여러 결과를 자연스럽게 하나로 모을 수 있다.  

예제로 진행할 예시 구조는 다음과 같다. 

- `fan-out` : `begin` 노드에서 `parallel-1`, `parallel-2` 노드로 분기
- `fan-in` : `parallel-1`, `parallel-2` 노드에서 `agg` 노드로 결합
