--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Intro"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LLM 기반의 복잡한 워크플로우와 멀티에이전트 협업을 그래프 구조로 설계하고 구현할 수 있는 AI Framework 인 LangGraph 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - RAG
    - Multi-Agent
    - LLM
    - AI
    - Graph
    - Workflow
    - AI Framework
toc: true
use_math: true
---  

## LangGraph
`LangGraph` 는 대규모 언어 모델(`LLM`)을 기반으로 복잡한 워크플로우,
멀티에이전트 협업, 그리고 다양한 인공지능 작업을 그래프 구조로 설계하고 구현할 수 있도록 돕는 `AI Framework` 이다.
`LangChain` 은 순차적 `Chain` 구조를 넘어, 비순차적 흐름, 반복, 조건 분기, 상태 관리 등
실제 서비스와 다양한 환경에서 요구되는 복잡한 로직을 직관적이고 유연하게 다룰 수 있게 한다.

### LangGraph Features

#### Graph-Based Workflow
`LangGraph` 은 각 작업(task), 노드(node), 에이전트(agent), 체인(chain) 등을 그래프의 `Node` 로 표현한다.
`Edge` 는 노드 간 데이터 및 제어 흐름, 즉 에이전트 간 상호작용을 정의한다.
이러한 구조를 통해 데이터 흐름, 조건 분기(`if-else`), 반복(`loop`), 별렬 처리 등이 작관적으로 구현되어 복잡한 워크플로우나 멀티에이전트 시스템도
자연스럽게 설계할 수 있다.


#### Flexible Branching, Looping, State Management
조건 분기, 반복 등 프로그래밍의 흐름 제어를 그래프로 손쉽게 구현할 수 있다.
`State` 를 명시적으로 정의하고 주고받을 수 있어, 각 노드-에이전트의 작업 결과나 컨텍스트가 장기적으로 관리될 수 있다.
상태는 파이썬 딕셔너리 등으로 자유롭게 정의되며, 장기 맥락 유지, 오류 복구, 체크포인팅 등에 활용된다.


#### Multi-Agent Support and Human-agent Collaboration
여러 `LLM` 에이전트가 서로 협력, 경쟁, 정보 교환을 할 수 있도록 설계할 수 있다.
각 에이전트의 역할을 그래프 내에서 명확하게 분리 정리가 가능하고,
인간의 개입(`Human-in-the-loop`) 이 필요한 시나리오도 쉽게 구현할 수 있다.
에이전트의 행동을 추적하고, 필요한 경우 과거 상태로 되돌리거나 다른 경로로 분기하는 `Time Travel` 기능도 가능하다.

#### Integration with LangChain and Scalability
`LangChain` 의 모든 체인, 에이전트, 툴 등을 `LangGraph` 의 노드로 활용할 수 있다.
기존 `LangChain` 사용자라면 손쉽게 마이그레이션 및 확장이 가능하다.
다양한 언어 모델, 외부 도구, 데이터 소스를 자유롭게 연결할 수 있어 멀티모달(텍스트, 음성, 이미지 등) 데이터 처리도 가능하다.


#### Reliability and Quality Management
중간중간 품질 체크, `Moderation`, 오류 복구 등 실제 운영 환경에서 요구되는 신뢰성 확보 기능이 내장돼 있다.
그래프 구조로 워크플로우를 시각화할 수 있어 로직 파악과 유지보수가 쉽고, 직관적인 디버깅이 가능하다.


### LangGraph Use Cases
- 복잡한 대화형 AI 시스템 : 여러 에이전트가 협력하는 챗봇, 상담 시스템 등
- 데이터 처리 및 자동화 : 대량의 데이터를 다양한 모델과 도구로 병렬 처리 및 분석
- 맞춤형 AI 솔류션 설계 : 특정 비지니스 요구에 맞춘 워크플로우 및 의사결정 시스템 구축
- `RAG` 시스템 : 외부 지식 검색과 생성형 `AI` 의 결합
- 멀티 모달 `AI` : 텍스트, 이미지, 음성 등 다양한 데이터 유형을 통합한 복합 AI 서비스
- 조건 분기/반복이 필요한 `LLM` 파이프라인 : 사용자의 입력에 따라 다양한 응답 경로나 반복적 처리가 필요한 시스템
- 멀티에이전트 협업 시나리오 : 여러 `LLM` 이 서로 다른 역할을 맡아 협동(e.g. 질문 분류-전문 응답-걀과 종합 등)

### LangGraph core Concepts

| 용어      | 설명                                                         |
|-----------|--------------------------------------------------------------|
| 노드(Node)   | 그래프 내의 한 작업 단위. LLM 실행, 함수 호출, 조건 분기 등 |
| 엣지(Edge)   | 노드 간의 연결. 데이터/상태가 이동하는 경로, 분기·반복·병렬 등 제어 흐름 |
| 상태(State)  | 그래프 실행 중 주고받는 정보(딕셔너리 등), 맥락·데이터·진행 상황 등 |
| 그래프(Graph)| 전체 워크플로우 구조. 여러 노드/엣지로 구성됨                |
| 멀티에이전트(Multi-Agent) | 여러 에이전트가 상호작용하는 구조                |

### LangGraph vs LangChain

| 구분          | LangGraph                          | LangChain                       |
|---------------|------------------------------------|----------------------------------|
| 구조          | 그래프(노드-엣지) 기반             | 체인(Chain) 및 에이전트 기반    |
| 상태 관리     | 명시적, 세밀한 제어                | 암시적, 자동화된 관리           |
| 유연성        | 매우 높음 (커스텀 로직 자유로움)   | 중간 (미리 정의된 컴포넌트 중심)|
| 주요 목적     | 복잡한 워크플로우, 의사결정        | LLM 통합, 체인 구성             |





---  
## Reference
[LangGraph](https://langchain-ai.github.io/langgraph/)  
[LangGraph Github](https://github.com/langchain-ai/langgraph)  
[What is LangGraph?](https://www.ibm.com/think/topics/langgraph)  
[LangGraph Reference](https://langchain-ai.github.io/langgraph/reference/)  
[LangGraph Tutorial](https://github.com/langchain-ai/langgraph/tree/main/docs/docs/tutorials)  
[LangGraph Docs](https://github.com/langchain-ai/langgraph/tree/main/docs/docs)  