--- 
layout: single
classes: wide
title: "[LangChain]LangChain Retriever 2nd"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: 'LangChain 에서 Retriever 의 역할과 종류에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
    - Retriever
    - RAG
    - Vector Store
    - Chroma
    - ContextualCompressionRetriever
    - LLMChainExtractor
    - LLMChainFilter
    - LLMListwiseRerank
    - EmbeddingsFilter
    - DocumentCompressorPipeline
    - MultiVectorRetriever
    - SelfQueryRetriever
    - TimeWeightedVectorStoreRetriever
    - LongContextReorder
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



### SelfQueryRetriever
`SelfQueryRetriever` 는 사용자의 자연어 질문을 자동으로 해석하여 구조화된 쿼리와 메타데이터 필터로 변환한다. 
이를 통해 더 정확하고 관련성 높은 문서를 검색할 수 있다. 
자연어 질문에서 메타데이터 필터링 조건과 의미 검색 쿼리를 자동으로 추출하여 정확한 검색 결과를 제공하는 검색기이다.  

동작 방식은 아래와 같다. 

1. 질문 분석 : 사용자의 자연어 질문을 받아 `LLM` 을 사용해 분석
2. 필터 추출 : 질문에서 메타데이터 필터링 조건을 식별
3. 쿼리 변환 : 필터링 조건을 제외한 실제 검색 의도를 추출
4. 병합 실행 : 의미 검색과 메타데이터 피렅링을 함께 적용하여 결과 반환

주요한 특징을 정리하면 다음과 같다. 

- 메타데이터 기반 필터링 : 메타데이터의 필드를 다양한 조건으로 검색 가능하다. 숫자 범위, 문자열 일치, 불리언 조건 등 가능하다. 
- 자연어 이해 : 자연어인 `최근 2년 이내` 를 `year >= 2023` 와 같이 자동으로 메타데이터 조건으로 변환할 수 있다. 
- 결합된 검색 : 의미 기반 검색과 메타데이터 필터링을 단일 쿼리로 통합할 수 있다. 

지원하는 필터 연산자를 정리하면 다음과 같다. (이는 벡터 저장소 구현에 따라 지원이 달라질 수 있다.)

- `eq` : 같음
- `ne` 같지 않음
- `lt/gt` 작음/큼
- `lte/gte` 작거나 같음/크거나 같음
- `in` : 리스트에 포함됨
- `nin` : 리스트에 포함되지 않음
- `and/or/not` : 논리 연산자 

예제는 컴퓨터 관련 제품에 대한 문서에 출시년도, 카테고리 등 메타정보를 추가해서 `Chroma` 벡터 저장소에 저장해 진행한다.  

```python
from langchain_chroma import Chroma
from langchain_core.documents import Document

docs = [
    Document(
        page_content="최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다.",
        metadata={"year": 2024, "category": "hardware", "user_rating": 4.7},
    ),
    Document(
        page_content="Adobe Photoshop 2025 버전, AI 기능이 강화되어 이미지 편집이 더욱 직관적이고 효율적으로 개선되었습니다.",
        metadata={"year": 2025, "category": "software", "user_rating": 4.5},
    ),
    Document(
        page_content="Wi-Fi 7 기술 지원 무선 공유기, 더 넓은 대역폭과 낮은 지연 시간으로 안정적인 네트워크 환경을 제공합니다.",
        metadata={"year": 2023, "category": "netwoking", "user_rating": 4.8},
    ),
    Document(
        page_content="엔비디아 RTX 4090 그래픽카드, 8K 게이밍과 딥러닝 작업에 최적화된 성능을 제공합니다.",
        metadata={"year": 2023, "category": "hardware", "user_rating": 4.6},
    ),
    Document(
        page_content="올인원 안티바이러스 솔루션, 실시간 모니터링과 AI 기반 위협 감지로 시스템을 안전하게 보호합니다.",
        metadata={"year": 2024, "category": "security", "user_rating": 4.4},
    ),
    Document(
        page_content="32인치 4K HDR 모니터, 넓은 색 영역과 높은 주사율로 생생한 디스플레이 환경을 제공합니다.",
        metadata={"year": 2024, "category": "peripherals", "user_rating": 4.9},
    ),
    Document(
        page_content="클라우드 기반 백업 솔루션, 자동 동기화와 버전 관리 기능으로 중요한 데이터를 안전하게 보관합니다.",
        metadata={"year": 2023, "category": "cloud", "user_rating": 4.5},
    ),
    Document(
        page_content="기계 학습 개발 키트, 코딩 경험이 적은 사용자도 쉽게 AI 모델을 훈련하고 배포할 수 있습니다.",
        metadata={"year": 2025, "category": "ai", "user_rating": 4.7},
    ),
    Document(
        page_content="인체공학적 무선 키보드, 손목 피로를 줄이고 장시간 타이핑에도 편안함을 제공합니다.",
        metadata={"year": 2024, "category": "peripherals", "user_rating": 4.3},
    ),
    Document(
        page_content="컨테이너 기반 가상화 플랫폼, 애플리케이션 개발과 배포를 간소화하여 DevOps 환경을 최적화합니다.",
        metadata={"year": 2023, "category": "software", "user_rating": 4.8},
    ),
]

self_query_store = Chroma.from_documents(
    docs, hf_embeddings, collection_name="self_query_retriever"
)

```  

`SelfQueryRetriever` 는 문서가 지원하는 메타데이터 필드와 문서 내용에 대한 간단한 설명을 제공해야 한다. 
이는 `AttributeInfo` 를 사용해 제공할 수 있다. 

```python
from langchain.chains.query_constructor.base import AttributeInfo
from langchain.retrievers.self_query.base import SelfQueryRetriever


metadata_field_info = [
    AttributeInfo(
        name="category",
        description="The category of the cosmetic product. One of ['hardware', 'software', 'networking', 'security', 'peripherals', 'cloud', 'ai']",
        type="string",
    ),
    AttributeInfo(
        name="year",
        description="The year the cosmetic product was released",
        type="integer",
    ),
    AttributeInfo(
        name="user_rating",
        description="A user rating for the cosmetic product, ranging from 1 to 5",
        type="float",
    ),
]
```  

위에 작성한 메타데이터 필드에 대한 정의를 사용해 `SelfQueryRetriever` 를 생성한다. 
검색기가 실제로 사용하는 쿼리를 확인하기 위해 `verbose=True` 로 설정하고 로깅 레벨 설정도 추가한다.  

```python
from langchain.retrievers.self_query.base import SelfQueryRetriever
import logging

self_query_retriever = SelfQueryRetriever.from_llm(
    llm=model,
    vectorstore=self_query_store,
    document_contents="컴퓨터 관련 제품 설명",
    metadata_field_info=metadata_field_info,
    verbose=True
)

logging.getLogger("langchain.retrievers.self_query").setLevel(logging.INFO)
```  

메타데이터 필터링과 관련된 쿼리를 테스트해 보면 아래와 같다.  

```python
self_query_result = self_query_retriever.invoke('별점이 4.5')
# INFO:langchain.retrievers.self_query.base:Generated Query: query=' ' filter=Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='user_rating', value=4.5) limit=None
# [Document(id='65563f51-143d-4bfa-b1e8-4d67ced6e99e', metadata={'category': 'software', 'user_rating': 4.5, 'year': 2025}, page_content='Adobe Photoshop 2025 버전, AI 기능이 강화되어 이미지 편집이 더욱 직관적이고 효율적으로 개선되었습니다.'),
#  Document(id='e06b8d92-e805-445b-a395-6716063aa2f0', metadata={'category': 'cloud', 'user_rating': 4.5, 'year': 2023}, page_content='클라우드 기반 백업 솔루션, 자동 동기화와 버전 관리 기능으로 중요한 데이터를 안전하게 보관합니다.')]

self_query_result = self_query_retriever.invoke('별점이 4.5 이상인 것들')
# INFO:langchain.retrievers.self_query.base:Generated Query: query=' ' filter=Comparison(comparator=<Comparator.GTE: 'gte'>, attribute='user_rating', value=4.5) limit=None
# [Document(id='7539eba0-8de4-45fd-91c2-8de33f978921', metadata={'category': 'peripherals', 'user_rating': 4.9, 'year': 2024}, page_content='32인치 4K HDR 모니터, 넓은 색 영역과 높은 주사율로 생생한 디스플레이 환경을 제공합니다.'),
#  Document(id='2a0df170-2234-4cfd-a592-996c1af46c6a', metadata={'category': 'netwoking', 'user_rating': 4.8, 'year': 2023}, page_content='Wi-Fi 7 기술 지원 무선 공유기, 더 넓은 대역폭과 낮은 지연 시간으로 안정적인 네트워크 환경을 제공합니다.'),
#  Document(id='85dc50d9-e8f1-40a9-97b0-d6b090654d52', metadata={'category': 'hardware', 'user_rating': 4.6, 'year': 2023}, page_content='엔비디아 RTX 4090 그래픽카드, 8K 게이밍과 딥러닝 작업에 최적화된 성능을 제공합니다.'),
#  Document(id='d543fd69-2232-4f37-8de6-37fb0cfa9290', metadata={'category': 'hardware', 'user_rating': 4.7, 'year': 2024}, page_content='최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다.')]

self_query_result = self_query_retriever.invoke('별점이 4.5 또는 4.9인 것들')
# INFO:langchain.retrievers.self_query.base:Generated Query: query=' ' filter=Operation(operator=<Operator.OR: 'or'>, arguments=[Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='user_rating', value=4.5), Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='user_rating', value=4.9)]) limit=None
# [Document(id='7539eba0-8de4-45fd-91c2-8de33f978921', metadata={'category': 'peripherals', 'user_rating': 4.9, 'year': 2024}, page_content='32인치 4K HDR 모니터, 넓은 색 영역과 높은 주사율로 생생한 디스플레이 환경을 제공합니다.'),
#  Document(id='65563f51-143d-4bfa-b1e8-4d67ced6e99e', metadata={'category': 'software', 'user_rating': 4.5, 'year': 2025}, page_content='Adobe Photoshop 2025 버전, AI 기능이 강화되어 이미지 편집이 더욱 직관적이고 효율적으로 개선되었습니다.'),
#  Document(id='e06b8d92-e805-445b-a395-6716063aa2f0', metadata={'category': 'cloud', 'user_rating': 4.5, 'year': 2023}, page_content='클라우드 기반 백업 솔루션, 자동 동기화와 버전 관리 기능으로 중요한 데이터를 안전하게 보관합니다.')]

self_query_result = self_query_retriever.invoke('카테고리가 하드웨어 인것')
# INFO:langchain.retrievers.self_query.base:Generated Query: query=' ' filter=Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='category', value='hardware') limit=None
# [Document(id='85dc50d9-e8f1-40a9-97b0-d6b090654d52', metadata={'category': 'hardware', 'user_rating': 4.6, 'year': 2023}, page_content='엔비디아 RTX 4090 그래픽카드, 8K 게이밍과 딥러닝 작업에 최적화된 성능을 제공합니다.'),
#  Document(id='d543fd69-2232-4f37-8de6-37fb0cfa9290', metadata={'category': 'hardware', 'user_rating': 4.7, 'year': 2024}, page_content='최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다.')]

self_query_result = self_query_retriever.invoke('카테고리가 소프트웨어 인것')
# INFO:langchain.retrievers.self_query.base:Generated Query: query=' ' filter=Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='category', value='software') limit=None
# [Document(id='e706c8a0-27a1-4f4e-84c0-43f4b633296c', metadata={'category': 'software', 'user_rating': 4.8, 'year': 2023}, page_content='컨테이너 기반 가상화 플랫폼, 애플리케이션 개발과 배포를 간소화하여 DevOps 환경을 최적화합니다.'),
#  Document(id='65563f51-143d-4bfa-b1e8-4d67ced6e99e', metadata={'category': 'software', 'user_rating': 4.5, 'year': 2025}, page_content='Adobe Photoshop 2025 버전, AI 기능이 강화되어 이미지 편집이 더욱 직관적이고 효율적으로 개선되었습니다.')]

self_query_result = self_query_retriever.invoke('카테고리가 소프트웨어 이면서 별점이 4.7 이상')
# INFO:langchain.retrievers.self_query.base:Generated Query: query=' ' filter=Operation(operator=<Operator.AND: 'and'>, arguments=[Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='category', value='software'), Comparison(comparator=<Comparator.GTE: 'gte'>, attribute='user_rating', value=4.7)]) limit=None
# [Document(id='e706c8a0-27a1-4f4e-84c0-43f4b633296c', metadata={'category': 'software', 'user_rating': 4.8, 'year': 2023}, page_content='컨테이너 기반 가상화 플랫폼, 애플리케이션 개발과 배포를 간소화하여 DevOps 환경을 최적화합니다.')]
```  

좀 더 복합하게 자연어와 메타데이터 필터링을 결합한 질의를 수행하면 아래와 같다. 

```python
self_query_result = self_query_retriever.invoke('사람과 같이 학습하고 추론할 수 있는 것을 만드는데 필요한 제품 중 평점이 4.5 이상인 것에 대해 알려줘')
# INFO:langchain.retrievers.self_query.base:Generated Query: query='인공지능' filter=Operation(operator=<Operator.AND: 'and'>, arguments=[Comparison(comparator=<Comparator.GTE: 'gte'>, attribute='user_rating', value=4.5), Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='category', value='ai')]) limit=None
# [Document(id='fdc258be-ec1e-42d8-9b6b-373467c23b72', metadata={'category': 'ai', 'user_rating': 4.7, 'year': 2025}, page_content='기계 학습 개발 키트, 코딩 경험이 적은 사용자도 쉽게 AI 모델을 훈련하고 배포할 수 있습니다.')]

self_query_result = self_query_retriever.invoke('최근 3년 이내 제품 중 해킹 방어에 도움이 되는 제품을 알려줘')
# INFO:langchain.retrievers.self_query.base:Generated Query: query='해킹 방어' filter=Operation(operator=<Operator.AND: 'and'>, arguments=[Comparison(comparator=<Comparator.GTE: 'gte'>, attribute='year', value=2021), Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='category', value='security')]) limit=None
# [Document(id='9649cd6d-6013-483f-8f40-04a8d7ee9e20', metadata={'category': 'security', 'user_rating': 4.4, 'year': 2024}, page_content='올인원 안티바이러스 솔루션, 실시간 모니터링과 AI 기반 위협 감지로 시스템을 안전하게 보호합니다.')]

self_query_result = self_query_retriever.invoke('서버를 구성할 때 필요한 제품중 2023년에 출시한 제품을 알려줘')
# INFO:langchain.retrievers.self_query.base:Generated Query: query='서버 구성' filter=Comparison(comparator=<Comparator.EQ: 'eq'>, attribute='year', value=2023) limit=None
# [Document(id='eca8d856-5c98-4f8a-a281-c8b4528f63c1', metadata={'category': 'software', 'user_rating': 4.8, 'year': 2023}, page_content='컨테이너 기반 가상화 플랫폼, 애플리케이션 개발과 배포를 간소화하여 DevOps 환경을 최적화합니다.'),
#  Document(id='4647e795-b8f9-49fd-96b4-f0896e8af69c', metadata={'category': 'netwoking', 'user_rating': 4.8, 'year': 2023}, page_content='Wi-Fi 7 기술 지원 무선 공유기, 더 넓은 대역폭과 낮은 지연 시간으로 안정적인 네트워크 환경을 제공합니다.'),
#  Document(id='66e8b875-629f-4452-a1ef-a13a7f8e6620', metadata={'category': 'cloud', 'user_rating': 4.5, 'year': 2023}, page_content='클라우드 기반 백업 솔루션, 자동 동기화와 버전 관리 기능으로 중요한 데이터를 안전하게 보관합니다.'),
#  Document(id='a6b14a4d-9ce1-485b-b17c-e0915c3eaa14', metadata={'category': 'hardware', 'user_rating': 4.6, 'year': 2023}, page_content='엔비디아 RTX 4090 그래픽카드, 8K 게이밍과 딥러닝 작업에 최적화된 성능을 제공합니다.')]
```


### TimeWeightedVectorStoreRetriever
`TimeWeightedVectorStoreRetriever` 는 의미론적 유사성과 시간에 따른 감쇠를 결합해 검색을 수행하는 검색기이다. 
즉 질의에 대해 문서의 관련성 뿐만아니라, 시간적 신선도(`rencency`)까지 고려하여 검색 결과를 제공한다. 
일반 벡터 검색이 유사도만 거려하는 것과 달리, 
`언제 문서가 추가되었는지/언제 최근 사용됐는지` 를 검색 알고리즘에 반영한다.  

작동 원리는 아래와 같다. 

1. 벡터 유사도 계산 : 일반적인 벡터 검색과 마찬가지로 의미적 유사도를 계산한다. 
2. 시감 감쇠 적용 : 각 문서의 나이에 기반하여 감쇠 계수를 적용한다. 
3. 최종 점수 계산 : 유사도와 시간 가중치를 결합하여 최종 관련성 점수를 산출한다. 
4. 결과 반환 : 최정 점수에 따라 정렬된 문서들을 반환한다. 

여기서 시간 감쇠에 대한 스코어링 알고리즘은 다음과 같다.  

```
semantic_similarity + (1.0 - decay_rate) ^ hours_passed
```  

해당 검색기는 객체의 마지막 접근 시간을 기준으로 `정보의 신선함` 을 평가한다. 
자주 접근 되는 객체는 시간이 지나도 높은 점수를 유지하여, 
중요한 정보가 검색 상위에 노출되 가능성이 높아진다. 
이는 최신성과 관련성을 모두 고려한 동적 검색 결과를 제공할 수 있다는 장점이 있다. 
시간이 지남에 따라 점수가 얼마나 감소하는지를 나타내는 비율인 `decacy_rate` 는 객체 생성 후가 아닌, 
마지막 접근 이후 경과 시간을 의미하므로, 자주 접근하는 객체는 계속 최신 상태로 유지된다는 점을 기억해야 한다.  

`decay_rate` 는 값에 따라 아래와 같이 해석할 수 있다. 

- 늦은 감쇠율(`low decay_rate`) : 감쇠율이 낮다는 것은 기억이 더 오래 기억됨을 의미한다. 즉 값이 0인 경우 기억이 절대 잊혀지지 않는다는 의미로, 이는 기존 벡터 저장소와 동일한 결과가 나오게 된다. 
- 높은 감쇠율(`high decay_rate`) : 감쇠율이 높다는 것은 기억이 더 빠르게 잊혀짐을 의미한다. 즉 값이 1인 경우 기억이 바로 잊혀지게 되므로, 이는 기존 벡터 저장소와 동일한 결과가 나오게 된다. 

`TimeWeightedVectorStoreRetriever` 는 신선도를 위한 필드로 `last_accessed_at` 와 `created_at` 을 사용한다. 

- `last_accessed_at` : 문서가 마지막으로 접근된 시간으로, 마지막으로 검색되거나 액세스된 시간을 자동으로 추적한다. 
- `created_at` : 문서가 처음 생성되거나 시스템에 추가된 시간을 의미한다. 

`낮은 감쇠율` 에대한 테스트는 아래와 같다.  

```python
from datetime import datetime, timedelta
from langchain.docstore import InMemoryDocstore
from langchain.retrievers import TimeWeightedVectorStoreRetriever
from langchain_core.documents import Document
from langchain_community.vectorstores import FAISS
import faiss

embedding_size = 768
index = faiss.IndexFlatL2(embedding_size)
low_time_weighted_vectorstore = FAISS(hf_embeddings, index, InMemoryDocstore({}), {})

low_time_weighted_retriever = TimeWeightedVectorStoreRetriever(
    vectorstore=low_time_weighted_vectorstore,
    # 0 에 가까운 값으로 설정
    decay_rate=0.0000000000000000000000001, 
    k=1
)

yesterday = datetime.now() - timedelta(days=1)

# 어제 문서
low_time_weighted_retriever.add_documents(
    [
        Document(
            page_content="최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다.",
            metadata={"last_accessed_at": yesterday},
        )
    ]
)

# 최신 문서
low_time_weighted_retriever.add_documents(
    [
        Document(
            page_content="최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다. new new !!",
        )
    ]
)


low_time_weighted_retriever.invoke("노트북")
# [Document(metadata={'last_accessed_at': datetime.datetime(2025, 5, 1, 10, 54, 54, 542202), 'created_at': datetime.datetime(2025, 5, 1, 10, 54, 54, 180153), 'buffer_idx': 0}, page_content='최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다.')]
```  

거의 동일한 문서 내용을 사용하고 `last_accessed_at` 만 다르게 설정했다. 
`낮은 감쇠율` 인 경우 어제 날짜로 설정된 문서가 의미적 유사도 입장에서는 가장 높기 때문에 결과로 반환 된 것을 확인 할 수 있다.  

다음은 `높은 감쇠율` 에 대한 테스트이다. 

```python
high_time_weighted_vectorstore = FAISS(hf_embeddings, index, InMemoryDocstore({}), {})

high_time_weighted_retriever = TimeWeightedVectorStoreRetriever(
    vectorstore=time_weighted_vectorstore,
    decay_rate=0.999, 
    k=1
)

yesterday = datetime.now() - timedelta(days=1)

high_time_weighted_retriever.add_documents(
    [
      Document(
          page_content="최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다.",
          metadata={"last_accessed_at": yesterday},
      )
    ]
)

high_time_weighted_retriever.add_documents(
    [
      Document(
          page_content="최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다. new new !!",
          # metadata={"created_at": datetime.now().timestamp()},
      )
    ]
)

high_time_weighted_retriever.invoke("노트북")
# [Document(metadata={'last_accessed_at': datetime.datetime(2025, 5, 1, 11, 20, 45, 916331), 'created_at': datetime.datetime(2025, 5, 1, 11, 20, 34, 316318), 'buffer_idx': 1}, page_content='최신 M3 프로세서를 탑재한 고성능 노트북, 복잡한 작업도 빠르게 처리하고 배터리 효율성이 뛰어납니다. new new !!')]
```  

`높은 감쇠율` 인 경우 `last_accessed_at` 이 어제 날짜로 설정된 문서가 의미적 유사도 입장에서는 가장 높지만,
`decay_rate` 가 높기 때문에 `high_time_weighted_retriever` 에서 검색할 때는 가장 최근에 추가된 문서가 결과로 반환된다.  


시간에 따른 변화가 검색 결과에 영향을 주기 때문에 
이를 좀 더 편리하게 테스트를 지원하는 방법이 제공된다. 
[mock_now](https://python.langchain.com/docs/how_to/time_weighted_vectorstore/#virtual-time) 
함수는 `LangChain` 에서 제공하는 유틸리티로, 현재 시간을 `mock` 하는 역할을 수행한다.  


### LongContextReorder
`LongConextReorder` 는 대규모 문맥(`long context`)을 처리할 떄, 제공된 텍스트를 중요도에 따라 재정렬하여 보다 효율적으로 처리할 수 있도록 돕는 기능이다.
이 기능은 모델에 입력하기 전에 데이터를 재구성하여 중요한 정보가 우선적으로 처리되도록 한다. 
재구성된 결과는 가장 중요한 정보가 시작과 끝에 오도록 재배치한다.  

- `LLM` 모델 주요 특징 : `LLM` 은 텍스트의 앞부분과 끝부분에 있는 정보를 더 잘 이애하고 반응하는 경향이 있다. [참고](https://arxiv.org/abs/2307.03172)
- 중요 정보 우선 처리 : 대규모 문서나 데이터셋에서 중요한 정보를 우선적으로 모델에 전달할 수 있다.
- 효율적 정보 전달 : 모델이 가장 중요한 정보부터 처리하도록 데이터의 순서를 재배치하여, 결과적으로 더 정확하고 유용한 응답을 얻을 수 있다.

벡터 검색기의 결과를 `LongContextReorder` 를 사용해 재정렬하면 아래와 같다.  

```python
from langchain_community.document_transformers import LongContextReorder


test_vector_retriever = memory_db.as_retriever(search_kwargs={"k":6 })
docs = test_vector_retriever.invoke("사람처럼 학습하고 추론하는 시스템은 ?")
# [Document(id='15ea84e6-f27b-4017-b99f-dadd738f0b7c', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(id='67e5572a-2727-4fc0-b3db-02e73f04f83b', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='1f727ace-037a-49ff-92f7-5c7261a39905', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'),
#  Document(id='790e5d6c-5f54-438e-8811-f7e589df4604', metadata={'source': './computer-keywords.txt'}, page_content='GPU\n\n정의: GPU(Graphics Processing Unit)는 컴퓨터의 그래픽 렌더링과 복잡한 병렬 처리를 전문적으로 수행하는 프로세서입니다.\n예시: NVIDIA GeForce RTX 3080은 게임 및 인공지능 학습에 활용되는 고성능 GPU입니다.\n연관키워드: 그래픽 카드, 렌더링, CUDA, 병렬 처리\n\nSSD'),
#  Document(id='029ac855-e8b0-4e89-9e31-55a638910217', metadata={'source': './computer-keywords.txt'}, page_content='IoT\n\n정의: IoT(Internet of Things)는 인터넷을 통해 데이터를 수집하고 교환할 수 있는 센서와 소프트웨어가 내장된 물리적 장치들의 네트워크입니다.\n예시: 스마트 홈 시스템은 조명, 온도 조절 장치, 보안 카메라 등을 인터넷에 연결하여 원격으로 제어할 수 있게 합니다.\n연관키워드: 스마트 기기, 센서, M2M, 연결성, 자동화\n\n인공지능'),
#  Document(id='0f897f42-735c-4708-b2f2-9ebaf4494758', metadata={'source': './computer-keywords.txt'}, page_content='빅데이터\n\n정의: 빅데이터는 기존 데이터베이스 도구로 처리하기 어려운 대량의 정형 및 비정형 데이터를 의미합니다.\n예시: 소셜 미디어 플랫폼은 매일 페타바이트 규모의 사용자 활동 데이터를 분석하여 맞춤 콘텐츠를 제공합니다.\n연관키워드: 하둡, 스파크, 데이터 마이닝, 분석, 볼륨\n\n머신러닝')]

# 재정렬
reordering = LongContextReorder()
reordered_docs = reordering.transform_documents(docs)
# [Document(id='67e5572a-2727-4fc0-b3db-02e73f04f83b', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='790e5d6c-5f54-438e-8811-f7e589df4604', metadata={'source': './computer-keywords.txt'}, page_content='GPU\n\n정의: GPU(Graphics Processing Unit)는 컴퓨터의 그래픽 렌더링과 복잡한 병렬 처리를 전문적으로 수행하는 프로세서입니다.\n예시: NVIDIA GeForce RTX 3080은 게임 및 인공지능 학습에 활용되는 고성능 GPU입니다.\n연관키워드: 그래픽 카드, 렌더링, CUDA, 병렬 처리\n\nSSD'),
#  Document(id='0f897f42-735c-4708-b2f2-9ebaf4494758', metadata={'source': './computer-keywords.txt'}, page_content='빅데이터\n\n정의: 빅데이터는 기존 데이터베이스 도구로 처리하기 어려운 대량의 정형 및 비정형 데이터를 의미합니다.\n예시: 소셜 미디어 플랫폼은 매일 페타바이트 규모의 사용자 활동 데이터를 분석하여 맞춤 콘텐츠를 제공합니다.\n연관키워드: 하둡, 스파크, 데이터 마이닝, 분석, 볼륨\n\n머신러닝'),
#  Document(id='029ac855-e8b0-4e89-9e31-55a638910217', metadata={'source': './computer-keywords.txt'}, page_content='IoT\n\n정의: IoT(Internet of Things)는 인터넷을 통해 데이터를 수집하고 교환할 수 있는 센서와 소프트웨어가 내장된 물리적 장치들의 네트워크입니다.\n예시: 스마트 홈 시스템은 조명, 온도 조절 장치, 보안 카메라 등을 인터넷에 연결하여 원격으로 제어할 수 있게 합니다.\n연관키워드: 스마트 기기, 센서, M2M, 연결성, 자동화\n\n인공지능'),
#  Document(id='1f727ace-037a-49ff-92f7-5c7261a39905', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'),
#  Document(id='15ea84e6-f27b-4017-b99f-dadd738f0b7c', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치')]
```  

위 재정렬된 결과를 보면 질의 중에서 가장 중요한 인공지능과 머신러닝에 대한 문서가 시작과 끝에 위치한 것을 확인할 수 있다.  

`ContextReorder` 를 검색기와 함께 사용하는 구현 예시는 아래와 같다.  

```python

from langchain.prompts import ChatPromptTemplate
from operator import itemgetter
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda

def format_docs(docs):
  return "\n\n\n".join(
      [
          f"[{i}] {doc.page_content}"
          for i, doc in enumerate(docs)
      ]
  )

def reorder_documents(docs):
  reordering = LongContextReorder()
  redordered_docs = reordering.transform_documents(docs)
  result = format_docs(redordered_docs) 
  print(result)
  return result


template = """다음 텍스트 발췌를 참고하세요:
{context}

-----
다음 질문에 답변해주세요:
{query}

다음 언어로 답변해주세요: {lang}
"""

prompt = ChatPromptTemplate.from_template(template)

chain = (
        {
          "context": itemgetter("query")
                     | test_vector_retriever
                     | RunnableLambda(reorder_documents),
          "query" : itemgetter("query"),
          "lang" : itemgetter("lang")
        }
        | prompt
        | model
        | StrOutputParser()
)
```  

```python
results = chain.invoke({
      "query" : "사람처럼 학습하고 추론하는 시스템에서 높은 가용량을 보장하는 방법은 ?",
      "lang" : "한국어"
    })
# 인공지능(AI) 시스템에서 높은 가용성을 보장하는 방법은 여러 가지가 있습니다. 하지만, 가장 대표적인 방법은 분산 시스템을 활용하여 클러스터링하는 것입니다. 클러스터링을 통해 여러 대의 컴퓨터 또는 서버를 결합하여 하나의 시스템처럼 작동하도록 만듭니다. 이렇게 하면, 시스템의 일부가 장애가 발생하더라도 다른 컴퓨터 또는 서버가 작업을 인계받아 시스템의 가용성을 높일 수 있습니다. 예를 들어, Kubernetes와 같은 컨테이너 오케스트레이션 도구를 사용하여 클러스터를 관리하면, 애플리케이션의 확장성과 안정성을 제공할 수 있습니다.
```



---  
## Reference
[Retrievers](https://python.langchain.com/docs/concepts/retrievers/)  
[How to do retrieval with contextual compression](https://python.langchain.com/docs/how_to/contextual_compression/)  
[Self-querying retrievers](https://python.langchain.com/docs/integrations/retrievers/self_query/)  
[time-weighted vectorstore retrievers](https://python.langchain.com/docs/how_to/time_weighted_vectorstore/#virtual-time)  
[How to add scores to retriever results](https://python.langchain.com/docs/how_to/add_scores_retriever)  
[retrievers](https://python.langchain.com/api_reference/langchain/retrievers.html)  
[검색기(Retriever)](https://wikidocs.net/233779)  


