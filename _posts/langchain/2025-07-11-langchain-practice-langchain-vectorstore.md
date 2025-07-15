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


### 사전 준비

`VectorStore` 테스트를 위해 아래와 같은 문서를 사용한다.  

- `computer-keywords.txt`

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

- `property-keywords.txt`

```text
아파트

정의: 아파트는 다층 건물 내에 여러 세대가 독립적으로 거주하는 공동주택으로, 현대 도시 주거의 대표적인 형태입니다.
예시: 강남구 래미안 아파트는 85㎡ 면적에 20층 높이의 건물로, 지하주차장과 커뮤니티 시설을 갖추고 있습니다.
연관키워드: 공동주택, 분양, 입주, 관리비, 세대

전세

정의: 전세는 집주인에게 일정 금액(전세금)을 보증금으로 맡기고 계약 기간 동안 월세 없이 거주한 후, 계약 만료 시 전세금을 돌려받는 한국의 독특한 주택 임대 방식입니다.
예시: 서울 마포구 원룸을 2억 원 전세로 2년 계약하고 월세 부담 없이 거주하고 있습니다.
연관키워드: 보증금, 계약갱신, 전세자금대출, 임대차보호법

월세

정의: 월세는 집주인에게 일정 금액의 보증금을 맡기고 매달 임대료를 지불하는 주택 임대 방식입니다.
예시: 보증금 1천만 원에 월 50만 원의 조건으로 원룸을 임대하는 계약을 체결했습니다.
연관키워드: 보증금, 임대료, 갱신, 임대차계약서, 월세전환율

부동산 중개

정의: 부동산 중개는 부동산 거래 당사자 간의 매매, 임대차 등의 계약 체결을 알선하는 서비스입니다.
예시: 공인중개사는 매도인과 매수인 사이에서 적정 가격을 제안하고 계약 체결을 도왔습니다.
연관키워드: 공인중개사, 중개수수료, 부동산 거래, 중개계약, 거래정보망

등기부등본

정의: 등기부등본은 부동산의 소유권, 저당권 등 권리관계와 현황이 기재된 공적 문서로, 누구나 열람할 수 있습니다.
예시: 주택 구매 전 등기부등본을 확인하여 해당 물건에 근저당이 설정되어 있는지 확인했습니다.
연관키워드: 소유권, 담보권, 등기소, 권리관계, 인터넷등기소

재개발

정의: 재개발은 노후화된 주거지역을 철거하고 새로운 주택단지로 조성하는 도시 정비 사업입니다.
예시: 서울 용산구 한남동 일대가 재개발 구역으로 지정되어 향후 5년 내 고층 아파트 단지로 탈바꿈할 예정입니다.
연관키워드: 도시정비사업, 조합, 이주비, 분담금, 관리처분계획

재건축

정의: 재건축은 노후화된 공동주택을 허물고 같은 자리에 새로운 주택을 건설하는 사업입니다.
예시: 강남구 대치동의 30년 된 아파트는 재건축을 통해 현대식 고층 아파트로 변모했습니다.
연관키워드: 안전진단, 조합설립, 용적률, 층고제한, 재건축초과이익환수제

분양권

정의: 분양권은 아파트 등 주택의 신규 건설 시 건설사와 맺은 분양계약에 따른 입주할 수 있는 권리입니다.
예시: 아직 건설 중인 아파트의 분양권을 프리미엄 2천만 원을 주고 매수했습니다.
연관키워드: 분양계약, 중도금, 프리미엄, 명의변경, 입주권

청약

정의: 청약은 신규 주택 공급 시 일정 자격을 갖춘 사람들이 주택을 분양받기 위해 신청하는 제도입니다.
예시: 부산 해운대구 신축 아파트 1순위 청약에 10대 1의 경쟁률을 기록했습니다.
연관키워드: 청약통장, 1순위, 가점제, 청약경쟁률, 당첨자발표

LTV

정의: LTV(Loan to Value ratio)는 주택담보대출비율로, 주택 가격 대비 대출 가능한 최대 금액의 비율을 의미합니다.
예시: 정부는 투기과열지구의 LTV를 40%로 제한하여 6억 원 아파트 구입 시 최대 2억 4천만 원까지만 대출이 가능합니다.
연관키워드: 담보대출, 규제지역, 대출한도, 주택금융, 부채

DTI

정의: DTI(Debt to Income ratio)는 총부채상환비율로, 연간 소득 대비 연간 부채 상환액의 비율을 나타냅니다.
예시: DTI 규제가 40%로 설정되어, 연소득 7천만 원인 사람은 연간 원리금 상환액이 2,800만 원을 넘는 주택담보대출을 받을 수 없습니다.
연관키워드: 대출규제, 소득증빙, 원리금상환, 신용평가, 주택금융

공시지가

정의: 공시지가는 국토교통부가 매년 1월 1일 기준으로 전국의 토지에 대해 공시하는 공적인 땅값으로, 각종 세금 산정의 기준이 됩니다.
예시: 강남구 압구정동의 공시지가는 전년 대비 5% 상승하여 평당 3천만 원을 기록했습니다.
연관키워드: 개별공시지가, 표준지, 토지 과세, 시세, 감정평가

부동산 세금

정의: 부동산 세금은 부동산의 취득, 보유, 양도 과정에서 발생하는 다양한 조세를 통칭합니다.
예시: 9억 원 초과 주택을 구매할 경우 취득세율이 1~3%로 차등 적용됩니다.
연관키워드: 취득세, 재산세, 종합부동산세, 양도소득세, 양도세 중과

주택담보대출

정의: 주택담보대출은 주택을 담보로 금융기관에서 자금을 빌리는 대출 상품으로, 주택 구입이나 자금 조달에 활용됩니다.
예시: 4억 원 아파트 구입을 위해 기존 주택을 담보로 2억 원의 주택담보대출을 받았습니다.
연관키워드: 금리, 담보설정, 원리금균등상환, 중도상환수수료, 대출심사

오피스텔

정의: 오피스텔은 업무와 주거 기능을 겸한 건물로, 법적으로는 업무시설이지만 실제로는 주거용으로 많이 활용됩니다.
예시: 서울 역삼동의 30평형 오피스텔은 업무공간과 주거공간이 분리된 구조로 설계되었습니다.
연관키워드: 원룸, 투자용 부동산, 수익률, 관리비, 생활숙박시설

분양가상한제

정의: 분양가상한제는 신규 주택의 분양가격을 정부가 정한 상한액 이하로 제한하는 제도입니다.
예시: 분양가상한제가 적용된 서울 강동구 고덕지구의 아파트는 주변 시세보다 20% 낮은 가격에 공급되었습니다.
연관키워드: 분양가 규제, 원가공개, 민간택지, 공공택지, 후분양제

임대차 계약

정의: 임대차 계약은 임대인과 임차인이 주택의 사용과 대가 지불에 관해 맺는 법적 약속입니다.
예시: 임대차 계약 시 계약금, 계약기간, 계약조건 등을 명시한 표준임대차계약서를 작성했습니다.
연관키워드: 주택임대차보호법, 계약갱신요구권, 임대료인상제한, 임차권등기, 묵시적 갱신

실거래가

정의: 실거래가는 부동산 거래 시 실제로 거래된 금액으로, 국토교통부에 신고된 공식적인 거래 가격입니다.
예시: 강남구 아파트의 실거래가는 호가(매도 희망가)보다 약 5% 낮은 수준에서 형성되고 있습니다.
연관키워드: 실거래가 신고, 부동산 시세, 허위신고, 거래가격 검증, 국토교통부

토지 이용계획

정의: 토지 이용계획은 국가나 지방자치단체가 토지의 효율적 이용을 위해 용도와 이용 방식을 계획하고 규제하는 제도입니다.
예시: 해당 토지는 토지 이용계획상 제1종 일반주거지역으로 지정되어 있어 15층 이하의 주택만 건설할 수 있습니다.
연관키워드: 용도지역, 도시계획, 건폐율, 용적률, 지구단위계획

부동산 투자

정의: 부동산 투자는 수익이나 자산가치 상승을 목적으로 토지, 건물 등의 부동산에 자금을 투입하는 행위입니다.
예시: 수도권 역세권 오피스텔에 투자하여 월 임대 수익 100만 원과 연 5%의 시세 상승 효과를 얻고 있습니다.
연관키워드: 수익률, 임대수익, 시세차익, 투자위험, 부동산 펀드
```  

`VectorStore` 사용전 미리 텍스트 파일을 로드하고 `RecursiveCharacterTextSplitter` 를 사용해 `chunking` 한다.  

```python
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma

text_splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=100)

computerKeywordLoader = TextLoader("./computer-keywords.txt")
propertyKeywordLoader = TextLoader("./property-keywords.txt")

split_computer_keywords = computerKeywordLoader.load_and_split(text_splitter)
split_property_keywords = propertyKeywordLoader.load_and_split(text_splitter)

print(len(split_computer_keywords))
# 20
print(len(split_property_keywords))
# 19
```  

`VectorStore` 구성을 위해 사용하는 `Embedding` 모델은 `Hugging Face` 에서 제공하는 `SentenceTransformer` 모델을 사용한다. 
`SentenceTransformer` 모델 중 한글 임베딩에 적합한 [KoSimCSE](https://github.com/BM-K/KoSimCSE-SKT)
을 사용한다.  


```python
from langchain_huggingface.embeddings import HuggingFaceEmbeddings

os.environ["HUGGINGFACEHUB_API_TOKEN"] = "hf_xxx"  # Hugging Face Hub API Token
model_name = "BM-K/KoSimCSE-roberta"

hf_embeddings = HuggingFaceEmbeddings(
    model_name=model_name,
)
```  




### Chroma
`Chroma` 는 파이썬 기반의 경량 벡터 데이터베이스이다. 
텍스트, 이미지 등 다양한 데이터를 임베딩 벡터로 저장하고, 유사도 검색을 빠르게 할 수 있다. 
설치와 사용이 매우 쉽고, 로컬 환경에서 바로 사용할 수 있다는 장점이 있다. 

주요 특징은 아래와 같다. 

- 간편한 설치 : `pip install chromadb` 로 바로 설치해서 사용
- 로컬 환경 지원 : 별도의 서버 없이 파일 시스템에 데이터를 저장
- 바른 검색 : 소규모-중규모 데이터에 대해 빠른 벡터 유사도 검색을 제공
- 메타데이터 지원 : 벡터와 함께 다양한 메타데이터(문서 ID, 태그 등) 저장 가능
- `LangChain` 과 완벽 연동 : `LangChain` 의 `VectorStore` 인터페이스를 통해 쉽게 사용

단점으로는 아래와 같은 것들이 있다. 

- 대용량 데이터(수백만~수억 건)에는 적합하지 않다.
- 분산 환경, 클라우드 기반 확장성은 제한적이다.

#### from_documents
`from_documents` 는 `Chroma` 에서 여러 개의 문서(`Document`) 객체를 임베딩하여 `Chroma` 벡터스토어에 저장하는 메서드이다. 
각 문서는 텍스트와 함께 메타데이터도 함께 저장할 수 있다. 
주로 문서 단위로 검색, 분류, 필터링이 필요한 경우 적합하게 사용할 수 있다. 

`from_documents` 메서드는 아래와 같은 인자를 받는다. [참고](https://python.langchain.com/api_reference/chroma/vectorstores/langchain_chroma.vectorstores.Chroma.html#langchain_chroma.vectorstores.Chroma.from_documents)

- `documents` : 저장할 문서 리스트
- `embedding` : 문서 임베딩을 위한 임베딩 모델
- `ids` : 문서 ID 리스트 (선택적)
- `collection_name` : 저장할 컬렉션 이름 (선택적)
- `persist_directory` : 벡터스토어를 저장할 디렉토리 (선택적)
- `client_settings` : 클라이언트 설정 (선택적)
- `client` : 클라이언트 객체 (선택적)
- `collection_metadata` : 컬렉션 메타데이터 (선택적)

인자중 `persist_directory` 를 지정하면, 벡터스토어를 디스크에 저장할 수 있다. 
지정되지 않으면 데이터는 메모리에 임시로 저장된다. 

아래는 메모리를 사용하는 임시 벡터스토어를 생성하는 예시이다. 

```python
memory_db = Chroma.from_documents(documents=split_computer_keywords, 
                                  embedding=hf_embeddings, 
                                  collection_name="computer_keywords_db")
```  

아래는 `persist_directory` 인자를 사용해 벡터스토어를 디스크에 저장하는 예시이다. 

```python
db_path = "./chroma_computer_keywords_db"

persistent_db = Chroma.from_documents(documents=split_computer_keywords,
                                      persist_directory=db_path,
                                      embedding=hf_embeddings,
                                      collection_name="computer_keywords_db")
```  

디스크에 저장된 벡터스토어는 아래와 같이 불러올 수 있다. 

```python
loaded_db = Chroma(persist_directory=db_path, 
                   embedding_function=hf_embeddings,
                   collection_name="computer_keywords_db")
```  

#### from_texts
`from_texts` 는 `Chroma` 에서 어려 개의 텍스트(문장, 문단 등)을 임베딩하여 벡트 스토어에 저장하는 메서드이다. 
`from_documents` 와 유사하지만, `Document` 객체가 아닌 일반 텍스트를 사용한다.  

`from_texts` 메서드는 아래와 같은 인자를 받는다. [참고](https://python.langchain.com/api_reference/chroma/vectorstores/langchain_chroma.vectorstores.Chroma.html#langchain_chroma.vectorstores.Chroma.from_texts)

- `texts` : 저장할 텍스트 리스트
- `embedding` : 텍스트 임베딩을 위한 임베딩 모델
- `metadatas` : 텍스트 메타데이터 리스트 (선택적)
- `ids` : 텍스트 ID 리스트 (선택적)
- `collection_name` : 저장할 컬렉션 이름 (선택적)
- `persist_directory` : 벡터스토어를 저장할 디렉토리 (선택적)
- `client_settings` : 클라이언트 설정 (선택적)
- `client` : 클라이언트 객체 (선택적)
- `collection_metadata` : 컬렉션 메타데이터 (선택적)

아래는 `from_texts` 를 사용해서 메모리에 임시 벡터스토어를 생성하는 예시이다.  

```python
memory_db_2 = Chroma.from_texts(
    ["안녕하세요.", "반갑습니다.", "오늘 점심은 제육입니다."],
    embedding=hf_embeddings
)
``` 
