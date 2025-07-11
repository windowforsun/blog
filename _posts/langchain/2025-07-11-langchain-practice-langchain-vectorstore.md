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

## VectorStore
`VectorStore` 는 자연어 처리(`NLP`) 에서 텍스트 데이터를 임베딩(`Embedding`) 벡터로 변환하여 저장하고, 
유사한 텍스트를 빠르게 검색할 수 있게 해주는 컴포넌트이다. 
텍스트(문장, 문서 등)을 벡터(숫자 배열)로 변환해서 저장하는 데이터베이스이다. 
벡터는 텍스트의 의미를 수치적으로 표현한 것으로, 임베딩 모듈(`OpenAI`, `HuggingFace` 등)을 통해 생성한다. 
저장된 벡터들 중에서, 새로운 쿼리와 가장 유사한 벡터(텍스트)를 빠르게 찾을 수 있다. 

`VectorStore` 의 주요 역할은 아래와 같다. 

1. 임베딩 생성 : 텍스트 -> 임베딩 모델 -> 벡터(숫자 배열)
2. 벡터 저장 : 생성된 벡터를 `VectorStore` 에 저장
3. 유사도 검색 : 쿼리 텍스트를 임베딩한 후, 저장된 벡터들과 유사도를 계산하여 가장 비슷한 텍스트(문서)를 반환

`VectorStore` 는 대표적으로 `RAG(Retrieval-Augmented Generation)` 에서 사용할 수 있다. 
이는 `LLM` 이 답변을 생성할 떄, `VectorStore` 에서 관련 문서를 찾아 참고자료로 활용하는 것을 의미한다. 
그리고 대용량 문서에서 의미적으로 유사한 문서/문장을 검색하는데도 활용할 수 있고, 
이를 바탕으로 챗봇에서 사용자의 질문과 유사한 `FAQ`, 메뉴얼, 대화 이력을 검색하는 데도 활용할 수 있다.  

`VectorStore` 의 종류에는 대표적으로 아래와 같은 것들이 있다. 

- `FAISS` : Facebook AI Research에서 개발한 벡터 검색 라이브러리로, 대규모 데이터셋에서 빠른 검색을 지원한다.
- `Chroma` : 오픈 소스 벡터 데이터베이스로, 파이썬 기반의 경략 벡터 DB 이다. 
- `Pinecone`, `Weaviate`, `Milvus` : 클라우드 기반의 벡터 데이터베이스로, 대규모 데이터셋에서 빠른 검색을 지원한다. 

