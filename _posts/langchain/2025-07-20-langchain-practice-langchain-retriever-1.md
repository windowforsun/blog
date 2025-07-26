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
`LangChain` 에서 `Retriever` 는 검색(`retrieval`) 작업을 수행하는 핵심적인 구성 요소로, 
정보 검색 및 문맥 제공의 중심 역할을 한다. 
`Retriever` 는 대규모 데이터 소스에서 정보를 찾아 `LLM` 이 보다 정확하고 맥락에 적합한 응답을 생성할 수 있도록 도움을 준다.  

사용자가 제공한 질문(쿼리)에 따라 데이터 소스(예: 데이터베이스, 문서, 웹 페이지 등)에서 관련 정보를 검색하는 기능을 담당한다. 
언어 모델과 결합해 `RAG(Retrieval-Augmented Generation)` 아키텍처를 구현하는 아래와 같은 주요한 기능을 제공한다.  

- 정보 검색 : 특정 데이터베이스나 문서에서 사용자의 질문과 관련된 정보를 찾아낸다. 
- 맥락 제공 : 검색된 정보를 바탕으로 `LLM` 이 다 나은 답변을 생성할 수 있도록 필요한 배경 지식을 제공한다. 
- 효율적 데이터 처리 : 대규모 데이터에서 필요한 정보를 빠르고 정확하게 검색한다. 

`Retriever` 의 동작 방식은 아래와 같다.

1. 사용자 쿼리 수신 : 사용자가 입력한 질문이 `Retriever` 에 전달된다. 
2. 쿼리 처리 및 변환 : 자연어로 작성된 질문을 검색에 적합한 형식으로 변환한다. (벡터 임베딩으로 변환하거나 키워드 기반 쿼리 생성)
3. 데이터 검색 : 변환된 쿼리를 기반으로 데이터베이스나 문서 저장소에서 관련 정보를 검색한다. 검색 방식은 텍스트 기반 검색, 벡터 검색 등 다양하다. 
4. 결과 반환 : 검색된 데이터를 정렬하고, 가장 관련성이 높은 정보를 반환한다. 이 데이터는 `LLM` 의 입력으로 사용되어 더 나은 답변을 생성하는데 활용된다. 

주요한 `Retriever` 의 종류는 아래와 같은 것들이 있다. 

- `VectorStoreRetriever` : `VectorStore` 기반으로 동작하고, 문서의 `Embedding` 을 생성해 사용자의 쿼리와 문서 간의 의미적 유사성을 기준으로 검색한다. 대규모 문서 집합에서 의미적으로 유사한 정보를 검색할 때 사용할 수 있다. 
- `BM25Retriever` : `BM25` 알고리즘을 기반으로 한 전통적인 키워드 기반 검색기로, 문서와 쿼리의 키워드 매칭에 의존한다. 전통적인 텍스트 검색과 비슷해, 키워드 매칭 알고리즘으로 간단하고 빠른 검색이 필요할 떄 사용할 수 있다. 
- `ContextualCompressionRetriever` : 검색된 문서에서 중요한 컨텍스트만 추출해 문서 크기를 줄이고, `LLM` 이 사용하는 데 필요한 핵심 정보를 제공한다. 대규모 데이터를 효율적으로 처리하기 위해 불필요한 정보를 제거할 때 사용할 수 있다. 
- `EnsembleRetriever` : 서로 다른 검색기(`Sparse`, `Dense`)의 결과를 조합하여 최적의 검색 성능을 제공한다. `RRF`(Ranked Retrieval Fusion) 알고리즘을 사용해 결과를 재정렬한다. 다양한 검색 전략을 결합해 검색 정확도 향상이 필요한 경우 사용할 수 있다. 
- `ParentDocumentRetriever` : 특정 문서나 문장에 연결된 부모 문서를 반환하여 더 넓은 맥락을 제공한다. 특정 문장의 출처 추적이 필요한 경우 사용할 수 있다. 
- `MultiQueryRetriever` : 하나의 질문을 여러 검색 쿼리로 변환해 다양한 관점에서 검색 결과를 얻는다. 다의어 또는 다양한 표현이 가능한 질문에 적합하게 사용할 수 있다. 
- `MultiVectorRetriever` : 하나의 문서에 대해 여러 벡터를 생성해 다양한 관점에서 검색을 수행한다. 다중 언어 문서 또는 복잡한 데이터셋에서 의미적 검색이 필요한 경우 사용할 수 있다. 
- `SelfQueryRetriever` : 사용자의 질문을 분석하여 검색 쿼리를 자동으로 생성한다. 자연어를 데이터베이스 쿼리로 변환하는 기능을 제공한다. 사용자가 `SQL` 이나 복잡한 쿼리 언어에 익숙하지 않은 경우 유용하다. 
- `TimeWeightedVectorStoreRetriever` : 시간 정보를 기반으로 가중치를 부여하여 최신 정보에 우선순위르 두는 벡터 스토어 검색기이다. 최신 정보가 중요한 데이터 검색에서 사용할 수 있다. 
- `ElasticsearchRetriever` : `Elasticsearch` 를 기반으로 한 고성능 검색기로, 대규모 텍스트 데이터를 빠르게 검색할 수 있다. 검색 엔진이나 추천 시스템에 활용할 수 있다. 

정보 검색에서 데이터를 검색하는 주요 방식에는 `Sparse Retriever` 와 `Dense Retriever` 가 있다. 
이 두 접근 방식은 쿼리 간의 유사성을 계산하는 방식에서 큰 차이를 보이며, 특정 사용 사례에 따라 적합성이 달라진다. 

- `Sparse Retriever` : 전통적인 키워드 기반 검색에 속하며, 문서와 쿼리 간의 직접적인 단어 매칭을 기반으로 작동한다. 즉 단어의 정확한 일치를 중요하게 여긴다. 
  - `TF-IDF`(Term Frequency-Inverse Document Frequency) : 문서 내 단어의 중요도를 평가하는 통계적 방법으로, 특정 단어가 문서에서 얼마나 자주 등장하는지를 기반으로 한다.
  - `BM25` : `TF-IDF` 의 확장으로, 문서의 길이와 단어 빈도를 고려해 점수를 계산한다.
  - 이렇게 `Sparse Retriever` 는 각 단어의 존재 여부만을 고려하기 때문에 계산 비용이 낮고, 구현이 간단하다. 검색 결과의 품질이 키워드의 선택에 크게 의존한다는 의미이기도 하다. 
- `Dense Retriever` : 딥러닝 기반의 `Embedding` 을 사용하여 문서와 쿼리를 의미적으로 표현하고, 유사성을 계산하는 현대적인 접근 방식이다. 키워드가 완벽하게 일치하지 않아도 의미적으로 관련된 문서를 검색할 수 있다. 


| 특징              | Sparse Retriever                              | Dense Retriever                              |
|-----------------------|--------------------------------------------------|-------------------------------------------------|
| 기반              | 키워드 매칭                                       | 의미적 유사성                                   |
| 표현 방식         | 희소 벡터(TF-IDF, BM25 등)                        | 밀집 벡터(딥러닝 임베딩)                        |
| 성능              | 작은 데이터셋에서 빠르고 효율적                   | 대규모 데이터셋에서 뛰어난 검색 성능            |
| 설명 가능성       | 검색 결과가 설명 가능                              | 결과의 해석이 어려울 수 있음                    |
| 구현 도구         | ElasticSearch, BM25 등                            | FAISS, Pinecone, Weaviate 등                    |
| 사용 사례         | 키워드 중심의 간단한 검색                          | 문맥적 이해가 필요한 의미 기반 검색             |
| 확장성            | 데이터 크기가 커질수록 성능 저하 가능              | 대규모 데이터셋에 적합                          |
| 유사성 계산 방법  | 단순 키워드 매칭                                   | 코사인 유사도 등 벡터 간 유사성 계산            |
| 데이터 준비       | 사전 처리 작업이 적음                              | 데이터 임베딩 생성 및 모델 학습 필요            |



예제는 `Chroma` 벡터 스토어를 사용하고 아래와 같은 키워드 문서를 사용한다.

```text
CPU

정의: CPU(Central Processing Unit)는 컴퓨터의 두뇌 역할을 하는 하드웨어 구성 요소로, 연산과 명령어 실행을 담당합니다.
예시: Intel Core i9, AMD Ryzen 9 같은 프로세서는 고성능 컴퓨팅을 위한 CPU입니다.
연관키워드: 프로세서, 코어, 클럭 속도, 연산 처리

RAM

정의: RAM(Random Access Memory)은 컴퓨터가 현재 작업 중인 데이터와 프로그램을 임시로 저장하는 휘발성 메모리입니다.
예시: 16GB DDR4 RAM을 장착한 노트북은 여러 프로그램을 동시에 실행할 때 더 나은 성능을 제공합니다.
연관키워드: 메모리, 휘발성, DDR, 임시 저장

GPU

정의: GPU(Graphics Processing Unit)는 컴퓨터의 그래픽 렌더링과 복잡한 병렬 처리를 전문적으로 수행하는 프로세서입니다.
예시: NVIDIA GeForce RTX 3080은 게임 및 인공지능 학습에 활용되는 고성능 GPU입니다.
연관키워드: 그래픽 카드, 렌더링, CUDA, 병렬 처리

SSD

정의: SSD(Solid State Drive)는 기계적 부품 없이 플래시 메모리를 사용하는 저장 장치로, 기존 하드 디스크보다 빠른 읽기/쓰기 속도를 제공합니다.
예시: 노트북에 1TB NVMe SSD를 설치하면 운영체제 부팅 시간이 크게 단축됩니다.
연관키워드: 저장 장치, 플래시 메모리, NVMe, SATA

운영체제

정의: 운영체제는 컴퓨터의 하드웨어 자원을 관리하고 응용 프로그램과 사용자 간의 인터페이스를 제공하는 시스템 소프트웨어입니다.
예시: Windows 11, macOS, Linux Ubuntu는 널리 사용되는 데스크톱 운영체제입니다.
연관키워드: Windows, macOS, Linux, 시스템 소프트웨어

방화벽

정의: 방화벽은 승인되지 않은 접근으로부터 컴퓨터 네트워크를 보호하는 보안 시스템으로, 들어오고 나가는 네트워크 트래픽을 모니터링하고 제어합니다.
예시: 윈도우 기본 방화벽은 사용자의 컴퓨터를 외부 위협으로부터 보호하는 첫 번째 방어선입니다.
연관키워드: 네트워크 보안, 패킷 필터링, 침입 방지, 포트 차단

클라우드 컴퓨팅

정의: 클라우드 컴퓨팅은 인터넷을 통해 서버, 스토리지, 데이터베이스, 소프트웨어 등의 컴퓨팅 리소스를 제공하는 서비스입니다.
예시: AWS, Microsoft Azure, Google Cloud Platform은 기업들이 자체 서버 인프라 구축 없이 필요한 만큼 IT 자원을 사용할 수 있게 해줍니다.
연관키워드: IaaS, PaaS, SaaS, 서버리스, 확장성

API

정의: API(Application Programming Interface)는 서로 다른 소프트웨어 애플리케이션이 통신할 수 있게 해주는 인터페이스입니다.
예시: 날씨 앱은 기상청 API를 통해 실시간 날씨 데이터를 가져와 사용자에게 표시합니다.
연관키워드: REST, SOAP, 엔드포인트, JSON, 웹서비스

빅데이터

정의: 빅데이터는 기존 데이터베이스 도구로 처리하기 어려운 대량의 정형 및 비정형 데이터를 의미합니다.
예시: 소셜 미디어 플랫폼은 매일 페타바이트 규모의 사용자 활동 데이터를 분석하여 맞춤 콘텐츠를 제공합니다.
연관키워드: 하둡, 스파크, 데이터 마이닝, 분석, 볼륨

머신러닝

정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.
예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.
연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링

가상화

정의: 가상화는 물리적 컴퓨터 자원을 여러 가상 환경으로 나누어 효율적으로 사용할 수 있게 하는 기술입니다.
예시: VMware, VirtualBox와 같은 소프트웨어는 하나의 물리적 서버에서 여러 운영체제를 동시에 실행할 수 있게 합니다.
연관키워드: 하이퍼바이저, VM, 컨테이너, 리소스 최적화

블록체인

정의: 블록체인은 분산된 컴퓨터 네트워크에서 데이터 블록이 암호화 기술로 연결된 디지털 장부 시스템입니다.
예시: 비트코인은 블록체인 기술을 활용하여 중앙 은행 없이 안전한 금융 거래를 가능하게 합니다.
연관키워드: 암호화폐, 분산원장, 스마트 계약, 합의 알고리즘

알고리즘

정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.
예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.
연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화

DNS

정의: DNS(Domain Name System)는 사람이 읽을 수 있는 도메인 이름을 컴퓨터가 인식할 수 있는 IP 주소로 변환하는 시스템입니다.
예시: 사용자가 브라우저에 'www.example.com'을 입력하면 DNS가 해당 웹사이트의 IP 주소(예: 192.168.1.1)로 변환합니다.
연관키워드: 도메인, 네임서버, IP 주소, URL

인터넷 프로토콜

정의: 인터넷 프로토콜은 데이터가 네트워크를 통해 어떻게 전송되는지 정의하는 규칙 세트입니다.
예시: HTTP, HTTPS, FTP, SMTP는 모두 특정 유형의 데이터 전송을 위한 인터넷 프로토콜입니다.
연관키워드: TCP/IP, HTTP, HTTPS, 패킷, 라우팅

데이터베이스

정의: 데이터베이스는 구조화된 형식으로 데이터를 저장, 관리, 검색할 수 있는 전자적 시스템입니다.
예시: MySQL, PostgreSQL, MongoDB는 다양한 응용 프로그램에서 사용되는 인기 있는 데이터베이스 시스템입니다.
연관키워드: SQL, NoSQL, DBMS, 쿼리, 테이블

웹 브라우저

정의: 웹 브라우저는 인터넷에서 웹 페이지를 검색, 표시하고 사용자가 웹 콘텐츠와 상호작용할 수 있게 해주는 소프트웨어 애플리케이션입니다.
예시: Google Chrome, Mozilla Firefox, Safari, Microsoft Edge는 널리 사용되는 웹 브라우저입니다.
연관키워드: HTML, CSS, JavaScript, 렌더링 엔진, 웹 표준

사이버 보안

정의: 사이버 보안은 컴퓨터 시스템, 네트워크, 데이터를, 무단 접근과 공격으로부터 보호하는 기술, 프로세스 및 관행입니다.
예시: 안티바이러스 소프트웨어, 암호화, 다중 인증은 모두 사이버 보안을 강화하는 방법입니다.
연관키워드: 해킹, 멀웨어, 피싱, 암호화, 취약점

IoT

정의: IoT(Internet of Things)는 인터넷을 통해 데이터를 수집하고 교환할 수 있는 센서와 소프트웨어가 내장된 물리적 장치들의 네트워크입니다.
예시: 스마트 홈 시스템은 조명, 온도 조절 장치, 보안 카메라 등을 인터넷에 연결하여 원격으로 제어할 수 있게 합니다.
연관키워드: 스마트 기기, 센서, M2M, 연결성, 자동화

인공지능

정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.
예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.
연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전
```  

위 문서를 불러와 `Chroma` 벡터 스토어에 저장하는 등 사전 작업 내용은 아래와 같다.

```python
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma
from langchain_huggingface.embeddings import HuggingFaceEndpointEmbeddings
from langchain_huggingface.embeddings import HuggingFaceEmbeddings


os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_XXXX"
model_name = "BM-K/KoSimCSE-roberta"
hf_endpoint_embeddings = HuggingFaceEndpointEmbeddings(
    model=model_name,
    huggingfacehub_api_token=os.environ["HUGGINGFACEHUB_API_TOKEN"],
)
hf_embeddings = HuggingFaceEmbeddings(
    model_name=model_name,
    encode_kwargs={'normalize_embeddings':True},
)

text_splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=100)
computerKeywordLoader = TextLoader("./computer-keywords.txt")
split_computer_keywords = computerKeywordLoader.load_and_split(text_splitter)
print(len(split_computer_keywords))
# 20

memory_db = Chroma.from_documents(documents=split_computer_keywords, embedding=hf_embeddings, collection_name="computer_keywords_db")
```  


### VectorStoreRetriever
`VectorStoreRetriever` 는 `VectorStore` 를 기반으로 문서와 사용자의 쿼리 간의 `Semantic Similarity` 를 계산해 관련 문서를 검색한다. 
이는 전통적인 키워드 기반 검색 방식(`Sparse Retrieval`)과 달리, 문서와 쿼리를 벡터로 변환해 의미를 이해하고 검색하는 `Dense Retrieval` 방식을 따른다.


#### as_retriever()
`as_retriever` 메서드는 `VectorStore` 객체를 `VectorStoreRetriever` 객체로 변환하는 메서드이다. 
해당 메서드를 사용해서 다양한 검색 옵션을 설정해 사용자의 요구사항에 맞는 문서 검색을 수행할 수 있다.  

`add_documents` 메서드는 아래와 같은 인자를 받는다. [참고](https://python.langchain.com/api_reference/core/vectorstores/langchain_core.vectorstores.base.VectorStore.html#langchain_core.vectorstores.base.VectorStore.as_retriever)

- `search_kwargs` : 검색 시 사용할 추가 인자들을 설정할 수 있다.
  - `k` : 검색할 문서의 개수(기본값 4)
  - `score_threshold` : `similarity_score_threshold` 검색에서 최소 유사도 임계값
  - `fetch_k` : `MMR` 검색에서 전달할 문서 수(기본값 20)
  - `lambda_mult` : `MMR` 검색의 다양성 조절(0~1 사이값, 기본값 0.5)
  - `filter` : 메타데이터 기반 필터링
- `**kwargs` : 검색 함수에 전달할 키워드 인자
- `search_type` : 검색 유형
  - `similarity` : 유사도 검색
  - `mmr` : 최대 다양성 검색(유사도와 다양성을 동시에 고려하는 방식)
  - `similarity_score_threshold` : 유사도 점수 임계값 검색

참고/주의 사항으로는 아래와 같다. 

- `search_type` : 단순한 의미적 유사성이 중요하다면 `similarity` 검색을 사용하고, 검색 결과가 유사한 항목으로 과도하게 치우치지 않게 하려면 `mmr` 사용을 고려할 수 있다. 
- `search_kwargs`
  - `k` 값을 너무 낮게 설명하면 관련 문서를 놓칠 수 있고, 너무 높게 설정하면 `LLM` 처리 비용이 증가할 수 있다. (일반적으로 3~10 사이의 값이 적절하다.)
  - `MMR` 사용시 `lambda_mul` 값을 실험적으로 조정하여 최적의 검색 품질을 찾는 것이 좋다. 
  - `score_threshold` : 너무 높은 임계값을 설정하면 결과가 반환되지 않을 수 있다. 일반적으로 `0.6~0.8` 사이의 값을 사용하여 적절한 결과를 얻도록 조절이 필요하다. 
- `VectoreStoreRetriever` 검색 결과의 품질은 사용된 임베딩 품질에 크게 의존함을 기억해야 한다. 
- `MMR` 검색은 검색 결과의 다양성을 높여줄 수 있지만, 계산 비용이 추가로 발생한다. 필요하지 않다면 기본 값인 `similarity` 를 사용하는 것이 효율작이다. 
- `retriever.get_relevant_documents(query)` 를 호출해 반환된 문서와 유사도 점수를 확인하여 최적화/디버깅하는 것이 좋다.  

```python
vector_retriever = memory_db.as_retriever()
```  


#### invoke()
`invoke` 메서드는 `Retriever` 객체에서 동기적으로 검색을 수행하는 메서드이다. 

`invoke` 메서드는 아래와 같은 인자를 받는다. [참고](https://python.langchain.com/api_reference/core/vectorstores/langchain_core.vectorstores.base.VectorStoreRetriever.html#langchain_core.vectorstores.base.VectorStoreRetriever.invoke)

- `input` : 검색할 쿼리
- `config` : `Retriever` 구성 정보
- `**kwargs` : `Retriever` 에 전달할 추가 인자

```python
vector_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(id='fe9306c1-cbbe-4889-be65-5c5cff58c211', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전'),
#  Document(id='b9c7972e-e484-4daa-a9e8-57ded0fabc0b', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='04b6978a-c99e-4447-bda5-90c1798054e3', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'),
#  Document(id='8acab6b5-f194-4056-b061-fbc378633506', metadata={'source': './computer-keywords.txt'}, page_content='GPU\n\n정의: GPU(Graphics Processing Unit)는 컴퓨터의 그래픽 렌더링과 복잡한 병렬 처리를 전문적으로 수행하는 프로세서입니다.\n예시: NVIDIA GeForce RTX 3080은 게임 및 인공지능 학습에 활용되는 고성능 GPU입니다.\n연관키워드: 그래픽 카드, 렌더링, CUDA, 병렬 처리\n\nSSD')]
```  

#### MMR
`MMR`(Maximal Marginal Relevance) 검색은 `VectorStoreRetriever` 에서 제공하는 검색 방법 중 하나로,
검색 결과를 정렬할 때 유사성과 다양성을 동시에 고려하는 방식이다.  

- 유사성(`Similarity`) : 사용자의 쿼리와 문서간의 의미적 유사성을 최대화
- 다양성(`Diversity`) : 반환된 문서들 간의 중복을 최소화하여 다양한 주제를 포함

`MMR` 은 다음과 같은 절차로 동작한다. 

1. 초기 후보군 선택 : `fetch_k` 개수의 문서 만큼 쿼리와 가장 유사한 문서를 초기 후보군으로 선정한다. 
2. 유사성과 다양성 계산 : 각 문서의 쿼리와의 유사도를 계산하고(유사성 점수) 반환된 기존 문서들과의 중복도를(다양성 점수) 계산한다.
3. 최적 문서 선택 : 유사성과 다양성 간의 균형을 고려하여 최적의 문서를 선택한다. 
4. 반환된 문서 리스트의 수가 `k` 개에 도달할 때까지 과정을 반복한다. 

`MMR` 검색을 사용하는 `Retriever` 를 생성하는 `as_retriever()` 의 인자 설정 예시는 아래와 같다. 

- `search_type` : `mmr`
- `search_kwargs` : 
  - `k` : 최종 반환할 문서 수
  - `fetch_k` : 후보군으로 검색할 문서의 수
  - `lambda_mult` : 유사성과 다양성 간의 균형을 조절하는 값(0~1 사이값)
    - 0(다양성↑, 유사성↓) : 다양성(`Diversity`)에 초점을 두고, 유사성은 무시한다. 검색 결과가 중복되지 않도록 다양한 주제와 내용을 가진 문서들이 선택된다. 
    - 1(유사성↑, 다양성↓) : 유사성(`Similarity`)에 초점을 두고, 다양성은 무시한다. 사용자의 쿼리와 가장 유사한 문서들만 반환한다. 결과가 중복될 가능성이 높아진다. 

```python
mmr_vector_retriever = memory_db.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 4,
        "fetch_k" : 20,
        "lambda_mult": 0
    }
)

mmr_vector_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(id='fe9306c1-cbbe-4889-be65-5c5cff58c211', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전'),
#  Document(id='04b6978a-c99e-4447-bda5-90c1798054e3', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'),
#  Document(id='ddf519a6-44a7-4477-a998-8e121a1dbeef', metadata={'source': './computer-keywords.txt'}, page_content='SSD\n\n정의: SSD(Solid State Drive)는 기계적 부품 없이 플래시 메모리를 사용하는 저장 장치로, 기존 하드 디스크보다 빠른 읽기/쓰기 속도를 제공합니다.\n예시: 노트북에 1TB NVMe SSD를 설치하면 운영체제 부팅 시간이 크게 단축됩니다.\n연관키워드: 저장 장치, 플래시 메모리, NVMe, SATA\n\n운영체제'),
#  Document(id='176ee677-26e4-4720-a10e-912ac36f859f', metadata={'source': './computer-keywords.txt'}, page_content='방화벽\n\n정의: 방화벽은 승인되지 않은 접근으로부터 컴퓨터 네트워크를 보호하는 보안 시스템으로, 들어오고 나가는 네트워크 트래픽을 모니터링하고 제어합니다.\n예시: 윈도우 기본 방화벽은 사용자의 컴퓨터를 외부 위협으로부터 보호하는 첫 번째 방어선입니다.\n연관키워드: 네트워크 보안, 패킷 필터링, 침입 방지, 포트 차단\n\n클라우드 컴퓨팅')]
```  


#### similarity_score_threshold
`similarity_score_threshold` 검색은 검색 결과의 유사도 점수를 기준으로 반환할 문서를 필터링 하는 방식이다. 
쿼리와 문서간의 유사도를 계산하고, 설정된 임계값 이상을 만족하는 문서만 반환한다. 
이를 통해 검색 결과의 품질을 높이고, 낮은 관련성의 문서를 걸러낼 수 있다.  

유사도 점수는 쿼리의 임베딩과 문서 임베딩 간의 `코사인 유사도(Cosine Similarity)` 를 사용하여 계산한다. 
`0~1` 사이의 값으로 표현되고, 1에 가까울 수록 유사성이 높고, 0에 가까울 수록 유사성이 낮다. 
일반적으로 `0.6~0.8` 사이의 값이 적합하다고 알려져 있다. 
하지만 도메인이나 데이터의 특성에 따라 적절한 임계값을 설정하는 것이 필요하다. 

만약 `similarity_score_threshold` 를 만족하는 문서가 `k` 보다 적다면, 
반환되는 결과의 개수가 줄어들 수 있다. 

```python
similarity_score_vector_retriever = memory_db.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "score_threshold": 0.4
    }
)

similarity_score_vector_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(id='35c69bca-fbbc-4ff9-bb45-96f7d1453450', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전')]
```  

#### top_k
`top_k` 는 쿼리에 대해 반환할 가장 관련성이 높은 검색 결과의 개수를 설정하는 매개변수이다. 
검색 엔진에서 상위 `k` 개의 결과를 반환하는 것과 유사하다. 
임베딩 기반 검섹에서는 쿼리 벡터와 저장소에 저장된 문서 벡터 간의 유사도를 계산하여, 
이 유사도 값이 높은 상위 `k` 개의 결과를 반환한다.  

`top_k` 를 사용해서 가장 유사도가 높은 검색 결과를 반환하는 예시는 아래와 같다.  

```python
top_1_retriever = memory_db.as_retriever(search_kwargs={"k" : 1})

top_1_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(id='35c69bca-fbbc-4ff9-bb45-96f7d1453450', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전')]
```

#### Configurable Retriever
`ConfigurableField` 를 사용하면 동적으로 조정 가능한 필드를 정의해 검색기를 사용할 수 있다. 
외부 입력에 따라 동적으로 검색기의 구성을 변경하거나 미리 정의된 값을 상황에 따라 사용하도록 할 수 있다.
즉 런타임에 검색기의 설정을 동적으로 변경할 수 있다.  

`ConfigurableField` 는 검색 매세변수의 고유 식별자, 이름 등을 설정하는 역할을 하고, 
검색 설정을 조정하기 위해서는 `config` 매개변수를 사용한다.  

검색기에서 `search_type` 과 `search_kwargs` 필드를 동적 설정할 수 있도록 설정해 생성하면 아래와 같다.  

```python
from langchain_core.runnables import ConfigurableField

configurable_retriever = memory_db.as_retriever(search_kwargs={"k":1}).configurable_fields(
      search_type=ConfigurableField(
          # 검색 매개변수의 식별자
          id="search_type",
          # 검색 매개변수의 이름
          name="Search Type",
          # 매개변수 설명
          description="Type for Search"
      ),
      search_kwargs=ConfigurableField(
          id="search_kwargs",
          name="Search Kwargs",
          description="Kwargs for Search",
      ),
  )
```  

런타임에 동적으로 `top_k` 설정을 변경해 검색을 수행하면 아래와 같다.  

```python
config = {
    "configurable" : {
        "search_kwargs" : {
            "k": 3
        }
    }
}

configurable_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?", config=config)
# [Document(id='35c69bca-fbbc-4ff9-bb45-96f7d1453450', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전'),
# Document(id='764387b2-b8df-459f-88a8-10f9975511cc', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
# Document(id='91faf2d5-09af-4d17-81e2-3c891a5507d9', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS')]
```  

`MMR` 검색 방식을 사용해 최대 다양성으로 설정을 변경해 검색을 수행하면 아래와 같다.  

```python
config = {
    "configurable" : {
        "search_type" : "mmr",
        "search_kwargs" : {
          "k": 4,
          "fetch_k" : 20,
          "lambda_mult": 0
        }
    }
}

configurable_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?", config=config)
# [Document(id='35c69bca-fbbc-4ff9-bb45-96f7d1453450', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전'),
#  Document(id='91faf2d5-09af-4d17-81e2-3c891a5507d9', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'),
#  Document(id='96f22d03-317a-4e0d-8078-54af0003d4ee', metadata={'source': './computer-keywords.txt'}, page_content='SSD\n\n정의: SSD(Solid State Drive)는 기계적 부품 없이 플래시 메모리를 사용하는 저장 장치로, 기존 하드 디스크보다 빠른 읽기/쓰기 속도를 제공합니다.\n예시: 노트북에 1TB NVMe SSD를 설치하면 운영체제 부팅 시간이 크게 단축됩니다.\n연관키워드: 저장 장치, 플래시 메모리, NVMe, SATA\n\n운영체제'),
#  Document(id='1a2e64a6-77c9-464b-8783-16aaa8213a22', metadata={'source': './computer-keywords.txt'}, page_content='방화벽\n\n정의: 방화벽은 승인되지 않은 접근으로부터 컴퓨터 네트워크를 보호하는 보안 시스템으로, 들어오고 나가는 네트워크 트래픽을 모니터링하고 제어합니다.\n예시: 윈도우 기본 방화벽은 사용자의 컴퓨터를 외부 위협으로부터 보호하는 첫 번째 방어선입니다.\n연관키워드: 네트워크 보안, 패킷 필터링, 침입 방지, 포트 차단\n\n클라우드 컴퓨팅')]
```  


### EnsembleRetriever
`EnsembleRetriever` 는 다중 검색기를 조합하여 동작하는 메커니즘이다. 
여러 개의 `Retriever` 를 동시에 사용하고, 각각의 검색 결과를 합치거나 가중치를 기반으로 결합하여 최종 결과를 반환한다. 
이는 하나의 `Retriever` 는 벡터 임데딩 기반 검색을 사용하고, 다른 `Retriever` 는 키워드 기반 검색을 사용하므로 서로 보완적인 검색 결과를 제공할 수 있다. 
그러므로 단일 알고리즘이 놓칠 수 있는 정보를 보완할 수 있다는 장점이 있어, 
여러 검색기의 결과를 통합하여 더 정확하고 풍부한 문서를 반환할 수 있다.  

`EnsembleRetriever` 는 하나의 `Retrievver` 는 벡터 임베딩 기반 검색을 사용하고, 
다른 `Retriever` 는 키워드 기반 검색을 사용해 서로 보완적인 검색 결과를 제공할 수 있다.  

동작원리는 다음과 같다. 
1. 여러 개의 `Retriever` 를 생성한다.
2. 각 `Retriever` 에 대해 쿼리를 실행하여 검색 결과를 얻는다. 
3. 각 검색기의 검색결과의 점수와 매핑되 가중치를 곱해 최종 점수 산출
3. 최종 점수를 기반으로 결과 정렬

아래는 `Chroma` 검색기와 `BM25Retriever` 를 사용해 `EnsembleRetriever` 를 생성하는 예시이다. 
`weights` 의 매개번수의 역할은 각 검색기가 반환하는 결과의 중요도를 나타낸다. 
그러므로 가중치는 단순히 겨로가의 정렬 점수에 영향을 미치는 비율로 적용된다. 
만약 가중치가 0인 검색기가 있다면, 해당 검색기의 결과는 여젼히 최종결과에 포함되지만 
중요도가 아주 낮기 때문에 가장 마지막 부분에 위치할 것이다. 

```python
from langchain.retrievers import BM25Retriever, EnsembleRetriever

bm25_retriever = BM25Retriever.from_documents(split_computer_keywords)
bm25_retriever.k = 1

chroma_vectorstore = Chroma.from_documents(documents=split_computer_keywords, embedding=hf_embeddings, collection_name="computer_keywords_db")

chroma_retriever = chroma_vectorstore.as_retriever(search_kwargs={"k":1})

ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, chroma_retriever],
    weights=[0.7, 0.3]
)
```  

질의를 보내면 `BM25Retriever` 와 `Chroma` 검색기를 사용해 검색을 수행하고 이를 합쳐 최종 결과를 도출하는 것을 볼 수 있다. 


```python
query = "사람처럼 학습하고 추론하는 시스템은?"
ensemble_result = ensemble_retriever.invoke(query)
print(ensemble_result)
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'), 
# Document(id='aff2aa55-c67b-4915-8430-320bb163f464', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전')]

bm_25_result = bm25_retriever.invoke(query)
print(bm_25_result)
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화')]

chroma_result = chroma_retriever.invoke(query)
print(chroma_result)
# [Document(id='aff2aa55-c67b-4915-8430-320bb163f464', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전')]
```  

아래는 다른 질의의 예시이다. 

```python
query = "소프트웨어를 외부로 제공하는 방법은?"
ensemble_result = ensemble_retriever.invoke(query)
print(ensemble_result)
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='운영체제\n\n정의: 운영체제는 컴퓨터의 하드웨어 자원을 관리하고 응용 프로그램과 사용자 간의 인터페이스를 제공하는 시스템 소프트웨어입니다.\n예시: Windows 11, macOS, Linux Ubuntu는 널리 사용되는 데스크톱 운영체제입니다.\n연관키워드: Windows, macOS, Linux, 시스템 소프트웨어\n\n방화벽'), 
# Document(id='89990330-8114-4d8a-adc1-3915b08e767b', metadata={'source': './computer-keywords.txt'}, page_content='클라우드 컴퓨팅\n\n정의: 클라우드 컴퓨팅은 인터넷을 통해 서버, 스토리지, 데이터베이스, 소프트웨어 등의 컴퓨팅 리소스를 제공하는 서비스입니다.\n예시: AWS, Microsoft Azure, Google Cloud Platform은 기업들이 자체 서버 인프라 구축 없이 필요한 만큼 IT 자원을 사용할 수 있게 해줍니다.\n연관키워드: IaaS, PaaS, SaaS, 서버리스, 확장성\n\nAPI')]

bm_25_result = bm25_retriever.invoke(query)
print(bm_25_result)
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='운영체제\n\n정의: 운영체제는 컴퓨터의 하드웨어 자원을 관리하고 응용 프로그램과 사용자 간의 인터페이스를 제공하는 시스템 소프트웨어입니다.\n예시: Windows 11, macOS, Linux Ubuntu는 널리 사용되는 데스크톱 운영체제입니다.\n연관키워드: Windows, macOS, Linux, 시스템 소프트웨어\n\n방화벽')]

chroma_result = chroma_retriever.invoke(query)
print(chroma_result)
# [Document(id='89990330-8114-4d8a-adc1-3915b08e767b', metadata={'source': './computer-keywords.txt'}, page_content='클라우드 컴퓨팅\n\n정의: 클라우드 컴퓨팅은 인터넷을 통해 서버, 스토리지, 데이터베이스, 소프트웨어 등의 컴퓨팅 리소스를 제공하는 서비스입니다.\n예시: AWS, Microsoft Azure, Google Cloud Platform은 기업들이 자체 서버 인프라 구축 없이 필요한 만큼 IT 자원을 사용할 수 있게 해줍니다.\n연관키워드: IaaS, PaaS, SaaS, 서버리스, 확장성\n\nAPI')]
```

아래와 같이 `ConfigurableField` 를 사용하면 런타임에 `EnsembleRetriever` 의 `weights` 를 동적으로 변경할 수 있다.  


```python
from langchain_core.runnables import ConfigurableField

ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, chroma_retriever],
).configurable_fields(
    weights=ConfigurableField(
        id="ensemble_weights",
        name="Ensemble Weights",
        description="Weights for Ensemble Retriever",
    )
)

config = {"configurable" : {"ensemble_weights" : [1, 0]}}
query = "서버 인증 방법"

docs = ensemble_retriever.invoke(query, config=config)
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='로드 밸런서\n\n정의: 로드 밸런서는 여러 서버 간에 네트워크 트래픽을 분산시켜 성능과 가용성을 향상시키는 장치 또는 소프트웨어입니다.  \n예시: Nginx는 웹 서버와 애플리케이션 서버 사이의 로드 밸런서로 자주 사용됩니다.  \n연관키워드: 트래픽 분산, 서버 관리, 성능 최적화, 고가용성\n\nCI/CD'),
# Document(id='fcb059ef-db76-47ef-9106-cba4a8712d27', metadata={'source': './computer-keywords.txt'}, page_content='SSH\n\n정의: SSH(Secure Shell)는 네트워크를 통해 장치를 안전하게 관리하거나 명령어를 실행할 수 있도록 하는 암호화 프로토콜입니다.  \n예시: 서버 관리자는 SSH를 사용하여 원격 서버에 연결하여 작업을 수행합니다.  \n연관키워드: 원격 접속, 보안 프로토콜, 암호화, 터미널\n\nCDN')]
``` 

가중치를 런타임에 변경하면, 가중치가 변경된 설정으로 최종 검색 결과의 순서가 변경되는 것을 확인할 수 있다. 

```python
config = {"configurable" : {"ensemble_weights" : [0, 1]}}
query = "서버 인증 방법"

docs = ensemble_retriever.invoke(query, config=config)
# [Document(id='fcb059ef-db76-47ef-9106-cba4a8712d27', metadata={'source': './computer-keywords.txt'}, page_content='SSH\n\n정의: SSH(Secure Shell)는 네트워크를 통해 장치를 안전하게 관리하거나 명령어를 실행할 수 있도록 하는 암호화 프로토콜입니다.  \n예시: 서버 관리자는 SSH를 사용하여 원격 서버에 연결하여 작업을 수행합니다.  \n연관키워드: 원격 접속, 보안 프로토콜, 암호화, 터미널\n\nCDN'),
# Document(metadata={'source': './computer-keywords.txt'}, page_content='로드 밸런서\n\n정의: 로드 밸런서는 여러 서버 간에 네트워크 트래픽을 분산시켜 성능과 가용성을 향상시키는 장치 또는 소프트웨어입니다.  \n예시: Nginx는 웹 서버와 애플리케이션 서버 사이의 로드 밸런서로 자주 사용됩니다.  \n연관키워드: 트래픽 분산, 서버 관리, 성능 최적화, 고가용성\n\nCI/CD')]
```  



### ParentDocumentRetriever
`ParentDocumentRetriever` 는 문서를 작은 조각으로 나누면서도 원본 문서의 맥락을 유지해 두 가지 요구 사항을 균형있게 해결하는 검색기이다. 

문서를 분할할 때 아래와 같은 내용을 고려해 균형을 고려해야 한다. 
- 작은 조각으로 문서를 나누는 경우 임베딩이 의미를 가장 정확하게 반영할 수 있다. 하지만 문서의 맥락은 유지되기가 힘들다. 
- 큰 조각으로 문서를 나누는 경우 문서의 전체적인 맥락은 유지될 수 있지만, 임베딩이 의미를 정확하게 파악하기 힘들다. 

위와 같은 두 관점의 균형을 맞추기 위한 것이 바로 `ParentDocumentRetriever` 이다. 
문서를 작은 조각으로 나누고, 검색을 진행할 때 먼저 작은 조각들을 찾는다. 
그리고 각 작은 조각들이 속한 원본 문서(더 큰 조각)를 통해 전체적인 맥락을 유지할수 있도록 한다. 

여기서 `Parent Document` 즉 부모 문서란 작은 조각이 나누어진 우너본 문서를 의미한다. 
이는 문서 전체일 수도 있고, 더 큰 문서의 조각일 수도 있다.  

이렇게 `ParentDocumentRetriever` 는 문서 검색의 효율을 높이기 위해 문서간 계층 구조를 활용하는 방법이다. 
관련성이 높은 문서를 빠르고 좀 더 정확하게 찾을 수 있고, 검색 결과에 대한 맥락도 함께 유지할 수 있다.  

`ParentDocumentRetriever` 는 검색시 사용하는 작은 조각을 위한 `child_splitter` 와 검색 결과에 사용하는 큰 조각을 위한 `parent_splitter` 설정이 필요하다. 
가장 먼저 예시에서는 `parent_splitter` 는 사용하지 않고, 
동작을 확인하기 위해 문서 검색은 `child_splitter` 로 진행하지만 결과는 전체 문서를 부모 문서로 사용해 반환하도록 하는 예시를 먼저 살펴본다.  

```python
from langchain.storage import InMemoryStore
from langchain.retrievers.parent_document_retriever import ParentDocumentRetriever

# 전체 문서
docs = []
docs.extend(computerKeywordLoader.load())

# 작은 문서
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200)

full_vectorstore = Chroma(
    collection_name="full_docs",
    embedding_function=hf_embeddings
)

# 검색기에서 사용할 부모 저장소
full_store = InMemoryStore()

# 검색시 생성
full_parent_retriever = ParentDocumentRetriever(
    vectorstore=full_vectorstore,
    docstore=full_store,
    child_splitter=child_splitter
)

# 검색기에 문서 추가해 검색기 구성
full_parent_retriever.add_documents(docs, ids=None, add_to_docstore=True)

# 부모 저장소에 생성된 문서 수 확인(전체 문서 내용을 부모로 사용하기 때문에 1개의 키만 존재)
list(full_store.yield_keys())
# ['960624d7-dc03-419c-83af-61dbb6dff20f']

# 검색기에서 질의에 대한 검색으로 사용하는 벡터 스토어는 작은 청크를 사용
sub_docs = full_vectorstore.similarity_search("사람처럼 학습하고 추론하는 시스템에서 높은 가용량을 보장하는 방법은 ?")
# [Document(id='cb1773e1-2298-4a0c-be42-3ba7446e1778', metadata={'doc_id': '960624d7-dc03-419c-83af-61dbb6dff20f', 'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(id='04a4ea6e-8ceb-466d-abf4-4ac3928747b9', metadata={'doc_id': '960624d7-dc03-419c-83af-61dbb6dff20f', 'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='1a4c92f6-601e-41ff-a18a-5e44723ec992', metadata={'doc_id': '960624d7-dc03-419c-83af-61dbb6dff20f', 'source': './computer-keywords.txt'}, page_content='백업\n\n정의: 백업은 데이터 손실에 대비하여 데이터를 복사하고 저장하는 과정입니다.  \n예시: 클라우드 백업 솔루션은 중요한 데이터를 안전하게 저장하고 복구할 수 있는 방법을 제공합니다.  \n연관키워드: 데이터 보호, 복구, 스냅샷, 클라우드'),
#  Document(id='68f68660-59b9-43fa-9b62-4cffdb079612', metadata={'doc_id': '960624d7-dc03-419c-83af-61dbb6dff20f', 'source': './computer-keywords.txt'}, page_content='클러스터링\n\n정의: 클러스터링은 여러 컴퓨터 또는 서버를 결합하여 하나의 시스템처럼 작동하도록 만드는 기술입니다.  \n예시: Kubernetes는 컨테이너를 클러스터로 관리하여 애플리케이션의 확장성과 안정성을 제공합니다.  \n연관키워드: 서버 그룹, 분산 시스템, 고가용성, 확장성\n\n데이터 암호화')]

# 실제 검색기의 검색 결과는 부모 청크 즉 전체 문서를 반환
search_docs = full_parent_retriever.get_relevant_documents("사람처럼 학습하고 추론하는 시스템에서 높은 가용량을 보장하는 방법은 ?")
len(search_docs[0].page_content)
# 6322
```  

부모 문서를 전체 문서로 하는 경우 어떠한 질의를 하든 전체 문서가 반환된다는 것을 확인했다. 
이제 본격적으로 작은 청크와 큰 청크 개념을 사용해 구성하고 테스트하면 아래와 같다. 

```python
# 부모 문서(큰 청크)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1000)
# 자식 문서(작은 청크)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200)

parent_vectorstore = Chroma(
    collection_name="parent_docs",
    embedding_function=hf_embeddings
)

parent_store = InMemoryStore()

parent_retriever = ParentDocumentRetriever(
    vectorstore=parent_vectorstore,
    docstore=parent_store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter
)

# 청크가 부모, 자식 문서가 생성될 수 있도록 검색기에 문서 추가
parent_retriever.add_documents(docs)
# 생성된 문서 수 확인(큰 청크로 전체 문서를 나눈 수)
len(list(parent_store.yield_keys()))
# 9

# 검색기에서 질의에 대한 검색으로 사용하는 벡터 스토어는 작은 청크를 사용
sub_docs = parent_vectorstore.similarity_search("사람처럼 학습하고 추론하는 시스템에서 높은 가용량을 보장하는 방법은 ?")
# [Document(id='272a1832-1154-407c-8bf1-80828e947d00', metadata={'doc_id': '5c7e9aaa-80b6-48de-a8ba-f1426b1e1821', 'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(id='d70ab21c-e0be-49b9-932d-8f366013329c', metadata={'doc_id': 'fb9e962d-e61e-4fdb-a640-f04b39dada36', 'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='97cc4e57-409c-481c-b3df-39f64e74a03c', metadata={'doc_id': 'c7bb8aef-c816-4c11-8f9b-a4825a7f9348', 'source': './computer-keywords.txt'}, page_content='백업\n\n정의: 백업은 데이터 손실에 대비하여 데이터를 복사하고 저장하는 과정입니다.  \n예시: 클라우드 백업 솔루션은 중요한 데이터를 안전하게 저장하고 복구할 수 있는 방법을 제공합니다.  \n연관키워드: 데이터 보호, 복구, 스냅샷, 클라우드'),
#  Document(id='6c08026e-a43d-4276-815c-cb41861b14b7', metadata={'doc_id': 'c7bb8aef-c816-4c11-8f9b-a4825a7f9348', 'source': './computer-keywords.txt'}, page_content='클러스터링\n\n정의: 클러스터링은 여러 컴퓨터 또는 서버를 결합하여 하나의 시스템처럼 작동하도록 만드는 기술입니다.  \n예시: Kubernetes는 컨테이너를 클러스터로 관리하여 애플리케이션의 확장성과 안정성을 제공합니다.  \n연관키워드: 서버 그룹, 분산 시스템, 고가용성, 확장성\n\n데이터 암호화')]


# 실제 검색기의 검색 결과는 부모 문서를 반환
search_docs = parent_retriever.invoke("사람처럼 학습하고 추론하는 시스템에서 높은 가용량을 보장하는 방법은 ?")
# [Document(metadata={'source': './computer-keywords.txt'}, page_content='사이버 보안\n\n정의: 사이버 보안은 컴퓨터 시스템, 네트워크, 데이터를, 무단 접근과 공격으로부터 보호하는 기술, 프로세스 및 관행입니다.\n예시: 안티바이러스 소프트웨어, 암호화, 다중 인증은 모두 사이버 보안을 강화하는 방법입니다.\n연관키워드: 해킹, 멀웨어, 피싱, 암호화, 취약점\n\nIoT\n\n정의: IoT(Internet of Things)는 인터넷을 통해 데이터를 수집하고 교환할 수 있는 센서와 소프트웨어가 내장된 물리적 장치들의 네트워크입니다.\n예시: 스마트 홈 시스템은 조명, 온도 조절 장치, 보안 카메라 등을 인터넷에 연결하여 원격으로 제어할 수 있게 합니다.\n연관키워드: 스마트 기기, 센서, M2M, 연결성, 자동화\n\n인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치\n\n정의: 네트워크 스위치는 여러 장치들을 네트워크에 연결하고 데이터 패킷을 목적지로 효율적으로 전달하는 장치입니다.  \n예시: Cisco Catalyst 시리즈는 기업 네트워크 환경에서 사용되는 고성능 네트워크 스위치입니다.  \n연관키워드: 네트워크, 데이터 패킷, LAN, 포트, 트래픽 관리\n\nDNS 캐싱\n\n정의: DNS 캐싱은 도메인 이름과 IP 주소 매핑 데이터를 로컬에 저장하여 DNS 조회 시간을 단축하는 기술입니다.  \n예시: 웹 브라우저가 자주 방문한 웹사이트의 DNS 정보를 캐싱하여 연결 속도를 향상시킵니다.  \n연관키워드: DNS, 캐시 메모리, 네임서버, 성능 최적화\n\nRAID'), Document(metadata={'source': './computer-keywords.txt'}, page_content='빅데이터\n\n정의: 빅데이터는 기존 데이터베이스 도구로 처리하기 어려운 대량의 정형 및 비정형 데이터를 의미합니다.\n예시: 소셜 미디어 플랫폼은 매일 페타바이트 규모의 사용자 활동 데이터를 분석하여 맞춤 콘텐츠를 제공합니다.\n연관키워드: 하둡, 스파크, 데이터 마이닝, 분석, 볼륨\n\n머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화\n\n정의: 가상화는 물리적 컴퓨터 자원을 여러 가상 환경으로 나누어 효율적으로 사용할 수 있게 하는 기술입니다.\n예시: VMware, VirtualBox와 같은 소프트웨어는 하나의 물리적 서버에서 여러 운영체제를 동시에 실행할 수 있게 합니다.\n연관키워드: 하이퍼바이저, VM, 컨테이너, 리소스 최적화\n\n블록체인\n\n정의: 블록체인은 분산된 컴퓨터 네트워크에서 데이터 블록이 암호화 기술로 연결된 디지털 장부 시스템입니다.\n예시: 비트코인은 블록체인 기술을 활용하여 중앙 은행 없이 안전한 금융 거래를 가능하게 합니다.\n연관키워드: 암호화폐, 분산원장, 스마트 계약, 합의 알고리즘\n\n알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'), Document(metadata={'source': './computer-keywords.txt'}, page_content='클러스터링\n\n정의: 클러스터링은 여러 컴퓨터 또는 서버를 결합하여 하나의 시스템처럼 작동하도록 만드는 기술입니다.  \n예시: Kubernetes는 컨테이너를 클러스터로 관리하여 애플리케이션의 확장성과 안정성을 제공합니다.  \n연관키워드: 서버 그룹, 분산 시스템, 고가용성, 확장성\n\n데이터 암호화\n\n정의: 데이터 암호화는 데이터를 보호하기 위해 특정 알고리즘을 사용하여 데이터를 변환하는 기술입니다.  \n예시: AES-256 암호화는 금융 데이터와 같은 민감한 정보를 보호하는 데 자주 사용됩니다.  \n연관키워드: 보안, 암호 알고리즘, 키 관리, 데이터 보호\n\n백업\n\n정의: 백업은 데이터 손실에 대비하여 데이터를 복사하고 저장하는 과정입니다.  \n예시: 클라우드 백업 솔루션은 중요한 데이터를 안전하게 저장하고 복구할 수 있는 방법을 제공합니다.  \n연관키워드: 데이터 보호, 복구, 스냅샷, 클라우드')]

len(search_docs[0].page_content)
# 888
```  

