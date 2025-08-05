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


#### LLMListwiseRerank
`LLMListwiseRerank` 는 `LLM` 을 사용해 검색된 문서들을 리스트 단위로 재정렬하는 방식으로 최적의 순서를 결정한다. 
문서의 전체적인 맥락과 쿼리와의 관련성을 평가해 더 적합한 순서를 제공한다.  

```python
from langchain.retrievers.document_compressors import LLMListwiseRerank

listwise_rerank_compressor = LLMListwiseRerank.from_llm(model, top_n=1)

listwise_rerank_compression_retriever = ContextualCompressionRetriever(
    base_retriever=vector_retriever,
    base_compressor=listwise_rerank_compressor
)

listwise_rerank_results = listwise_rerank_compression_retriever.invoke(query)
# [Document(id='ddc32905-8496-4700-b4dd-570ed8539642', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치')]
```  


#### EmbeddingsFilter
`EmbeddingsFilter` 는 검색된 문서들이 쿼리와 얼마나 유사한지(임베딩 유사도)를 기준으로 특정 문서들을 필터링하는 역할을 수행한다. 
검색된 문서들의 임베딩을 활용하여, 쿼리와 유사도가 높은 문서만 남기고 나머지는 제거함으로써, 
검색 결과의 품질을 높이고 컨텍스트 길이를 제한하는 데 도움을 준다.  

```python
from langchain.retrievers.document_compressors import EmbeddingsFilter

embeddings_filter_compressor = EmbeddingsFilter(
    embeddings=hf_embeddings,
    similarity_threshold=0.5
)

embeddings_filter_compression_retriever = ContextualCompressionRetriever(
    base_retriever=vector_retriever,
    base_compressor=embeddings_filter_compressor
)

embeddings_filter_results = embeddings_filter_compression_retriever.invoke(query)
# [_DocumentWithState(metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치', state={'embedded_doc': [0.027784746140241623, -0.02760959416627884, .. ], 'query_similarity_score': np.float64(0.5899350732822568)}),
#  _DocumentWithState(metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화', state={'embedded_doc': [0.032147958874702454, 0.000486963166622445, ... ], 'query_similarity_score': np.float64(0.5188017648577985)})]
```  


#### DocumentCompressorPipeline
`DocumentCompressorPipline` 은 여러 문서 압축 도구를 체계적으로 연결하여 검색된 문서를 단계적으로 압축하는 파이프라인이다. 
이를 통해 대규모 검색 결과를 효율적으로 처리하고, `LLM` 컨텍스트 제한에 맞는 최적화된 데이터를 반환할 수 있다.  

아래 예시는 `TextSplitter` 를 사용해 문서를 더 작은 조각으로 분할하고, 
`EmbeddingsRedundantFilter` 를 사용해 문서 간 임베딩 유사도를 기분으로 중복 문서를 제거한다. 
그리고 `EmbeddingsFilter` 를 통해 특정 유사도 이상인 문서만 포함시키고, 
최종적으로 `LLMChainExtractor` 를 사용해 문서 내용 중 중요한 정보만 추출하는 파이프라인이다.  

```python
from langchain.retrievers.document_compressors import DocumentCompressorPipeline
from langchain_community.document_transformers import EmbeddingsRedundantFilter
from langchain.retrievers.document_compressors import LLMChainExtractor
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(chunk_size=200, chunk_overlap=0)

redundant_filter = EmbeddingsRedundantFilter(
    embeddings=hf_embeddings
)

relevant_filter = EmbeddingsFilter(
    embeddings=hf_embeddings,
    similarity_threshold=0.5
)

extractor = LLMChainExtractor.from_llm(model)

pipeline_compressor = DocumentCompressorPipeline(
    transformers=[splitter, redundant_filter, relevant_filter, extractor]
)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=pipeline_compressor,
    base_retriever=vector_retriever
)

pipeline_results = compression_retriever.invoke(query)
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전'),
#  Document(metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링')]
```





### MultiVectorRetriever
`MultiVectorRetriever` 는 문서를 여러 벡터로 저장하고 관리하여 검색 정확도와 효율성을 향상시키는 검색기이다. 
문서당 여러 벡터를 생성하는 방법은 아래와 같다. 

- 작은 청크 생성 : 문서를 작은 단위로 나누고 각 청크에 임베딩을 생성하여 세부 정보를 더 잘 탐색
- 요약 임베딩 : 문서의 요약을 기반으로 임베딩을 생성해 핵심 내용을 빠르게 파악
- 가설 질문 활용 : 문서별 가설 질문을 생성하고 이를 기반으로 임베딩을 만들어 다양한 관점에서 접근
- 수동 추가 : 사용자가 직접 질문이나 쿼리를 추가해 맞춤형 검색 구현 

위와 같은 방식을 사용해서 좀 더 풍부한 검색 결과를 제공할 수 있다.
기존의 단일 벡터 스토어를 사용하는 검새긱와 달리, 
여러 벡터 스토어를 병렬로 처리하여 각 스토어에서 검색된 결과를 통합하거나 가중치를 부여해 최종 결과를 반환한다. 


#### Large Chunk & Small Chunk
대용량 문서에서 정보를 검색해야 하는 경우, 큰 청크와 작은 청크를 병렬로 사용하는 방식을 통해 정확도와 포괄성을 동시에 확보할 수 있다. 
큰 청크는 데이터의 더 큰 단위로 주로 문백을 잘 보존하며, 전체적인 흐름과 의미를 이해하는데 적합하다. 
그리고 작은 청크는 데이터의 세부적인 단위로 상세한 정보나 특정 키워드에 민감한 검색을 가능하게 한다. 
이렇게 서로 다른 특성을 가지고 있기 때문에 이를 함께 사용하면, 
큰 청크가 문맥을 제공하고 작은 청크가 세부 정보를 보강하여 더 나은 검색 결과를 기대할 수 있다.  

앞서 알아본 `ParentDocumentRetriever` 와의 차이는 `MultiVectorRetriever` 는 
벡터 스토어에 큰 청크, 작은 청크를 모두 벡터화해 병렬로 검색하고 두 결과를 통합해 최적의 검색 결과를 제공한다. 
반면 `ParentDocumentRetriever` 는 작은 청크만 벡터화해 검색에 사용하고, 
최종 결과는 검색 결과로 나온 작은 청크의 원본(큰 청크)를 반환한다는 차이가 있다.  

이러한 방식은 주로 문서의 계층적 구조가 중요할 떄나 
범률 문서와 같이 전체 볍률 조항과 세부 특정 조항이 모두 필요한 경우 유용하다.  


예제 진행을 위해 `컴퓨터 키워드 문서` 와 `부동산 키워드 문서` 를 사용한다.  

```python
import uuid
from langchain.storage import InMemoryStore
from langchain_chroma import Chroma
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain.retrievers.multi_vector import MultiVectorRetriever

multi_vectorstore = Chroma(
    collection_name="small_bigger_chunks",
    embedding_function=hf_embeddings
)

store = InMemoryStore()

id_key = "doc_id"

multi_vector_retriever = MultiVectorRetriever(
    vectorstore=multi_vectorstore,
    byte_store=store,
    id_key=id_key
)

docs = computerKeywordLoader.load() + propertyKeywordLoader.load()

doc_ids = [str(uuid.uuid4()) for _ in docs]

doc_ids
```


