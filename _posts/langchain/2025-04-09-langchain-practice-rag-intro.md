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
