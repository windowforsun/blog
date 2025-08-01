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

## Retriever
[LangChain Retriever 1st]({{site.baseurl}}{% link _posts/langchain/2025-07-20-langchain-practice-langchain-retriever-1.md %})
에 이어서 `LangChain` 을 바탕으로 사용할 수 있는 `Retriever` 의 종류와 그 특징에 대해서 좀 더 알아본다.  


### ContextualCompressionRetriever
`ContextualCompressionRetriever` 는 검색 과정에서 질의 관련된 정보를 추출하여 효율성을 높이는 검색기이다. 
데이터 수집 시 특정 질의를 미리 예측할 수 없어, 관려 정보가 방대한 텍스트에 묻혀 잇을 수 있다. 
이를 그대로 전달하면 높은 `LLM` 호출 비용과 낮은 응답 품질로 이어질 수 있다. 
이 문제를 해결하기 위해 `ContextualCompressionRetriever` 를 사용하면, 
질의 기반으로 문서를 압축해 관련 정보만 반환한다. 
이는 개별 문서의 내용을 줄이거나 필요 없는 문서를 제거하는 과정을 포함한다.  

동작은 질의를 `base retriever` 에 전달해 초기 문서를 가져온 뒤, 
`Document Compression` 을 통해 문서 내용을 축소하거나 목록에서 제거하는 방식을 사용한다. 
이를 통해 보다 효율적이고 정확한 검색 결과를 제공할 수 있다.  

이러한 특징으로 컨텍스트 길이 제한 문제를 해결할 수 있고, 
중요도 기반 데이터 필터링도 가능하다.  

벡터 방식 검색기를 사용하는 경우 질의에 대해 맞지 않는 검색 결과가 포함돼 있는 것을 확인할 수 있다. 

```python
vector_results = vector_retriever.invoke(query)
# [Document(id='ddc32905-8496-4700-b4dd-570ed8539642', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(id='968f2bec-f220-4bf4-bffc-04da45cfa383', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='59497918-6a17-4cb8-bf70-86436f92a40a', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'),
#  Document(id='aa16f442-43e2-403e-adf9-a4f1bc3fd950', metadata={'source': './computer-keywords.txt'}, page_content='GPU\n\n정의: GPU(Graphics Processing Unit)는 컴퓨터의 그래픽 렌더링과 복잡한 병렬 처리를 전문적으로 수행하는 프로세서입니다.\n예시: NVIDIA GeForce RTX 3080은 게임 및 인공지능 학습에 활용되는 고성능 GPU입니다.\n연관키워드: 그래픽 카드, 렌더링, CUDA, 병렬 처리\n\nSSD')]
```  

#### LLMChainExtractor
`LLMChainExtractor` 는 `LLM` 을 사용해 검색 문서 중 중요한 정보를 추출할 수 있다. 
쿼리와 관련된 핵심 내용만 추출해 컨텍스트 길이 제한이나 불필요한 정보를 `LLM` 에게 전달하지 않을 수 있다. 
문서의 실제 내용도 압축이 된다는 점을 기억해야 한다. 

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor


extractor_compressor = LLMChainExtractor.from_llm(model)
extractor_compression_retriever = ContextualCompressionRetriever(
    base_retriever=vector_retriever,
    base_compressor=extractor_compressor
)

extractor_results = extractor_compression_retriever.invoke(query)
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전'),
#  Document(metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링')]
```  


#### LLMChainFilter
`LLMChainFilter` 는 `LLM` 을 사용해 검색된 문서들 중에서 쿼리와 관련성 높은 문서만 필터링 하는 역할을 수행한다. 
쿼리와 관련성이 부족한 문서를 제거하여 보다 정밀한 검색 결과를 제공하는데 초점을 맞춘다. 
이는 문서의 실제 내용은 그대로 두고 문서 중 관련성 높은 문서만 선택적으로 반환한다.  

```python
from langchain.retrievers.document_compressors import LLMChainFilter

filter_compressor = LLMChainFilter.from_llm(model)
filter_compression_retriever = ContextualCompressionRetriever(
    base_retriever=vector_retriever,
    base_compressor=filter_compressor
)


filter_results = filter_compression_retriever.invoke(query)
# [Document(id='ddc32905-8496-4700-b4dd-570ed8539642', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(id='968f2bec-f220-4bf4-bffc-04da45cfa383', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화')]
```  

