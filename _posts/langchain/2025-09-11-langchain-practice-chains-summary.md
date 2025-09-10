--- 
layout: single
classes: wide
title: "[LangChain] LangChain Prompt"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain 에서 Prompt 를 사용해 언어 모델에 대한 입력을 구조화하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
toc: true
use_math: true
---  

## Chains for Summary
`LLM` 에서 문서를 요약하는 것은 자연어 처리(NLP)에서 중요한 작업 중 하나이다. 
이번에는 `LangChain` 에서 `Chains` 를 사용하여 문서를 요약하는 방법에 대해 알아본다. 
`LangChain` 의 문서 요약 방법에는 대표적으로 아래와 같은 것들이 있다. 

- `Stuff` : 모든 문서를 한번에 `LLM` 프롬프트에 넣어 요약한다. 
- `Map-Reduce` : 문서를 여러 청크로 나누고, 각 청크를 개별적으로 요약(`Map`)한 뒤, 이 요약본들을 다시 하나로 통합 요약(`Reduce`)한다. 
- `Map-Refine` : 문서를 청크로 나눈 뒤, 첫 번째 청크를 요약하고, 이후 각 청크마다 이전 요약본을 바탕으로 점진적으로 내용을 보완(`Refine`)한다.
- `Chain of Density` : 여러 번 반복적으로 요약을 수행하면서, 매 단계마다 누락된 핵심 엔티티(정보)를 추가로 반영하여 점점 더 졍보 밀도가 높은 요약을 만들어낸다. 
- `Clustering-Map-Refine` : 문서 청크를 의미적으로 `N`개 클러스터로 그룹화한 뒤, 각 클러스터의 중심 청크를 중심으로 `Refine`(점진적 요약) 방식을 적용한다. 

