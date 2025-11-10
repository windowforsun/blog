--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Agent Memory and Stream"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: 'LangGraph 에이전트에서 Human-in-the-Loop(HITL) 구현과 상태 수동 업데이트 방법을 알아보자'
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

## LangGraph Human-in-the-Loop
`LangGraph` 에서 `Human-in-the-Loop`(`HITL`) 는 `LLM` 기반 워크플로우 또는 에이전트 플로우를 설계할 떄, 
자동화된 흐름 중간에 `사람의 개입` 이 필요할 때 이를 쉽게 삽입할 수 있도록 만든 기능이다. 
순전히 `LLM` 이나 자동화된 노드들만으로 해결하기 어려운 경우, 특정 지점에서 사람의 판단, 입력, 승인 등을 기다리고 그 결과를 받아 
흐름을 이어갈 수 있게 해준다.  

주요 개념 및 특징은 다음과 같다. 

- 중간 개입
  - `LLM` 이 답변을 만들어냈지만 민감한 이슈나 정확성이 중요한 단계에서는 사람이 검토/수정하도록 할 수 있다. 
  - 그래프의 노드로 `Human Node` 를 넣으면 해당 시점에 사람이 개입해서 입력을 주거나, 승인을 할 때까지 워크플로우가 일시 정지된다. 
- 비동기 처리
  - 실제로는 사람이 답변을 입력할 떄까지 시스템이 기다릴 수 있어야 하므로, `LangGraph` 에서는 이러한 노드를 비동기적으로 처리할 수 있도록 지원한다. 
  - 슬랙, 이메일, 웹 UI, `CLI` 등 다양한 방식으로 사람에게 요청을 보내고, 답변이 돌아오면 워크플로우가 이어진다. 

    