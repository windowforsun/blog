--- 
layout: single
classes: wide
title: "[LangChain] LangChain Reranker"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain 에서 정보의 검색이나 생성형 인공지능 파이프라인에서 여러 후보 결과를 다시 평가해서 최적의 순서로 재졍렬하는 Reranker 에 대해 알아보자.'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
    - Reranker
    - RAG
    - Retriever
    - Cross-Encoder
    - Bi-Encoder
    - Cross Encoder Reranker
    - Cohere Reranker
    - FlashRank Reranker
    - Jina Reranker
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


대표적인 모델로는 `BERT`, `RoBERTa`, `DeBERTa` 등이 있다.



`Retriever` 와 `Reranker` 의 특징과 차이점을 비교 정리하면 아래와 같다. 

| 구분         | Retriever                                             | Reranker                                                 |
|--------------|------------------------------------------------------------|----------------------------------------------------------|
| 역할         | 검색 대상에서 관련 후보(문서 등)를 1차적으로 찾아줌              | 1차 후보를 받아, 가장 적합한 순서로 재정렬함                               |
| 주요 목적    | 빠르고 넓게 관련 정보를 효율적으로 수집                         | 후보 중에서 최고로 적합한 결과를 최상위에 올림                               |
| 사용 시점    | 파이프라인의 "초기" 단계(후보 생성)                             | 파이프라인의 "후속" 단계(후보 재정렬)                                   |
| 대표 방식    | 벡터 검색(Vector Search), 키워드 검색 등                        | LLM 기반 평가, Cross-Encoder, 점수 기반 재정렬 등                    |
| 입력         | 쿼리(질문)                                                 | 쿼리(질문) + Retriever가 찾은 후보 리스트                            |
| 출력         | 관련 후보 리스트(보통 5~50개)                                   | 재정렬된 후보 리스트(보통 상위 N개, 더 정밀하게 선별) + 점수                    |
| 속도         | 빠름(효율성 중시)                                            | 상대적으로 느림(정확성 중시, 모델 평가 필요)                               |
| 품질         | "관련성 있는 후보"를 넓게 찾는 데 강점                           | "정확히 가장 적합한 결과"를 1등에 올리는 데 강점                            |
| 활용 예시    | RAG, 챗봇, 추천 시스템 등에서 1차 후보 추출                      | RAG, 챗봇, 답변 랭킹, 검색 결과 최적화 등                              |
| 대표 라이브러리 | FAISS, ElasticSearch, Milvus 등                          | Cohere Rerank, OpenAI Rerank, HuggingFace CrossEncoder 등 |


`LangChain` 에서 `Reranker` 의 구현은 `ContextualCompressionRetriever` 를 사용해 구현된다. 
`ContextualCompressionRetriever` 는 기존 검색기 위에 추가 레이어로 작동하므로, 
원본 검색 메커니즘을 변경하지 않고 재순위화 작업을 적용할 수 있기 때문이다.  

`Reranker` 에서 사용할 문서를 로드하고 기본 벡터 저장소를 생성하는 코드는 아래와 같다. 

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

네트워크 스위치

정의: 네트워크 스위치는 여러 장치들을 네트워크에 연결하고 데이터 패킷을 목적지로 효율적으로 전달하는 장치입니다.  
예시: Cisco Catalyst 시리즈는 기업 네트워크 환경에서 사용되는 고성능 네트워크 스위치입니다.  
연관키워드: 네트워크, 데이터 패킷, LAN, 포트, 트래픽 관리

DNS 캐싱

정의: DNS 캐싱은 도메인 이름과 IP 주소 매핑 데이터를 로컬에 저장하여 DNS 조회 시간을 단축하는 기술입니다.  
예시: 웹 브라우저가 자주 방문한 웹사이트의 DNS 정보를 캐싱하여 연결 속도를 향상시킵니다.  
연관키워드: DNS, 캐시 메모리, 네임서버, 성능 최적화

RAID

정의: RAID(Redundant Array of Independent Disks)는 여러 하드 드라이브를 하나의 논리적 유닛으로 결합하여 성능 또는 데이터 보호를 향상시키는 기술입니다.  
예시: RAID 1은 데이터를 두 개의 드라이브에 동일하게 복제하여 데이터 손실 방지에 사용됩니다.  
연관키워드: 데이터 보호, 스토리지, 미러링, 스트라이핑

NAT

정의: NAT(Network Address Translation)는 사설 IP 주소를 공용 IP 주소로 변환하여 인터넷 트래픽을 관리하는 네트워크 기술입니다.  
예시: 가정용 라우터는 NAT를 사용하여 여러 장치가 하나의 공용 IP로 인터넷에 연결할 수 있게 합니다.  
연관키워드: 라우터, IP 주소, 네트워크 보안, 트래픽 변환

컨테이너

정의: 컨테이너는 애플리케이션과 그 실행 환경을 격리하여 경량으로 배포하고 실행할 수 있게 하는 가상화 기술입니다.  
예시: Docker 컨테이너를 사용하여 애플리케이션을 플랫폼 간에 쉽게 배포할 수 있습니다.  
연관키워드: 가상화, Docker, Kubernetes, 경량화

HTTP/2

정의: HTTP/2는 HTTP 프로토콜의 차세대 버전으로, 데이터 전송 속도와 효율성을 향상시키기 위해 멀티플렉싱과 헤더 압축을 제공합니다.  
예시: HTTP/2를 지원하는 웹 서버는 사용자가 웹 페이지를 더 빠르게 로드할 수 있도록 도와줍니다.  
연관키워드: 프로토콜, 멀티플렉싱, 헤더 압축, HTTPS

캐시 메모리

정의: 캐시 메모리는 CPU와 메인 메모리 간 데이터 전송 속도를 높이기 위해 자주 사용하는 데이터를 임시로 저장하는 고속 메모리입니다.  
예시: L1, L2, L3 캐시는 CPU 내부 또는 가까운 위치에 있는 캐시 계층입니다.  
연관키워드: 고속 메모리, CPU, 데이터 접근, 성능 최적화

DHCP

정의: DHCP(Dynamic Host Configuration Protocol)는 네트워크 장치에 자동으로 IP 주소를 할당하는 프로토콜입니다.  
예시: Wi-Fi 라우터는 DHCP를 사용하여 연결된 모든 장치에 고유한 IP 주소를 자동으로 할당합니다.  
연관키워드: IP 주소, 프로토콜, 네트워크 설정, 자동화

SSH

정의: SSH(Secure Shell)는 네트워크를 통해 장치를 안전하게 관리하거나 명령어를 실행할 수 있도록 하는 암호화 프로토콜입니다.  
예시: 서버 관리자는 SSH를 사용하여 원격 서버에 연결하여 작업을 수행합니다.  
연관키워드: 원격 접속, 보안 프로토콜, 암호화, 터미널

CDN

정의: CDN(Content Delivery Network)은 전 세계에 분산된 서버 네트워크를 통해 콘텐츠를 사용자에게 빠르게 제공하는 기술입니다.  
예시: Cloudflare와 같은 CDN은 웹사이트 로딩 시간을 단축하고 트래픽 부하를 분산시킵니다.  
연관키워드: 네트워크, 콘텐츠 배포, 캐싱, 분산 시스템

웹 애플리케이션 방화벽

정의: 웹 애플리케이션 방화벽(WAF)은 웹 애플리케이션에 대한 악성 트래픽을 탐지하고 차단하는 보안 시스템입니다.  
예시: AWS WAF는 클라우드 기반 웹 애플리케이션을 보호하기 위해 사용됩니다.  
연관키워드: 보안, SQL 인젝션, 클라우드 보안, 공격 방지

로드 밸런서

정의: 로드 밸런서는 여러 서버 간에 네트워크 트래픽을 분산시켜 성능과 가용성을 향상시키는 장치 또는 소프트웨어입니다.  
예시: Nginx는 웹 서버와 애플리케이션 서버 사이의 로드 밸런서로 자주 사용됩니다.  
연관키워드: 트래픽 분산, 서버 관리, 성능 최적화, 고가용성

CI/CD

정의: CI/CD(Continuous Integration/Continuous Deployment)는 코드 변경 사항을 자동으로 통합하고 배포하는 소프트웨어 개발 방식입니다.  
예시: Jenkins, GitHub Actions는 CI/CD 프로세스를 자동화하는 도구입니다.  
연관키워드: 자동화, 배포, 개발 파이프라인, DevOps

클러스터링

정의: 클러스터링은 여러 컴퓨터 또는 서버를 결합하여 하나의 시스템처럼 작동하도록 만드는 기술입니다.  
예시: Kubernetes는 컨테이너를 클러스터로 관리하여 애플리케이션의 확장성과 안정성을 제공합니다.  
연관키워드: 서버 그룹, 분산 시스템, 고가용성, 확장성

데이터 암호화

정의: 데이터 암호화는 데이터를 보호하기 위해 특정 알고리즘을 사용하여 데이터를 변환하는 기술입니다.  
예시: AES-256 암호화는 금융 데이터와 같은 민감한 정보를 보호하는 데 자주 사용됩니다.  
연관키워드: 보안, 암호 알고리즘, 키 관리, 데이터 보호

백업

정의: 백업은 데이터 손실에 대비하여 데이터를 복사하고 저장하는 과정입니다.  
예시: 클라우드 백업 솔루션은 중요한 데이터를 안전하게 저장하고 복구할 수 있는 방법을 제공합니다.  
연관키워드: 데이터 보호, 복구, 스냅샷, 클라우드

DRAM

정의: DRAM(Dynamic Random Access Memory)은 각 비트를 커패시터에 저장하여 주기적으로 재충전이 필요한 휘발성 메모리의 일종입니다. 
예시: DDR4 DRAM은 현대 컴퓨터 시스템의 주요 메모리 구성 요소로 사용됩니다. 
연관키워드: RAM, 메모리, 휘발성, 리프레시, DDR

NVMe SSD

정의: NVMe(Non-Volatile Memory Express) SSD는 PCI Express 인터페이스를 통해 직접 연결되어 기존 SATA SSD보다 빠른 속도를 제공하는 저장 장치입니다. 
예시: Samsung 980 Pro NVMe SSD는 초당 7,000MB의 읽기 속도를 제공하여 대용량 파일 전송 속도를 획기적으로 향상시킵니다. 
연관키워드: SSD, 플래시 메모리, PCIe, 고속 저장장치, M.2

리눅스 커널

정의: 리눅스 커널은 리눅스 운영체제의 핵심 구성 요소로, 하드웨어와 소프트웨어 간의 통신을 관리하는 시스템 소프트웨어입니다. 
예시: 리눅스 커널 5.15 LTS 버전은 향상된 보안 기능과 하드웨어 지원을 제공합니다. 
연관키워드: 운영체제, 오픈소스, 시스템 호출, 드라이버, GNU

하드웨어 방화벽

정의: 하드웨어 방화벽은 네트워크 트래픽을 모니터링하고 필터링하기 위한 전용 물리적 장치로, 소프트웨어 방화벽보다 높은 보안성을 제공합니다. 
예시: Cisco ASA 5500 시리즈는 기업 네트워크 경계에 배치되어 외부 위협으로부터 내부 시스템을 보호하는 하드웨어 방화벽입니다. 
연관키워드: 방화벽, 네트워크 보안, UTM, 패킷 필터링, 침입 탐지

엣지 컴퓨팅

정의: 엣지 컴퓨팅은 데이터 생성 지점 가까이에서 처리를 수행하여 지연 시간을 줄이고 대역폭 사용을 최적화하는 분산형 컴퓨팅 패러다임입니다. 
예시: 스마트 공장에서는 엣지 컴퓨팅 기술을 활용해 장비 센서 데이터를 실시간으로 분석하여 즉각적인 의사결정을 내립니다. 
연관키워드: 클라우드 컴퓨팅, IoT, 분산 처리, 실시간 분석, 지연 최소화

GraphQL API

정의: GraphQL API는 클라이언트가 필요한 데이터만 정확하게 요청할 수 있도록 설계된 쿼리 언어 및 런타임을 사용하는 API입니다. 
예시: GitHub API v4는 GraphQL을 사용하여 개발자가 필요한 정보만 효율적으로 가져올 수 있게 합니다. 
연관키워드: API, REST, 쿼리 언어, 데이터 페칭, 스키마

데이터 웨어하우스

정의: 데이터 웨어하우스는 의사 결정과 분석을 위해 여러 소스의 데이터를 통합하고 저장하는 대규모 중앙 집중식 데이터 저장소입니다. 
예시: Amazon Redshift는 클라우드 기반 데이터 웨어하우스로 페타바이트 규모의 데이터를 저장하고 분석할 수 있습니다. 
연관키워드: 빅데이터, ETL, 비즈니스 인텔리전스, 데이터 마이닝, OLAP

전이 학습

정의: 전이 학습은 한 문제에서 학습된 지식을 관련된 다른 문제에 적용하는 머신러닝 기법입니다. 
예시: BERT 모델은 대규모 텍스트 데이터로 사전 학습된 후, 특정 자연어 처리 작업에 맞게 미세 조정됩니다. 
연관키워드: 머신러닝, 딥러닝, 사전학습, 미세조정, NLP

컨테이너 오케스트레이션

정의: 컨테이너 오케스트레이션은 컨테이너화된 애플리케이션의 배포, 확장, 관리, 네트워킹을 자동화하는 기술입니다. 
예시: Kubernetes는 자동 확장, 로드 밸런싱, 장애 복구 기능을 갖춘 인기 있는 컨테이너 오케스트레이션 플랫폼입니다. 
연관키워드: 가상화, Docker, 마이크로서비스, DevOps, 클라우드 네이티브

DDoS 방어

정의: DDoS 방어는 분산 서비스 거부 공격으로부터 네트워크나 서비스를 보호하기 위한 기술과 전략의 집합입니다. 
예시: Cloudflare의 DDoS 방어 솔루션은 악의적인 트래픽을 식별하고 필터링하여 웹사이트의 가용성을 유지합니다. 
연관키워드: 사이버 보안, 트래픽 필터링, 공격 완화, 네트워크 보안, 트래픽 분석

IPv6

정의: IPv6는 기존 IPv4의 주소 고갈 문제를 해결하기 위해 개발된 128비트 인터넷 프로토콜 버전입니다. 
예시: 2001:0db8:85a3:0000:0000:8a2e:0370:7334와 같은 IPv6 주소는 더 많은 장치를 인터넷에 연결할 수 있게 합니다. 
연관키워드: 인터넷 프로토콜, IP 주소, 네트워크, 라우팅, NAT

NoSQL 데이터베이스

정의: NoSQL 데이터베이스는 관계형 데이터베이스와 달리 유연한 스키마를 가지며 대용량 분산 데이터를 효율적으로 처리하는 데이터베이스입니다. 
예시: MongoDB, Cassandra, Redis는 각각 문서 저장소, 열 기반 저장소, 키-값 저장소 형태의 NoSQL 데이터베이스입니다. 
연관키워드: 데이터베이스, 스키마리스, 확장성, 문서 저장소, 빅데이터

PWA

정의: PWA(Progressive Web App)는 웹 기술로 구축되었지만 네이티브 앱과 유사한 사용자 경험을 제공하는 웹 애플리케이션입니다. 
예시: Twitter Lite는 PWA로, 오프라인에서도 작동하며 모바일 데이터 사용량을 줄이면서 네이티브 앱과 유사한 경험을 제공합니다. 
연관키워드: 웹 브라우저, 서비스 워커, 오프라인 기능, 반응형 디자인, 웹 앱

양자 컴퓨팅

정의: 양자 컴퓨팅은 양자역학의 원리를 활용하여 특정 유형의 문제를 기존 컴퓨터보다 더 효율적으로 해결하는 컴퓨팅 형태입니다. 
예시: IBM Quantum은 클라우드를 통해 양자 컴퓨터에 접근할 수 있는 서비스를 제공하여 연구자들이 양자 알고리즘을 개발할 수 있게 합니다. 
연관키워드: 큐비트, 양자 얽힘, 양자 중첩, 양자 게이트, 양자 우위

TLS/SSL

정의: TLS/SSL은 네트워크를 통한 데이터 전송 시 암호화를 통해 보안을 제공하는 프로토콜입니다. 
예시: 온라인 뱅킹 웹사이트는 TLS 1.3을 사용하여 사용자의 금융 정보를 보호합니다. 
연관키워드: 암호화, 인증서, HTTPS, 보안 통신, 데이터 무결성
```  

```python
from langchain_huggingface.embeddings import HuggingFaceEndpointEmbeddings
from langchain_huggingface.embeddings import HuggingFaceEmbeddings
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma

# 벡터 저장소에서 사용할 입베딩 함수
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

# 텍스트 문서 로드
text_splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=100)
computerKeywordLoader = TextLoader("./computer-keywords.txt")
split_computer_keywords = computerKeywordLoader.load_and_split(text_splitter)

# 벡터 저장소 생성
memory_db = Chroma.from_documents(documents=split_computer_keywords, embedding=hf_embeddings, collection_name="computer_keywords_db")
vector_retriever = memory_db.as_retriever(search_kwargs={"k":10})
vector_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(id='0c3bb309-a2af-4e14-b71a-8cd3249dc02e', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(id='18de0a53-bcb1-4cf6-af86-8bd1e0c03238', metadata={'source': './computer-keywords.txt'}, page_content='전이 학습\n\n정의: 전이 학습은 한 문제에서 학습된 지식을 관련된 다른 문제에 적용하는 머신러닝 기법입니다. \n예시: BERT 모델은 대규모 텍스트 데이터로 사전 학습된 후, 특정 자연어 처리 작업에 맞게 미세 조정됩니다. \n연관키워드: 머신러닝, 딥러닝, 사전학습, 미세조정, NLP\n\n컨테이너 오케스트레이션'),
#  Document(id='ac0d7145-6ab9-48d3-991d-503e5c87b38e', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='217d1477-cd0e-454f-a629-0cb9bda6ac4d', metadata={'source': './computer-keywords.txt'}, page_content='양자 컴퓨팅\n\n정의: 양자 컴퓨팅은 양자역학의 원리를 활용하여 특정 유형의 문제를 기존 컴퓨터보다 더 효율적으로 해결하는 컴퓨팅 형태입니다. \n예시: IBM Quantum은 클라우드를 통해 양자 컴퓨터에 접근할 수 있는 서비스를 제공하여 연구자들이 양자 알고리즘을 개발할 수 있게 합니다. \n연관키워드: 큐비트, 양자 얽힘, 양자 중첩, 양자 게이트, 양자 우위\n\nTLS/SSL'),
#  Document(id='aec74c0c-62f1-4b4b-990f-dc1d8dae7647', metadata={'source': './computer-keywords.txt'}, page_content='엣지 컴퓨팅\n\n정의: 엣지 컴퓨팅은 데이터 생성 지점 가까이에서 처리를 수행하여 지연 시간을 줄이고 대역폭 사용을 최적화하는 분산형 컴퓨팅 패러다임입니다. \n예시: 스마트 공장에서는 엣지 컴퓨팅 기술을 활용해 장비 센서 데이터를 실시간으로 분석하여 즉각적인 의사결정을 내립니다. \n연관키워드: 클라우드 컴퓨팅, IoT, 분산 처리, 실시간 분석, 지연 최소화\n\nGraphQL API'),
#  Document(id='2ea1703a-0e4e-40bd-8f03-fe7497219607', metadata={'source': './computer-keywords.txt'}, page_content='알고리즘\n\n정의: 알고리즘은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다.\n예시: 구글의 검색 엔진은 PageRank 알고리즘을 사용하여 웹페이지의 관련성과 중요도를 평가합니다.\n연관키워드: 데이터 구조, 복잡도, 정렬, 검색, 최적화\n\nDNS'),
#  Document(id='f2eae6f0-e54a-43b1-bc43-2bf82dd8e9e5', metadata={'source': './computer-keywords.txt'}, page_content='GPU\n\n정의: GPU(Graphics Processing Unit)는 컴퓨터의 그래픽 렌더링과 복잡한 병렬 처리를 전문적으로 수행하는 프로세서입니다.\n예시: NVIDIA GeForce RTX 3080은 게임 및 인공지능 학습에 활용되는 고성능 GPU입니다.\n연관키워드: 그래픽 카드, 렌더링, CUDA, 병렬 처리\n\nSSD'),
#  Document(id='d045bed5-6752-4178-8ec6-897da2c22c76', metadata={'source': './computer-keywords.txt'}, page_content='IoT\n\n정의: IoT(Internet of Things)는 인터넷을 통해 데이터를 수집하고 교환할 수 있는 센서와 소프트웨어가 내장된 물리적 장치들의 네트워크입니다.\n예시: 스마트 홈 시스템은 조명, 온도 조절 장치, 보안 카메라 등을 인터넷에 연결하여 원격으로 제어할 수 있게 합니다.\n연관키워드: 스마트 기기, 센서, M2M, 연결성, 자동화\n\n인공지능'),
#  Document(id='86d06797-dfce-4c09-94ef-fcba2d754832', metadata={'source': './computer-keywords.txt'}, page_content='컨테이너 오케스트레이션\n\n정의: 컨테이너 오케스트레이션은 컨테이너화된 애플리케이션의 배포, 확장, 관리, 네트워킹을 자동화하는 기술입니다. \n예시: Kubernetes는 자동 확장, 로드 밸런싱, 장애 복구 기능을 갖춘 인기 있는 컨테이너 오케스트레이션 플랫폼입니다. \n연관키워드: 가상화, Docker, 마이크로서비스, DevOps, 클라우드 네이티브\n\nDDoS 방어'),
#  Document(id='1505f316-8659-4958-ac70-1e3a293c89d8', metadata={'source': './computer-keywords.txt'}, page_content='빅데이터\n\n정의: 빅데이터는 기존 데이터베이스 도구로 처리하기 어려운 대량의 정형 및 비정형 데이터를 의미합니다.\n예시: 소셜 미디어 플랫폼은 매일 페타바이트 규모의 사용자 활동 데이터를 분석하여 맞춤 콘텐츠를 제공합니다.\n연관키워드: 하둡, 스파크, 데이터 마이닝, 분석, 볼륨\n\n머신러닝')]
```  

벡터 저장소는 `Reranker` 테스트를 위해 10개의 문서를 가져온다.  


### Cross Encoder Reranker
`Cross Encoder Reranker` 는 앞서 설명한 것과 같이, 
쿼리와 검색된 문서를 동시에 인코딩하여 관련성 점수를 계산하는 모델이다. 
이는 검색 파이프라인에서 초기 검색 결과(`retriever`)의 순위를 다시 매기는데 사용할 수 있다. 



예제에서는 `HuggingFace` 에서 다국어 지원 `cross encoder` 모델인 [BAAI/bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3)
을 사용해 `CrossEncoderReranker` 를 구현한다.  

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

model = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-v2-m3")

cross_encoder_compressor = CrossEncoderReranker(model=model, top_n=3)

cross_encoder_compression_retriever = ContextualCompressionRetriever(
    base_compressor=cross_encoder_compressor, base_retriever=vector_retriever
)

result = cross_encoder_compression_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(id='41e7dfa0-24b4-4469-b9fb-33e50046de8f', metadata={'source': './computer-keywords.txt'}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(id='f4cb593b-211d-4a50-8e39-cd04fde5abfc', metadata={'source': './computer-keywords.txt'}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(id='8afb5185-2133-4c5a-a339-297cd5503574', metadata={'source': './computer-keywords.txt'}, page_content='전이 학습\n\n정의: 전이 학습은 한 문제에서 학습된 지식을 관련된 다른 문제에 적용하는 머신러닝 기법입니다. \n예시: BERT 모델은 대규모 텍스트 데이터로 사전 학습된 후, 특정 자연어 처리 작업에 맞게 미세 조정됩니다. \n연관키워드: 머신러닝, 딥러닝, 사전학습, 미세조정, NLP\n\n컨테이너 오케스트레이션')]
```  


### Cohere Reranker
`Cohere Reranker` 는 `Cohere` API를 사용하여 쿼리와 문서의 관련성을 평가하는 모델이다. 
이 또한 `Cross Encoder` 방식으로 동작한다.  

`Cohere API Key` 발급이 필요한데 [여기](https://dashboard.cohere.com/)
에서 발급 받을 수 있다. 


```python
from langchain.retrievers.contextual_compression import ContextualCompressionRetriever
from langchain_cohere import CohereRerank
import os

os.environ["CO_API_KEY"] = "api key"
cohere_compressor = CohereRerank(model="rerank-multilingual-v3.0")
cohere_compression_retriever = ContextualCompressionRetriever(
    base_compressor=cohere_compressor, base_retriever=vector_retriever
)

result = cohere_compression_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(metadata={'source': './computer-keywords.txt', 'relevance_score': 0.9828892}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(metadata={'source': './computer-keywords.txt', 'relevance_score': 0.4737637}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(metadata={'source': './computer-keywords.txt', 'relevance_score': 0.0130707845}, page_content='GPU\n\n정의: GPU(Graphics Processing Unit)는 컴퓨터의 그래픽 렌더링과 복잡한 병렬 처리를 전문적으로 수행하는 프로세서입니다.\n예시: NVIDIA GeForce RTX 3080은 게임 및 인공지능 학습에 활용되는 고성능 GPU입니다.\n연관키워드: 그래픽 카드, 렌더링, CUDA, 병렬 처리\n\nSSD')]
```  


### Jina Reranker
`Jina Reranker` 는 `Jina AI` 에서 제공하는 오픈소스 `Reranking` 라이브러리이다. 
`Cross Encoder` 기반 사전학습 된 모델을 사용해 쿼리와 각 후보 문서 쌍의 관련성 점수를 계산한다.  

`Jina API Key` 발급이 필요한데 [여기](https://jina.ai/)
에서 발급 받을 수 있다.  

```python
from ast import mod
from langchain.retrievers import ContextualCompressionRetriever
from langchain_community.document_compressors import JinaRerank

os.environ["JINA_API_KEY"] = "api key"
jina_compressor = JinaRerank(model="jina-reranker-v2-base-multilingual", top_n=3)
jina_compression_retriever = ContextualCompressionRetriever(
  base_compressor=jina_compressor, base_retriever=vector_retriever
)

result = jina_compression_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(metadata={'source': './computer-keywords.txt', 'relevance_score': 0.4532618522644043}, page_content='인공지능\n\n정의: 인공지능(AI)은 인간의 지능을 모방하여 학습, 추론, 문제 해결, 자연어 처리 등을 수행할 수 있는 시스템과 기계를 만드는 과학입니다.\n예시: 음성 비서인 시리, 알렉사, 구글 어시스턴트는 AI 기술을 활용하여 자연어로 사용자와 상호작용합니다.\n연관키워드: 머신러닝, 딥러닝, 신경망, 자연어 처리, 컴퓨터 비전\n\n네트워크 스위치'),
#  Document(metadata={'source': './computer-keywords.txt', 'relevance_score': 0.30239108204841614}, page_content='머신러닝\n\n정의: 머신러닝은 컴퓨터가 명시적 프로그래밍 없이 데이터로부터 학습하고 예측할 수 있게 하는 인공지능의 한 분야입니다.\n예시: 넷플릭스의 콘텐츠 추천 시스템은 사용자의 시청 이력을 기반으로 선호할 만한 영화와 시리즈를 제안합니다.\n연관키워드: 인공지능, 딥러닝, 신경망, 데이터 모델링\n\n가상화'),
#  Document(metadata={'source': './computer-keywords.txt', 'relevance_score': 0.13568955659866333}, page_content='전이 학습\n\n정의: 전이 학습은 한 문제에서 학습된 지식을 관련된 다른 문제에 적용하는 머신러닝 기법입니다. \n예시: BERT 모델은 대규모 텍스트 데이터로 사전 학습된 후, 특정 자연어 처리 작업에 맞게 미세 조정됩니다. \n연관키워드: 머신러닝, 딥러닝, 사전학습, 미세조정, NLP\n\n컨테이너 오케스트레이션')]
```  


### FlashRank Reranker
`FlashRank Reranker` 는 `FlashRank` 라이브러리를 사용하여 쿼리와 문서의 관련성을 평가하는 모델이다. 
가장 큰 특징은 별도의 외부 `API` 를 사용하지 않고 `Python` 라이브러리를 설치하는 방식으로 사용할수 있다는 점이다. 
그리고 다른 `Reranker` 들과 비교했을 때 빠른 성능을 제공할 수 있다. 
`BGE`, `MiniLM` 등 기반 리랭크 모델을 최적화해, `Cross-Encoder` 급 정확도를 로컬 `GPU/CPU` 에서 사용할 수 있도록 구현한 것이다. 

아래 명령으로 `flashrank` 라이브러리를 설치한다. 

```bash
pip install flashrank
```  

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import FlashrankRerank

# flashrank_compressor = FlashrankRerank(model="ms-marco-MultiBERT-L-12")
flashrank_compressor = FlashrankRerank(model="ms-marco-MiniLM-L-12-v2")
flashrank_compression_retriever = ContextualCompressionRetriever(
    base_compressor=flashrank_compressor, base_retriever=vector_retriever
)

result = flashrank_compression_retriever.invoke("사람처럼 학습하고 추론하는 시스템은?")
# [Document(metadata={'id': 8, 'relevance_score': np.float32(0.9995274), 'source': './computer-keywords.txt'}, page_content='컨테이너 오케스트레이션\n\n정의: 컨테이너 오케스트레이션은 컨테이너화된 애플리케이션의 배포, 확장, 관리, 네트워킹을 자동화하는 기술입니다. \n예시: Kubernetes는 자동 확장, 로드 밸런싱, 장애 복구 기능을 갖춘 인기 있는 컨테이너 오케스트레이션 플랫폼입니다. \n연관키워드: 가상화, Docker, 마이크로서비스, DevOps, 클라우드 네이티브\n\nDDoS 방어'),
#  Document(metadata={'id': 3, 'relevance_score': np.float32(0.99951947), 'source': './computer-keywords.txt'}, page_content='양자 컴퓨팅\n\n정의: 양자 컴퓨팅은 양자역학의 원리를 활용하여 특정 유형의 문제를 기존 컴퓨터보다 더 효율적으로 해결하는 컴퓨팅 형태입니다. \n예시: IBM Quantum은 클라우드를 통해 양자 컴퓨터에 접근할 수 있는 서비스를 제공하여 연구자들이 양자 알고리즘을 개발할 수 있게 합니다. \n연관키워드: 큐비트, 양자 얽힘, 양자 중첩, 양자 게이트, 양자 우위\n\nTLS/SSL'),
#  Document(metadata={'id': 7, 'relevance_score': np.float32(0.99951446), 'source': './computer-keywords.txt'}, page_content='IoT\n\n정의: IoT(Internet of Things)는 인터넷을 통해 데이터를 수집하고 교환할 수 있는 센서와 소프트웨어가 내장된 물리적 장치들의 네트워크입니다.\n예시: 스마트 홈 시스템은 조명, 온도 조절 장치, 보안 카메라 등을 인터넷에 연결하여 원격으로 제어할 수 있게 합니다.\n연관키워드: 스마트 기기, 센서, M2M, 연결성, 자동화\n\n인공지능')]
```  

### Compare Reranker

| 구분               | FlashRank Reranker                                                       | Cross Encoder Reranker        | Cohere Reranker                  | Jina Reranker                       |
|--------------------|--------------------------------------------------------------------------|-------------------------------|-----------------------------------|-------------------------------------|
| 정의/제공자    | 빠르고 정밀한 오픈소스 리랭커                           | Cross-Encoder 구조 모델        | Cohere사의 상용 Cross-Encoder API | Jina AI의 오픈소스 리랭커 라이브러리 |
| 동작 원리      | 쿼리-문서 쌍을 Cross-Encoder 구조로 평가하되, 속도 최적화                 | 쿼리-문서 쌍을 Cross-Encoder로 평가 | Cohere API에서 Cross-Encoder로 평가 | 다양한 모델로 쿼리-문서 쌍 평가     |
| 주요 특징      | Cross-Encoder급 정확도, 매우 빠름, CPU/GPU 지원, 로컬 설치                | 정확도 최고, 느림(후보 많으면 부담) | SaaS, 별도 인프라 불필요, 쉽고 빠름 | 오픈소스, 다양한 모델, 커스텀 가능   |
| 대표 모델      | BGE, MiniLM, Mistral 등                                                  | ms-marco-MiniLM, BERT 등      | cohere.rerank-english-v2 등       | jinaai/jina-reranker-v1-base-en 등  |
| 사용 방식      | pip설치 후 Python에서 직접 사용, 서버리스도 가능                          | 직접 모델 로드/추론            | API 호출, API 키 필요             | pip설치 후 Python에서 직접 사용      |
| 속도           | 매우 빠름 (동급 대비 수~수십배↑)                                      | 느림                          | 빠른 편(클라우드)                 | 모델/환경 따라 다름                 |
| 정확도         | Cross-Encoder와 유사 (리더보드 상위권)                                   | 최고 (최신 모델일수록↑)        | 최고 수준                         | Cross-Encoder 선택 가능              |
| 비용           | 무료(오픈소스)                                                           | 무료(오픈소스)                 | 유료(API 사용료)                  | 무료(오픈소스)                      |
| 커스터마이즈   | 모델 교체, 로컬 커스텀 가능                                               | 자유롭게 가능                  | 제한적                            | 자유롭게 가능                       |
| 장점           | 빠름, 정확, 오픈소스, 실무 적용 쉬움                                     | 정확, 다양한 모델              | 쉬운 사용, 인프라 불필요           | 다양한 모델, 오픈소스, 커스텀 가능   |
| 단점           | 인프라 직접 구축 필요(클라우드 X)                                         | 느림, 인프라 필요              | 커스텀 불가, 비용발생              | 인프라 직접 구축 필요                |



---  
## Reference
[RankLLM Reranker](https://python.langchain.com/docs/integrations/document_transformers/rankllm-reranker/)  
[Cohere reranker](https://python.langchain.com/docs/integrations/retrievers/cohere-reranker/)  
[FlashRank reranker](https://python.langchain.com/docs/integrations/retrievers/flashrank-reranker/)  
[FlashRank](https://github.com/PrithivirajDamodaran/FlashRank)  
[Reranker](https://wikidocs.net/253434)  


