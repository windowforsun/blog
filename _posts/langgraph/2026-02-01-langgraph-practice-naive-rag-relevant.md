--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Naive RAG"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
toc: true
use_math: true
---  

## LangGraph Naive RAG
`Naive RAG` 는 `Retrieval-Augmented Generation` 의 가장 기본적인 형태이다. 
`RAG` 는 자연어 처리 분야에서 정보를 생성할 때 외부 지식 소스를 검색(`retrieval`)하여 생성(`generation`) 모델에 결합하는 방법론이다. 
`naive` 란 용어는 복잡한 최적화나 고도화된 검색, 생성, 젼략 없이 가장 단순한 방식으로 검색과 생성을 결합한 구조이기 때문이다. 

데모 예제는 챗봇이 사용자의 질문에 답하기 위해 PDF 문서에서 정보를 검색하고, 이를 바탕으로 답변을 생성하는 구조로 되어 있다.  


### Define Retrieval Class
먼저 `RAG` 에 대한 구현을 클래스로 정의한다. 
해당 클래스에는 `RAG` 관련 `Vectorstore` 생성, `Retriever` 생성, `LLM` 생성, `Prompt` 생성 등의 메소드가 포함되어 있다.  

```python
from langchain_community.document_loaders import PDFPlumberLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from typing import List, Annotated
from langchain_chroma import Chroma
from langchain.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate

from abc import ABC, abstractmethod
from operator import itemgetter
from langchain_core.output_parsers import StrOutputParser

class DemoRetrievalChain:
  def __init__(self, **kwargs):
    self.source_uri = kwargs.get('source_uri', None)
    self.k = kwargs.get('k', 10)
    self.embeddings = kwargs.get('embeddings', None)
    self.llm = kwargs.get('llm', None)
    self.collection_name = kwargs.get('collection_name', None)
    self.persist_directory = kwargs.get('persist_directory', None)

  def load_documents(self, source_uris):
    docs = []

    for uri in source_uris:
      loader = PDFPlumberLoader(uri)
      docs.extend(loader.load())
    return docs

  def create_text_splitter(self):
    return RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=50)

  def split_documents(self, docs, text_splitter):
    return text_splitter.split_documents(docs)

  def get_embeddings(self):
    return self.embeddings

  def create_vectorstore(self, split_docs):
    if os.path.exists(self.persist_directory):
      return Chroma(
          embedding_function=self.get_embeddings(),
          collection_name=self.collection_name,
          persist_directory=self.persist_directory
      )
    else:
      return Chroma.from_documents(
          documents=split_docs,
          embedding=self.get_embeddings(),
          collection_name=self.collection_name,
          persist_directory=self.persist_directory
      )

  def create_retriever(self, vectorstore):
    dens_retriever = vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={"k": self.k}
    )
    return dens_retriever

  def get_llm(self):
    return self.llm

  def create_prompt(self):
    return ChatPromptTemplate.from_messages([
        SystemMessagePromptTemplate.from_template("""
        귀하는 검색 증강 생성(RAG) 시스템 내에서 질문 응답(QA) 작업을 전문으로 하는 AI 비서입니다.
        당신의 주요 임무는 제공된 문맥이나 채팅 기록을 바탕으로 질문에 답하는 것입니다.
        답변이 간결하고 추가 내레이션 없이 질문에 직접 답변할 수 있도록 하세요.

        ###

        질문에 답하기 위해 이전 대화 기록을 고려할 수 있습니다.

        # 이전 대화 기록은 다음과 같습니다:
        {chat_history}

        ###

        최종 답변은 간결하게 작성해야 하며(중요한 수치, 전문 용어, 전문 용어, 이름 등을 포함해야 합니다), 그 다음에 정보의 출처를 밝혀야 합니다.

        # 단계

        1. 제공된 맥락을 주의 깊게 읽고 이해하세요.
        2. 문맥 내에서 질문과 관련된 주요 정보를 식별하세요.
        3. 관련 정보를 바탕으로 간결한 답변을 작성하세요.
        4. 최종 답변이 문제를 직접 해결하도록 하세요.
        5. 답변의 출처를 글머리 기호로 나열하십시오. 이는 파일 이름(페이지 번호 포함) 또는 컨텍스트의 URL이어야 합니다. 답변이 이전 대화를 기반으로 하거나 출처를 찾을 수 없는 경우 생략합니다.

        # 출력 형식:
        [숫자 값, 전문 용어, 전문 용어 및 원래 언어로 된 이름을 포함한 최종 답변]

        **출처**(선택 사항)
        - 답변의 출처는 파일 이름(페이지 번호 포함) 또는 컨텍스트의 URL이어야 합니다. 답변이 이전 대화를 기반으로 하거나 출처를 찾을 수 없는 경우 생략합니다
        - (여러 출처가 있는 경우 자세히 나열하세요)
        - ...

        ###

        기억하세요:
        - 답변은 **제공된 컨텍스트** 또는 **채팅 기록**에만 기반하는 것이 중요합니다.
        - 주어진 자료에 없는 외부 지식이나 정보를 사용하지 마세요.
        - 사용자가 이전 대화를 바탕으로 질문하지만 이전 대화가 없거나 정보가 충분하지 않은 경우 모른다고 대답해야 합니다.
        """),
        HumanMessagePromptTemplate.from_template("""
        # 사용자의 질문은 다음과 같습니다:
        {question}

        # 질문에 답하기 위해 사용해야 할 맥락은 다음과 같습니다:
        {context}

        # 사용자의 질문에 대한 최종 답변:
        """)
    ])

  @staticmethod
  def format_docs_2(docs):
    return "\n".join(docs)

  @staticmethod
  def format_docs(docs):
    return "\n".join(
        [
            f"<document><content>{doc.page_content}</content><source>{doc.metadata['source']}</source><page>{int(doc.metadata['page'])+1}</page></document>"
            for doc in docs
        ]
    )

  @staticmethod
  def format_searched_docs(docs):
    return "\n".join(
        [
            f"<document><content>{doc['content']}</content><source>{doc['url']}</source></document>"
            for doc in docs
        ]
    )

  def create_chain(self):
    docs = self.load_documents(self.source_uri)
    text_splitter = self.create_text_splitter()
    split_docs = self.split_documents(docs, text_splitter)
    self.vectorstore = self.create_vectorstore(split_docs)
    self.retriever = self.create_retriever(self.vectorstore)
    llm = self.get_llm()
    prompt = self.create_prompt()
    self.chain = (
        {
            'question' : itemgetter('question'),
            'context' : itemgetter('context'),
            'chat_history' : itemgetter('chat_history')
        }
        | prompt
        | llm
        | StrOutputParser()
    )

    return self
```  

구현한 `DemoRetrievalChain` 클래스의 구현 결과를 확인해 본다. 
문서는 기상청에 있는 날씨 기후에 대한 보고서를 사용한다.  

```python
from langchain_huggingface.embeddings import HuggingFaceEndpointEmbeddings
from langchain_huggingface.embeddings import HuggingFaceEmbeddings
from langchain_google_genai import ChatGoogleGenerativeAI
import os

os.environ["GOOGLE_API_KEY"] = "api key"
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

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


# 디렉토리 경로
directory_path = "/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs"

file_list = [
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf'
]

# Ensure the persist directory exists
collection_name='weather-docs-6'
persist_directory = f"/content/drive/MyDrive/Colab Notebooks/data/vectorstore/{collection_name}"

pdf = DemoRetrievalChain(
    source_uri=file_list,
    embeddings=hf_embeddings,
    llm=llm,
    collection_name=collection_name,
    persist_directory=persist_directory
).create_chain()
```  
