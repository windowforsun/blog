--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Naive RAG"
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

## LangGraph Naive RAG
`Naive RAG` 는 `Retrieval-Augmented Generation` 의 가장 기본적인 형태이다. 
`RAG` 는 자연어 처리 분야에서 정보를 생성할 때 외부 지식 소스를 검색(`retrieval`)하여 생성(`generation`) 모델에 결합하는 방법론이다. 
`naive` 란 용어는 복잡한 최적화나 고도화된 검색, 생성, 젼략 없이 가장 단순한 방식으로 검색과 생성을 결합한 구조이기 때문이다. 

데모 예제는 챗봇이 사용자의 질문에 답하기 위해 PDF 문서에서 정보를 검색하고, 이를 바탕으로 답변을 생성하는 구조로 되어 있다.  

