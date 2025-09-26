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

테스트셋 생성에 사용할 `LLM` 모델과 `Embedding` 모델을 초기화한다.  

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from langchain_huggingface.embeddings import HuggingFaceEndpointEmbeddings
from langchain_huggingface.embeddings import HuggingFaceEmbeddings
import os

os.environ["GOOGLE_API_KEY"] = "api key"
generator_llm = LangchainLLMWrapper(ChatGoogleGenerativeAI(model="gemini-2.0-flash"))

os.environ["HUGGINGFACEHUB_API_TOKEN"] = "api key"
model_name = "BM-K/KoSimCSE-roberta"
hf_endpoint_embeddings = HuggingFaceEndpointEmbeddings(
    model=model_name,
    huggingfacehub_api_token=os.environ["HUGGINGFACEHUB_API_TOKEN"],
)
hf_embeddings = HuggingFaceEmbeddings(
    model_name=model_name,
    encode_kwargs={'normalize_embeddings':True},
)
generator_embeddings = LangchainEmbeddingsWrapper(hf_embeddings)
```  

불러온 문서를 `Knowledge Graph` 를 생성해 추가한다.  

```python
from ragas.testset.graph import KnowledgeGraph
from ragas.testset.graph import Node, NodeType

kg = KnowledgeGraph()

for doc in split_docs:
  kg.nodes.append(
      Node(
          type=NodeType.DOCUMENT,
          properties={
              "page_content" : doc.page_content,
              "document_metadata" : doc.metadata
          }
      )
  )

print(kg)
# KnowledgeGraph(nodes: 17, relationships: 0)


```  

노드로만 구성한 지식 그래프를 사용해 `SingleHop` 데이터셋 생성을 위해 그래프를 개선하고 쿼리 생성을 개선하기 위해 `KeyphrasesExtractor` 를 사용한다. 
필요한 경우 `Ragas` 에서 제공하는 다양한 변환을 적용할 수 있다. 
`KeyphrasesExtractor` 를 사용하면 생성된 쿼리의 다양성과 관련성을 풍부하게 하는 의미론적 시드 포인트 역할을 하는 핵심 주제 키구문을 식별할 수 있다.  

```python
from ragas.testset.transforms import apply_transforms
from ragas.testset.transforms import KeyphrasesExtractor

keyphrase_extractor = KeyphrasesExtractor(llm=generator_llm)

transforms = [
    keyphrase_extractor
]

apply_transforms(kg, transforms=transforms)
```  

그리고 쿼리 생성을 위한 `Persona` 를 구성한다. 
페르소나는 생성된 쿼리가 자연스럽고 사용자에 따라 다르며 다양한 맥락과 관점을 제공한다. 
쿼리를 다양한 사요앚 관점에 맞게 조정함으로써, 테스트 세트는 다양한 시나리오와 사용자 요구를 포괄할 수 있다. 
현재 예제 문서는 기업관련 `AI` 최신 기술 소개를 다루기 때문에 각 직무에 따른 시나리오를 다룬다.  

```python
from ragas.testset.persona import Persona

persona_general_public = Persona(
    name="general public",
    role_description="A person with little to no background in technology or artificial intelligence. They are interested in understanding how AI impacts their daily life and society, focusing on clear, jargon-free explanations and practical implications."
)

persona_software_developer = Persona(
    name="software developer",
    role_description="A professional with experience in programming and software engineering, but not necessarily specialized in AI. They are interested in how AI technologies work under the hood, potential integration points, and best practices for implementation."
)

persona_ai_expert = Persona(
    name="ai expert",
    role_description="An individual with a deep understanding of artificial intelligence, including theory, algorithms, and practical applications. They seek detailed technical information, research references, and advanced discussions on model architectures and performance."
)

personas = [persona_general_public, persona_software_developer, persona_ai_expert]
```  
