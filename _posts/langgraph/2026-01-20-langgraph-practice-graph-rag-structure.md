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

## LangGraph RAG Structure
`LangGraph` 는 복잡한 `AI Workflow` 를 그래프 형태로 구축할 수 있도록 지원하는 프레임워크이다. 
각 워크플로우의 구성 요소(노드)를 명확하게 분리하고, 이들 간의 데이터 흐름(엣지)을 정의함으로써, 유연하고 확장 가능한 `AI Workflow` 를 구축할 수 있다.  

이러한 `LangGraph` 의 특징을 사용해 `RAG (Retrieval-Augmented Generation)` 시스템을 구축하면 복잡하고 다양한/추가적인 기능을 가진 
워크플로우를 좀 더 구조적으로 구현할 수 있다. 

- 모둘화/재사용성 : 각 역할별 노드가 분리되어, 원하는 부분만 독립적으로 수행 및 재사용 가능
- 확장성 : 새로운 기능(추가 검색, 평가 노드 등)을 손쉽게 추가 가능
- 유연성 : 다양한 `RAG` 전략을 조합하여 실험적 워크플로우 구축 가능
- 가독성과 유지보수성 : 그래프 기반 시각화로 시스템 구조 파악 및 유지보수가 쉬움

이렇게 `LangGraph` 를 활용하면 `RAG` 시스템을 단순한 파이프라인이 아니라, 
각 기능 단위를 모듈화하여 유연하게 조합할 수 있다. 
이를 통해 복잡한 요구사항에도 대응하고, 새로운 아이디어나 전략을 빠르게 실험할 수 있는 강력한 `RAG` 플랫폼을 구축할 수 있다.  

본 포스팅에서는 실제 세부 구현은 다루지 않는다. 
`LangGraph` 사용해 `RAG` 를 구축할 때 어떠한 구조와 흐름으로 구성할 수 있는지에 대해 알아보고자 한다.  
