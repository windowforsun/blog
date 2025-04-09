--- 
layout: single
classes: wide
title: "[LangChain] LangChain RAG Introduction"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain RAG 에 대한 설명과 구현 예시를 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - RAG
    - LLM
    - AI
    - Embedding
    - Vector Store
    - Chroma
    - Nomic
    - Retriever
    - Chunking
toc: true
use_math: true
---  

## RAG
`RAG(Retrieval-Augmented Generation)` 은 `LLM(Large Language Model)` 의 한계를 보완하기 위해 외부 지식 검색(`Retrieval`)과 
텍스트 생성(`Generation`)을 결합한 기법이다. 
이는 주어진 컨텍스트나 질문에 대해 더욱 정확하고 풍부한 정보를 제공하는 방법이다. 
모델이 학습 데이터에 포함되지 않는 외부 데이터를 실시간으로 검색하고, 
이를 바탕으로 답변을 생성하는 과정을 포함한다. 
특히 환각(`hallucination`)을 방지하고, 모델이 최신 정보를 반영하거나 더 넓은 지식을 활용할 수 있게 한다.  

### RAG 주요 구성 요소
- `Retriever`(검색기) : 사용자의 질문에 대해 관련 문서를 검색한다. 벡터 검색, 키워드 검색, BM25 등 다양한 검색 기술을 사용할 수 있다. 여기서 벡터 스토어는 고차원 벡터를 저장하고 유사성 검색을 수행하는 데이터베이스이다. 
- `Generator`(생성기) : 검색된 문서를 바탕으로 응답을 생성한다. 주로 `Transformer` 기반의 언어 모델을 사용한다. `GPT-3`, `T5`, `BERT` 등이 있다.

### RAG 작동 방식
- 질문 입력 : 사용자가 질문을 입력한다. 
- 문서 검색 : 검색기가 잘문과 관련된 문서를 데이터베이스에서 검색한다. 
- 응답 생성 : 생성기가 검색된 문서를 바탕으로 질문에 대한 응답을 생성한다. 

### RAG 장점
- 정확성 향상 : 검색된 문서를 바탕으로 응답을 생성하기 때문에, 단순히 생성 모델만 사용하는 것보다 더 정확한 응답을 생성할 수 있다. 
- 정보 풍부성 : 외부 지식 소스를 활용하여 더 풍부하고 상세한 정보를 제공할 수 있다. 
- 유연성 : 다양한 검색 기술과 생성 모델을 결합하여 사용할 수 있다. 
