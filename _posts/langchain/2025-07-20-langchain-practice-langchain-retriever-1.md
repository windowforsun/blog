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

