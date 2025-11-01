--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Agent Memory and Stream"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LangGraph 에서 주어진 목표를 수행하는 Agent 구현과정에서 Agent Memory 와 Stream 기능에 대해 알아본다.'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - Agent
    - Memory
    - Stream
    - Tool
    - Persistent Checkpoint
    - DuckDuckGo
    - Google Gemini
    - LLM
toc: true
use_math: true
---  

## LangGraph Agent Memory and Stream
`LangGraph` 에서 `Agent` 는 특정 작업을 수행하는 독립적인 실행 단위로, 
각 `Agent` 는 자신의 상태(`State`)를 유지하고, 다른 `Agent` 와 상호작용하며, 
작업을 수행하는 데 필요한 도구(`Tool`)를 사용할 수 있다. 

`Agent` 는 `LLM` 에이전트, 데이터베이스 쿼리 에이전트, 파일 처리 에이전트 등 다양한 형태로 구현될 수 있으며, 
각 `Agent` 는 자신의 작업을 정의하고, 다른 `Agent` 와 협력하거나 경쟁하여
목표를 달성한다.

`Agent` 의 가장 큰 특징은 스스로 생각하고, 결정하며, 행동하는 `AI` 비서에 가까우 개념이다. 
단순히 텍스트를 생성하는 것이 아니라, 주어진 목표에 맞춰 논리적으로 추론하고 결정을 내릴 수 있다. 
그리고 다양한 상황과 맥락에 맞춰 행동을 조정하며 사용자 요구나 환경 변화에 유연하게 대응한다. 
그러므로 사용자의 개입이 최소화 될 수 있고, 목표 달성을 위해 필요한 작업들을 스스로 계획하고 설행한다.  

예제 진행을 위해 구현할 `Agent` 는 검색 `API` 를 활요앟여 검색 기능을 구현한 도구를 사용한다. 
검색 `API` 는 [DuckDuckGo](https://python.langchain.com/docs/integrations/tools/ddg/)
를 사용해 `Agent` 에서 사용할 [Tool](https://python.langchain.com/docs/concepts/tools/)
을 정의한다.  

그리고 정의된 `Tool` 을 바탕으로 `LangGraph` 의 `Persistent Checkpoint` 기능을 사용하여 
대화의 `Context` 상태를 추적할 수 있도록해 `Multi-Turn` 대화가 가능하도록 개선한다.  

마지막으로는 `LangGraph` 의 출력함수인 `stream()` 함수에 대해 좀 더 자세하게 어떠한 추가 기능을 수행할 수 있는지 알아본다.  
