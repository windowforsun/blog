--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Intro"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LLM 기반의 복잡한 워크플로우와 멀티에이전트 협업을 그래프 구조로 설계하고 구현할 수 있는 AI Framework 인 LangGraph 에 대해 알아보자
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

