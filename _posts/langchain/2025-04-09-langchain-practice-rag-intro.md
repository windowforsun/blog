--- 
layout: single
classes: wide
title: "[LangChain] LangChain RAG Introduction"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain RAG 에 대한 설명과 구현 예시를 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - RAG
    - LLM
    - AI
    - Embedding
    - Vector Store
    - Chroma
    - Nomic
    - Retriever
    - Chunking
toc: true
use_math: true
---  

## RAG
`RAG(Retrieval-Augmented Generation)` 은 `LLM(Large Language Model)` 의 한계를 보완하기 위해 외부 지식 검색(`Retrieval`)과 
텍스트 생성(`Generation`)을 결합한 기법이다. 
이는 주어진 컨텍스트나 질문에 대해 더욱 정확하고 풍부한 정보를 제공하는 방법이다. 
모델이 학습 데이터에 포함되지 않는 외부 데이터를 실시간으로 검색하고, 
이를 바탕으로 답변을 생성하는 과정을 포함한다. 
특히 환각(`hallucination`)을 방지하고, 모델이 최신 정보를 반영하거나 더 넓은 지식을 활용할 수 있게 한다.  

### RAG 주요 구성 요소
- `Retriever`(검색기) : 사용자의 질문에 대해 관련 문서를 검색한다. 벡터 검색, 키워드 검색, BM25 등 다양한 검색 기술을 사용할 수 있다. 여기서 벡터 스토어는 고차원 벡터를 저장하고 유사성 검색을 수행하는 데이터베이스이다. 
- `Generator`(생성기) : 검색된 문서를 바탕으로 응답을 생성한다. 주로 `Transformer` 기반의 언어 모델을 사용한다. `GPT-3`, `T5`, `BERT` 등이 있다.

### RAG 작동 방식
- 질문 입력 : 사용자가 질문을 입력한다. 
- 문서 검색 : 검색기가 잘문과 관련된 문서를 데이터베이스에서 검색한다. 
- 응답 생성 : 생성기가 검색된 문서를 바탕으로 질문에 대한 응답을 생성한다. 

### RAG 장점
- 정확성 향상 : 검색된 문서를 바탕으로 응답을 생성하기 때문에, 단순히 생성 모델만 사용하는 것보다 더 정확한 응답을 생성할 수 있다. 
- 정보 풍부성 : 외부 지식 소스를 활용하여 더 풍부하고 상세한 정보를 제공할 수 있다. 
- 유연성 : 다양한 검색 기술과 생성 모델을 결합하여 사용할 수 있다. 


## RAG 구현 예시
다음은 `비트코인` 관련 뉴스기사를 활요해 `RAG` 모델을 구현한 예시이다. 
`LLM` 모델로는 `groq` 를 사용해 `llama-3.3-70b-versatile` 모델을 사용했고, 
임베딩은 `Nomic` 에서 제공하는 `nomic-embed-text-v1.5` 모델을 사용했다. 

`RAG` 파이프라인은 크게 데이터 로드, 텍스트 분할, 인덱싱, 검색, 생성과 같은 다섯 단계로 구성된다. 

### 환경 설정
본 예제를 구현 및 실행하기 위해 필요한 파이썬 패키지는 아래와 같다.  

```text
# requirements.txt

langchain
langchain_core
langchain_groq
langchain_nomic
langchain_community
chromadb
beautifulsoup4

```  

코드 실행에 필요한 전체 `import` 구문은 아래와 같다. 

```python
import bs4
from langchain_community.document_loaders import WebBaseLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
import os
import getpass
from langchain_community.vectorstores import Chroma
from langchain_nomic import NomicEmbeddings
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser
```


### Raw Data Load
`RAG` 에서 사용할 데이터를 불러오는 단계로, 
외부 데이터 소스에서 정보를 수집하고 변환해 `Document` 화 시키는 작업이다. 
데이터는 다양한 방식으로 수집할 수 있고, 본 예제에서는 `WebBaseLoader` 를 사용해 웹 뉴스 기사를 크롤링해 사용한다. 


```python
# 비트코인 관련 뉴스 기사
loader = WebBaseLoader(
    web_paths=(
        'https://n.news.naver.com/mnews/article/001/0015253127?sid=104',
        'https://n.news.naver.com/mnews/article/003/0013105353?sid=104',
        'https://n.news.naver.com/mnews/article/053/0000048572?sid=101',
        'https://n.news.naver.com/mnews/article/003/0013102527?sid=101',
        'https://n.news.naver.com/mnews/article/003/0013104231?sid=104',
        'https://n.news.naver.com/mnews/article/001/0015252666?sid=104',
        'https://n.news.naver.com/mnews/article/028/0002734568?sid=101',
        'https://n.news.naver.com/mnews/article/001/0015249955?sid=100',
        'https://n.news.naver.com/mnews/article/011/0004458800?sid=105',
        'https://n.news.naver.com/mnews/article/020/0003619733?sid=100'


    ),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            'div',
            attrs={'class' : ['media_end_head_title', 'newsct_article _article_body']}
        )
    )
)

# 웹페이지 텍스트를 Documents 로 변환
docs = loader.load()

print(len(docs))
# 10
print(len(docs[0].page_content))
# 1136
print(docs[0].page_content[100:200])
# 프란시스코=연합뉴스) 김태종 특파원 = 가상화폐 대장주 비트코인이 7일(현지시간) 도널드 트럼프 미국 대통령이 주재한 첫 '디지털 자산 서밋'에도 하락세를 벗어나지 못하고 있다.
```  

### Text Split
불러온 `raw data` 를 작은 크기의 단위인 `chunk` 로 분할하는 과정이다. 
이는 자연어 처리(`NLP`) 활용해 큰 문서 처리의 용이성을 위해 무단, 문장 또는 구 단위로 나누는 작업을 의미하고, 검색 효율성을 높이기 위한 과정이다. 
또한 이를 통해 임베딩 모델이 처리할 수 있는 적당한 크기로 나누게 되어 임베딩 모델마다 한 번에 처리할 수 있는 토큰 수의 한계를 극복할 수 있다. 

텍스트 분할을 위해 `RecursiveCharacterTextSplitter` 를 사용한다. 
동작은 `1,136` 개로 이루어진 긴 문장을 최대 `200` 글자 단위로 분할한다. 
그리고 각 분할마다 `50` 글자가 겹치도록해 문맥이 잘려나가지 않고 유지되도록 한다.  

실제 사용에는 `LLM` 모델 또는 `API` 입력 크기 스펙에 따라 제한이 걸리지 않도록 적절한 크기로 설정을 해줘야 한다.  

```python
text_splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=50)
splits = text_splitter.split_documents(docs)

print(len(splits))
# 125
print(splits[10])
# page_content='이혜원 기자 = 도널드 트럼프 미국 대통령이 비트코인을 전략 비축하라는 행정명령을 내렸다.트럼프 대통령의 암호화폐 차르인 데이비드 색스는 6일(현지 시간) 소셜미디어 엑스(X, 옛 트위터)를 통해 "트럼프 대통령이 비트코인 전략 비축을 수립하라는 행정명령에 서명했다"고 밝혔다.색스는 "이번 비축은 민형사상 절차로 몰수된 연방 정부 보유 비트코인으로 구성될' metadata={'source': 'https://n.news.naver.com/mnews/article/003/0013105353?sid=104'}
print(splits[10].page_content)
# 이혜원 기자 = 도널드 트럼프 미국 대통령이 비트코인을 전략 비축하라는 행정명령을 내렸다.트럼프 대통령의 암호화폐 차르인 데이비드 색스는 6일(현지 시간) 소셜미디어 엑스(X, 옛 트위터)를 통해 "트럼프 대통령이 비트코인 전략 비축을 수립하라는 행정명령에 서명했다"고 밝혔다.색스는 "이번 비축은 민형사상 절차로 몰수된 연방 정부 보유 비트코인으로 구성될
print(splits[10].metadata)
# {'source': 'https://n.news.naver.com/mnews/article/003/0013105353?sid=104'}
```  

### Indexing(Embedding)
앞서 분할한 텍스트를 검색 가능한 헝태로 만드는 단계이다. 
인덱싱을 통해 검색 시간을 단축시키고, 정확도를 높일 수 있다. 
`langchain` 라이브러리를 통해 텍스트를 임베딩으로 변환하고, 
이를 `Chroma` 벡터 스토어에 저장해 임베딩 기반으로 유사성 검색을 수행할 수 있다.  

`Nomic` 에서 제공하는 임베딩 모델을 사용해 텍스트를 벡터로 변환하고, 이를 `Chroma` 벡터 스토어에 저장한다. 
그리고 `similarity_search()` 메서드를 사용해 주어진 쿼리 텍스트에 해당하는 가장 유사한 문서를 검색한다. 
쿼리와 유사성은 임베딩 간의 거리를 기반으로 계산된다.  

```python
# nomic api key 입력
os.environ["NOMIC_API_KEY"] = getpass.getpass("Enter your Nomic API key: ")

vectorstore = Chroma.from_documents(documents=splits, embedding=NomicEmbeddings(model="nomic-embed-text-v1.5"))
docs = vectorstore.similarity_search("비트 코인 현상황에 대해 알려주세요.")

print(len(docs))
# 4
print(docs[0].page_content)
# 코인과 비트코인 전략적 준비금을 금융정책과 연계해야 한다"고 주장했다.    김 총괄부본부장은 "트럼프 정부의 정책에 맞춰 외환보유고에 비트코인 편입 여부에 대한 논의를 시작해야 한다"며 "STO(토큰증권), 스테이블 코인 활성화를 통해 '디지털 금융 허브' 한국을 준비해야 할 것"이라고 말했다.    국회 정무위원회 소속 민병덕 의원도 이날 오후
```  

### Retrieval
사용자 질문이 들어오면 주어진 컨텍스트를 기반으로 가장 관련된 정보를 찾는 과정이다. 
이는 사용자 입력을 바탕으로 쿼리를 생성하고, 앞서 인덱싱한 데이터를 기반으로 가장 관련도 높은 정보를 검색하게 된다. 
`lanchain` 에서는 `retriever` 메소드를 사용하면 된다.  


### Generation
검색된 정보를 바탕으로 사용자의 쿼리에 답변을 생성하는 단계이다. 
`LLM` 모델에 사용자의 쿼리를 전달하고, 
모델은 앞서 학습한 지식과 검색 결과를 결합해 가장 절적한 답변을 생성한다.  

```python
llm = init_chat_model("llama-3.3-70b-versatile", model_provider="groq")


template = '''사용자의 질문을 사전에 학습한 context 를 기반으로 답변한다: {context}
질문: {question}
'''


prompt = ChatPromptTemplate.from_template(template)

# Retrieval
retriever = vectorstore.as_retriever()

# combine documents
def format_docs(docs):
    return '\n\n'.join(doc.page_content for doc in docs)

# RAG Chain 연결
rag_chain = (
        {'context' : retriever | format_docs, 'question' : RunnablePassthrough()}
        | prompt
        | llm
        | StrOutputParser()
)

# chain 실행
rag_chain.invoke('비트코인 현 상황에 대해 알려주세요.')
# 현재 비트코인의 상황은 강세를 보이고 있으며, 전문가들은 경기 회복 지표와 장기 상승 가능성에 주목하고 있습니다. 또한, 일부 전문가들은 비트코인의 강세가 시작 단계에 불과하다고 주장하고 있습니다. 그러나 양자컴퓨터의 등장으로 인해 암호화폐의 안전성에 대한 우려가 있기도 합니다. 하지만 양자컴퓨터가 실제로 비트코인에 영향을 미치기 위해서는 수백만 큐비트의 양자컴퓨터가 동원되어야 하므로, 현재로서는 가능성이 낮아 보입니다.
```  

학습시킨 뉴스 기사와 관련된 추가 내용을 질문하면 아래와 같다. 
그리고 관련없는 질문을 하는 경우 답변하지 않는 것을 확인할 수 있다.  

```python
rag_chain.invoke('비트코인의 가장 이슈는 무엇인가요?')
# 비트코인의 가장 큰 이슈는 양자컴퓨터의 등장과 관련하여 블록체인 암호화의 안전성에 대한 우려입니다. 양자컴퓨터의 개발로 인해 블록체인의 암호화가 깨질 가능성이 있어 비트코인의 가치 보존에 대한 우려가 제기되고 있습니다.

rag_chain.invoke('비트코인 가격 상승 전망에 대해 알려주세요')
# 비트코인 가격 상승 전망에 대해 알려드리겠습니다. 최근 비트코인 강세장에서 전문가들은 경기 회복 지표와 비트코인 가격의 상관관계를 주목하고 있습니다. 이들에 따르면, 경기 회복 지표가 긍정적인 방향으로 움직일 경우 비트코인 가격도 장기적으로 상승할 가능성이 있음을 시사합니다.
# 
# 구체적으로는, 전문가들은 경기 회복을 나타내는 지표들, 예를 들어 GDP 성장률, 소비자 信頼 지수, 또는 취업률 등이 긍정적인 추이를 보일 경우, 이는 비트코인과 같은 디지털 자산들의 가격에도 긍정적인 영향을 미칠 수 있다고 분석하고 있습니다.
# 
# 또한, 일부 전문가들은 비트코인 가격이 역사적으로 경기 회복기와 관련이 있음을 지적하며, 이러한 상관관계가 장기적으로 지속될 가능성도 언급하고 있습니다. 그러나, 비트코인 가격은 다양한 요인에 의해 영향을 받기 때문에, 이러한 전망은 불확실성과 변동성을 동반한다는 점을 명심해야 합니다.
# 
# 결론적으로, 비트코인 가격 상승 전망은 경기 회복 지표와 상관관계가 있음을 시사합니다. 그러나, 비트코인 투자는 높은 위험이 수반되므로, 투자 전반에 걸쳐 신중한 의사 결정과 투자 전략이 필요합니다.

rag_chain.invoke('비트코인 급락 여지가 있나요?')
# 비트코인이 하락하는 이유 중 하나는 양자컴퓨터의 등장에 따른 안전성 우려입니다. 양자컴퓨터의 등장으로 암호화폐의 안전성이 손상될 수 있다는 우려가 있습니다. 하지만 현재로서는 양자컴퓨터의 개발이 아직 초기 단계에 있으며, 비트코인이나 다른 암호화폐의 보안을 깨는 데에는 수백만 큐비트의 양자컴퓨터가 필요하다는 업계 평가입니다. 따라서, 비트코인이나 다른 암호화폐의 가격이 하락하는 이유는 다양하며, 단순히 양자컴퓨터의 등장으로 인한 것만은 아닙니다.

rag_chain.invoke('미국과 비트코인과 관련된 내용에 대해 설명해주세요')
# 미국 정부는 비트코인에 대해 초기에는 냉소적인 시선을 보내왔습니다. 비트코인을 사이버 범죄, 탈세, 제재 회피, 테러를 위한 재정지원 등의 수단으로 간주하였습니다. 그러나 최근에 미국 정부는 비트코인에 대한 태도를 바꾸고 있으며, 미 정부가 비트코인의 전략 비축을 추진하겠다고 발표했습니다. 다만, 세금으로 가상화폐를 구매하지는 않을 것이라고 했습니다. 이에 따라 비트코인의 신분상승이 시도되고 있으며, 미국의 최대 가상화폐 거래소 코인베이스의 최고경영자가 열리는 '디지털 자산 서밋'에 참석하는 등 비트코인에 대한 관심이 증가하고 있습니다.

rag_chain.invoke('트럼프와 비트코인과 관련된 내용에 대해 설명해주세요')
# 트럼프 대통령은 최근 비트코인과 관련된 정책을 발표하였습니다. 그는 미국의 외환보유액에 비트코인을 편입하는 방안을 제시하였으며, 이는 트럼프 정부의 암호화폐 전략 비축의 일환입니다. 또한, 트럼프 대통령은 비트코인과 관련된 산업을 지원하고, 규제를 완화하는 방안을 모색하고 있습니다.
# 
# 트럼프 대통령의 아들들은 암호화폐 플랫폼 업체에 투자하였으며, 트럼프 대통령이 설립한 사회관계망서비스(SNS) '트루스소셜' 운영업체는 자산의 상당 부분을 가상화폐에 투자한 것으로 알려졌습니다. 이러한 사실로 인해 트럼프 대통령의 자산 증식을 위해 전략비축 코인을 추진하는 것이 아니냐는 의혹이 제기되고 있습니다.
# 
# 트럼프 대통령의 비트코인 정책은 새로운 지지기반을 굳히려는 정치적 노림수로 보는 분석도 있습니다. 그는 가상자산에 대한 규제를 두고バイ든 행정부와 대립하던 거액 자산가들을 자신의 진영으로 끌어들여 새로운 정치자금줄의 물꼬를 트려는 선심성 정책이라는 것입니다.
# 
# 그러나, 트럼프 대통령의 비트코인 정책은 실효성 논란이 제기되고 있습니다. 그의 政策이 어떻게 작동할지, 미국 국민에게 실제로 혜택이 될지는 여전히 불분명합니다. 일부 전문가들은 트럼프 대통령이 추후 입장을 더 적극적으로 바꿔 정부가 암호화폐를 직접 매입하기 시작한다면 일부 '큰 손'들만 이익을 볼 것이란 관측을 내세우고 있습니다.

rag_chain.invoke('오늘 날씨는 어떤가요?')
# 해당 문맥에는 날씨에 대한 정보가 없습니다. 제공된 문맥은 주로 비트코인 가격과 가상자산에 관한 뉴스 기사입니다.
```  



---  
## Reference
[Build a Retrieval Augmented Generation (RAG) App: Part 1](https://python.langchain.com/docs/tutorials/rag/#setup)  
[Build a Retrieval Augmented Generation (RAG) App: Part 2](https://python.langchain.com/docs/tutorials/qa_chat_history/)  


