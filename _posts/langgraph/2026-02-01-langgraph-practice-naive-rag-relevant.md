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

간단한 질문을 던졌을 때 출처까지 포함해 문서 내용을 바탕으로 최종 답변이 생성되는 것을 확인할 수 있다.  

```python
question = '6월 평균 기온에 대해 알려줘'
chain_result = pdf_chain.invoke(
    {
        'question' : question,
        'context' : pdf_retriever.invoke(question),
        'chat_history' : []
    }
)
# 2025년 6월 평균 기온은 22.9℃이고 평년 값은 21.4℃이며, 평년 편차는 +1.5℃입니다.
# 
# **출처**:
# * /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf

question = '5월 평균 기온에 대해 알려줘'
chain_result = pdf_chain.invoke(
    {
        'question' : question,
        'context' : pdf_retriever.invoke(question),
        'chat_history' : []
    }
)
# 2025년 5월 평균 기온은 16.8℃이고, 평균 최고 기온은 22.4℃이며, 평균 최저 기온은 11.5℃입니다.
# 
# **출처:**
# * /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf

question = '5월 과 6월 강수량 특징에 대해 설명 및 비교해줘'
chain_result = pdf_chain.invoke(
    {
        'question' : question,
        'context' : pdf_retriever.invoke(question),
        'chat_history' : []
    }
)
# 5월 전국 평균 강수량은 116.6mm이고, 강수일수는 10.6일이며, 6월 전국 평균 강수량은 작년보다 56.9mm 많았습니다.
# 
# **출처:**
# * /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf
# * /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf

question = '대한민국의 지금까지 기후에 대해 특징 설명해줘'
chain_result = pdf_chain.invoke(
    {
        'question' : question,
        'context' : pdf_retriever.invoke(question),
        'chat_history' : []
    }
)
# 2025년 3월에는 중국 내륙의 따뜻하고 건조한 공기가 강한 서풍을 타고 유입되고 햇볕이 더해지면서 일 최고기온 최고 5순위 이내를 기록한 지역이 많았으며, 하순에는 고온이 지속되었습니다. 2025년 4월에는 찬 대륙고기압의 강도가 평년 대비 약하고 우리나라 주변 기압계 흐름이 원활하여 추위와 더위가 연이어 발생하는 급격한 기온 변동을 보였습니다. 2025년 6월에는 북태평양고기압 가장자리를 따라 덥고 습한 공기가 유입되고 햇볕이 더해지면서 폭염과 열대야가 발생했습니다.
# 
# **출처:**
# - /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf(페이지 6)
# - /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf(페이지 0)
# - /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf(페이지 0)
# - /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf(페이지 1)
```



### Build Graph
이제 위 클래스를 사용해서 `RAG` 구조를 그래프로 정의한다.  

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage

# 상태 정의
class GraphState(TypedDict):
  question: Annotated[str, "Question"]
  context: Annotated[str, 'Context']
  answer: Annotated[str, 'Answer']
  messages: Annotated[list, add_messages]
  
  
# 노드 정의
def retrieve_document(state: GraphState) -> GraphState:
    latest_question = state['question']
    retrieved_docs = pdf_retriever.invoke(latest_question)
    retrieved_docs = format_docs(retrieved_docs)
    return GraphState(context=retrieved_docs)

def llm_anwser(state: GraphState) -> GraphState:
    latest_question = state['question']
    context = state['context']
    response = pdf_chain.invoke(
        {
            'question' : latest_question,
            'context' : context,
            'chat_history' : messages_to_history(state['messages'])
        }
    )

    return GraphState(
        answer=response,
        messages=[('user', latest_question), ('assistant', response)]
    )

  
# 유틸 함수
def get_role_from_messages(msg):
    if isinstance(msg, HumanMessage):
        return "user"
    elif isinstance(msg, AIMessage):
        return "assistant"
    else:
        return "assistant"

def messages_to_history(messages):
    return "\n".join(
        [f"{get_role_from_messages(msg)}: {msg.content}" for msg in messages]
    )

def format_docs(docs):
    return "\n".join(
        [
            f"<document><content>{doc.page_content}</content><source>{doc.metadata['source']}</source><page>{int(doc.metadata['page'])+1}</page></document>"
            for doc in docs
        ]
    )

def format_searched_docs(docs):
    return "\n".join(
        [
            f"<document><content>{doc['content']}</content><source>{doc['url']}</source></document>"
            for doc in docs
        ]
    )

def execute_graph(graph, config, inputs):

    for event in graph.stream(inputs, config, stream_mode="updates"):
        # 업데이트 딕셔너리 순회
        for k, v in event.items():
            # 메시지 목록 출력
            print(k)
            print(v)


def format_task(tasks):
    # 결과를 저장할 빈 리스트 생성
    task_time_pairs = []

    # 리스트를 순회하면서 각 항목을 처리
    for item in tasks:
        # 콜론(:) 기준으로 문자열을 분리
        task, time_str = item.rsplit(":", 1)
        # '시간' 문자열을 제거하고 정수로 변환
        time = int(time_str.replace("시간", "").strip())
        # 할 일과 시간을 튜플로 만들어 리스트에 추가
        task_time_pairs.append((task, time))

    # 결과 출력
    return task_time_pairs

```  
