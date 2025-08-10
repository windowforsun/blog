--- 
layout: single
classes: wide
title: "[LangChain] LangChain Prompt"
header:
  overlay_image: /img/langchain-bg-2.jpg
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

## Reranker
`Reranker` 는 정보 검색이나 생성형 인공지능 파이프라인에서 여러 후보 결과(`Retriever`)를 다시 평가해서 최적의 순서로 재정렬하는 것을 의미한다. 
`Retriever` 의 검색 결과로 질의에 맞는 여러 후보가 도출됐을 때 이를 가장 관련성이 높거나 품질이 좋은 결과가 맨 위로 오도록 순서를 다시 매기는 역할을 한다.  

초기 검색은 비슷한 것을 잘 찾는 특징이 있지만, 
정확하게 가장 좋은 것을 찾는 것에는 부족한 부분이 있다. 
이를 보완하기 위해 1차 검색 결과를 다시 평가해서 정말로 가장 적합한 결과가 맨위로 오도록 보정하는 것이 리랭커의 역할이다.  

`Reranker` 는 주로 `RAG` 파이프라인에서 사용되며, 앞서 언급한 것처럼 파이프라인상 `Retriever` 의 뒤에 위치해 후보 문서를 좀 더 정교하게 분석해 최종적인 순위를 결정한다. 
동작 원리는 아래와 같다. 

- `Retriever` 를 통해 초기 후보 문서를 입력으로 받는다. 
- 질의와 각 후보를 함께 모델에 넣어 관련성 점수를 산출한다. 
- 점수가 높은 순서로 재정렬 한다. 

장점으로는 의미적 벡터 검색 결과를 보완해 정확도를 크게 향상 시킬 수 있다는 점과 복잡한 의미적 관계 모델링이 가능하다는 점이 있다. 
단점으로는 계산 비용과 처리시간이 증가하고 대규모 데이터셋에 직접 적용이 어렵다는 점이 있다.  

`Reranker` 는 일반적으로 `Cross-Encoder` 방식을 사용한다. 
이는 `Retriever` 에서 사용하는 `Bi-Encoder` 방식인 문서와 쿼리를 각각 인코딩해 벡터를 생성하고, 
벡터 간 코사인 유사도 등을 통해 관련 문서를 빠르게 찾는 것과는 차이가 있다. 
`Cross-Encoder` 는 쿼리와 문서를 하나의 입력으로 결합해 교차 인코딩해 두 입력 간의 사오작용을 정밀하게 평가할 수 있는 구조이다. 
이는 문서 마다 연산이 필요하기 때문에 계산량은 많지만 더 정확한 관련성 평가가 가능하다. 
`Cross-Encoder` 의 입력은 아래와 같은 형식으로 구성된다. 

```
[CLS] Query [SEP] Document [SEP]

[CLS]: 문장의 시작을 나타내는 특별한 토큰
[SEP]: 쿼리와 문서 간을 구분하는 역할을 하는 토큰
```    
