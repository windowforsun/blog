--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph RAG Structure"
header:
  overlay_image: /img/langgraph-img-2.jpeg
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

## Agentic RAG
`Agentic RAG` 는 전통적인 `RAG` 시스템에서 `Agent` 의 역할을 추가하여, 
사용자의 질문에 대해 더 깊이 있는 답변을 제공하는 시스템이다. 
기존 `RAG` 에서는 단순 검색과 생성의 파이프라인이 중심이었다면, 
`Agentic RAG` 는 복수의 에이전트들이 협력하며, 다양한 살태와 흐름 제어를 통해 더 복잡한 정보 검색 및 생성 작업을 수생할 수 있다. 

`LangGraph` 기반의 `Agentic RAG` 의 주요 특징은 아래와 같다. 

- `Agentic` 구조 : 단일 `LLM` 에 의존하지 않고, 여러 `Agent`(도구, 함수, `LLM` 등)가 각자 역할을 나누어 협력한다. 각 `Agent` 가 자신의 상태와 맥락에 따라 행동을 결정한다. 
- 동적 워크플로우 : 질의의 난이도, 유형, 중간 결과에 따라 워크플로우가 동적으로 변화한다. 예를 들어 검색 결과가 부족할 때 추가 검색, 요약, 재질문 등 자동 분기가 가능하다. 
- 상태 기반 처리 : 각 단계의 결과와 상태를 기록, 필요 시 이전 단계로 되돌리거나, 추가 작업을 수행할 수 있다. 이를 통해 복잡한 질의에도 유연하게 대응할 수 있다. 
- 모듈화와 확장성 : 각 노드를 별도의 함수/모듈로 분리하여 재사용 및 확장이 용이하다. 새로운 `Agent` 도 쉽게 추가할 수 있다.  

이렇게 `LangGraph` 를 활용한 `Agentic RAG` 는 기존 `RAG` 의 한계르 넘어, 
복잡한 질의와 다양한 처리 흐름을 에이전트와 그래프 기반 상태 관리로 더 유연하고 강력하게 구현할 수 있다. 
이 구조는 확장성과 유지보수가 뛰어나며, 다양한 도메인에 맞춘 맞춤형 `RAG` 시스템 개발에 매우 적합하다.  

본 포스팅에서 진행할 예제는 [이전 예제]({{site.baseurl}}{% link _posts/2026-02-01-langgraph-practice-naive-rag-relevant.md %})
의 내용을 포함하고 하고 있으므로 이전 내용의 숙지가 필요하다.  
