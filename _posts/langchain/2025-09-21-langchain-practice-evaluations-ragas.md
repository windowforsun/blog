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

## Ragas
`Ragas` 는 `RAG` 시스템의 품질을 평하가고 검증하기 위한 오픈소스 라이브러리이다. 
`RAG` 파이프라인은 검색기와 생성기가 결합된 구조로, `Ragas` 는 이 파이프라인의 각 단계가 얼마나 잘 동작하는지를 체계적으로 측정하는 역할을 한다. 
`RAG` 파이프라인의 정확성, 일관성, 출처 신뢰성, 정보 회수력 등 여러 품질 지표를 자동으로 평가하고, 
수동 평가 없이도 다양한 메트릭을 제공한다. 
그리고 `LangChain`, `LlamaIndex` 등 다양한 프레임워크에서 손쉽게 연동할 수 있다. 

이번 포스팅에서는 `LangChain` 과 `Ragas` 를 사용해 테스트 데이터 셋을 생성하고 평가는 방법에 대해 알아본다.  

> 본 포스팅에서 사용한 `Ragas` 버전은 `0.2.15` 이다.


### Synthetic Test Dataset
`Synthetic Test Dataset` 은 실제 사용자 데이터가 부족하거나, 
특정 상황/도메인에 맞는 평가 데이터가 필요할 떄, 
`LLM` 등 자동화된 방법을 이용해 인위적으로 생성한 평가용 데이터셋을 의미한다. 
보통 `RAG` 시스템의 질적 평가를 위해서는 `query`, `context`, `answer` 이 세트로 구성된 데이터셋이 필요한데, 
이걸 사람이 직접 만드는 것은 시간과 비용이 많이 든다. 
`Ragas` 는 이런 데이터셋을 자동으로 생성하는 기능을 제공한다.  

먼저 `RAG` 에 사용할 문서를 로드하고 일부분만 사용한다.  

```python
from langchain_community.document_loaders import PyPDFLoader

pdf_loader = PyPDFLoader("./SPRi AI Brief 5월호 산업동향.pdf")
pdf_docs = pdf_loader.load()

pdf_docs_mini = pdf_docs[10:17]

for doc in pdf_docs_mini:
  doc.metadata['filename'] = doc.metadata['source']

print(len(pdf_docs_mini))
# 7

splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)

split_docs = splitter.split_documents(pdf_docs_mini)
print(len(split_docs))
# 17
```  
