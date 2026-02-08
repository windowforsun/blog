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

이제 구현된 모든 내용들을 사용해서 `RAG` 구조를 그래프로 정의한다.  

```python
# 그래프 정의

from langgraph.graph import START, END, StateGraph
from langgraph.checkpoint.memory import MemorySaver

from IPython.display import Image, display

graph_builder = StateGraph(GraphState)

graph_builder.add_node('retrieve', retrieve_document)
graph_builder.add_node('llm_answer', llm_anwser)

graph_builder.add_edge('retrieve', 'llm_answer')
graph_builder.add_edge('llm_answer', END)

graph_builder.set_entry_point('retrieve')

memory = MemorySaver()

graph = graph_builder.compile(memory)

# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```


![그림 1]({{site.baseurl}}/img/langgraph/naive-rag-relevant-1.png)



정의한 `Naive RAG` 그래프를 실행하면 아래와 같다.  

```python
# 그래프 실행

from langchain_core.runnables import RunnableConfig

config = RunnableConfig(recursion_limit=20, configurable={'thread_id' : '2'})

inputs = GraphState(question='지금까지 각 월의 평균 기온에 대해 정리해줘')

execute_graph(graph, config, inputs)
# retrieve
# {'context': '<document><content>하거나\x01적었습니다.\n평균기온(℃) 강수량(㎜)\n낮음 비슷 높음 적음 비슷 많음\n※\x01네모\x01박스\x01위:\x01월\x01평균값(℃)/편차(℃),\x01아래:\x01평년(1991~2020년)\x01비슷범위(℃) ※\x01네모\x01박스\x01위:\x01월\x01누적값(㎜),\x01아래:\x01평년(1991~2020년)\x01비슷범위(㎜)\n평균기온\x01확률밀도분포\n▶\x01채색:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(4월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온\x01분포\n▶\x01점선:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(4월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>5</page></document>\n<document><content>평년과\x01비슷하거나\x01적었습니다.\n평균기온(℃) 강수량(㎜)\n낮음 비슷 높음 적음 비슷 많음\n※\x01네모\x01박스\x01위:\x01월\x01평균값(℃)/편차(℃),\x01아래:\x01평년(1991~2020년)\x01비슷범위(℃) ※\x01네모\x01박스\x01위:\x01월\x01누적값(㎜),\x01아래:\x01평년(1991~2020년)\x01비슷범위(㎜)\n평균기온\x01확률밀도분포\n▶\x01채색:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2023년(3월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온\x01분포\n▶\x01점선:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2023년(3월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>5</page></document>\n<document><content>이\x01평년과\x01비슷하거나\x01많았습니다.\n평균기온(℃) 강수량(㎜)\n낮음 비슷 높음 적음 비슷 많음\n※\x01네모\x01박스\x01위:\x01월\x01평균값(℃)/편차(℃),\x01아래:\x01평년(1991~2020년)\x01비슷범위(℃) ※\x01네모\x01박스\x01위:\x01월\x01누적값(㎜),\x01아래:\x01평년(1991~2020년)\x01비슷범위(㎜)\n평균기온\x01확률밀도분포\n▶\x01채색:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(6월\x01평균기온\x012위),\x01(초록)평년\x01월평균기온\x01분포\n▶\x01점선:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(6월\x01평균기온\x012위),\x01(초록)평년\x01월평균기온</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>5</page></document>\n<document><content>년보다\x01낮은\x01기온이\x01지속되다가,\x01이후에는\x01대체로\x01평년\x01수준으로\x01회복되었으나,\x0120~21일에는\x01기온이\x01일시적으로\x01크게\x01올랐습니다.\x01\n※\x01평년\x01비슷범위:\x0117.0~17.6℃\n2025년\x015월\x01평균기온/평균\x01최고기온/평균\x01최저기온\x01(1973년\x01이후\x01전국평균)\n2025년\x015월\n구분\n평균값\x01(℃) 평년값\x01(℃) 평년편차\x01(℃) 순위(상위)\n평균기온 16.8 17.3 -0.5 33위\n평균\x01최고기온 22.4 23.5 -1.1 45위\n평균\x01최저기온 11.5 11.6 -0.1 23위</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf</source><page>1</page></document>\n<document><content>2025년\n2025.\x017.\x014.\x01발간\n6월호\n6월\x01기후\x01동향\n기온\n6월\x01기온\x01시계열\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01막대:\x012025년\x016월\x01전국\x0166개\x01지점의\x01일별\x01(빨강)최고기온\x01범위,\x01(파랑)최저기온\x01범위\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01실선:\x012025년\x016월\x01전국\x0166개\x01지점\x01평균\x01일별\x01(초록)평균기온,\x01(빨강)최고기온,\x01(파랑)최저기온\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01점:\x011973~2025년\x016월\x01전국\x0166개\x01지점\x01기준\x01일별\x01(빨강)최고기온\x01극값,\x01(파랑)최저기온\x01극값</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>1</page></document>\n<document><content>평균기온 22.9 21.4 +1.5 1위\n평균\x01최고기온 28.2 26.7 +1.5 2위\n평균\x01최저기온 18.2 16.8 +1.4 3위\n※\x01전국평균:\x011973년\x01이후부터\x01연속적으로\x01관측한\x01전국\x0162개\x01지점의\x01관측자료를\x01활용((1973~1989년)\x0156개\x01지점,\x01(1990~2025년)\x0162개\x01지점)\n※\x01평년값:\x011991~2020년\x01적용\n1</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>1</page></document>\n<document><content>2025년\x014월\n구분\n평균값\x01(℃) 평년값\x01(℃) 평년편차\x01(℃) 순위(상위)\n평균기온 13.1 12.1 1.0 10위\n평균\x01최고기온 19.7 18.6 1.1 9위\n평균\x01최저기온 6.6 6.0 0.6 18위\n※\x01전국평균:\x011973년\x01이후부터\x01연속적으로\x01관측한\x01전국\x0162개\x01지점의\x01관측자료를\x01활용((1973~1989년)\x0156개\x01지점,\x01(1990~2025년)\x0162개\x01지점)\n※\x01평년값:\x011991~2020년\x01적용\n1</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>1</page></document>\n<document><content>2025년\n2025.\x014.\x013.\x01발간\n3월호\n3월\x01기후\x01동향\n기온\n3월\x01기온\x01시계열\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01막대:\x012025년\x013월\x01전국\x0166개\x01지점의\x01일별\x01(빨강)최고기온\x01범위,\x01(파랑)최저기온\x01범위\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01실선:\x012025년\x013월\x01전국\x0166개\x01지점\x01평균\x01일별\x01(초록)평균기온,\x01(빨강)최고기온,\x01(파랑)최저기온\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01점:\x011973~2025년\x013월\x01전국\x0166개\x01지점\x01기준\x01일별\x01(빨강)최고기온\x01극값,\x01(파랑)최저기온\x01극값</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>1</page></document>\n<document><content>관측자료\n4월\x01해수면\x01온도\x01시계열(℃)\n4월\x01전\x01해상\x01일별\x01평균해수면\x01온도\x01(파랑)\x01최근\x0110년\x01(2015~2024년),\x01(빨강)\x012025년\n2015년\x01이후부터\x01연속적으로\x01관측한\x01해양기상부이\x0111개\x01지점의\x01관측자료를\x01활용\n2025년\x014월\x01평균\x01해수면온도\x01분포 최근\x0110년\x014월\x01평균\x01해수면온도\x01분포\n재분석자료(OISST)\n2025년\x014월\x01평균\x01해수면온도\x01분포 평년(1991~2020년)\x01편차</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>8</page></document>\n<document><content>2025년\n2025.\x015.\x017.\x01발간\n4월호\n4월\x01기후\x01동향\n기온\n4월\x01기온\x01시계열\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01막대:\x012025년\x014월\x01전국\x0166개\x01지점의\x01일별\x01(빨강)최고기온\x01범위,\x01(파랑)최저기온\x01범위\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01실선:\x012025년\x014월\x01전국\x0166개\x01지점\x01평균\x01일별\x01(초록)평균기온,\x01(빨강)최고기온,\x01(파랑)최저기온\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01점:\x011973~2025년\x014월\x01전국\x0166개\x01지점\x01기준\x01일별\x01(빨강)최고기온\x01극값,\x01(파랑)최저기온\x01극값</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>1</page></document>'}
# llm_answer
# {'answer': '주어진 문맥에서 2025년 3월, 4월, 5월, 6월의 평균 기온에 대한 정보가 제공됩니다.\n\n*   **2025년 3월:** 정보가 없습니다.\n*   **2025년 4월:** 평균 기온은 13.1℃ 입니다.\n*   **2025년 5월:** 평균 기온은 16.8℃ 입니다.\n*   **2025년 6월:** 평균 기온은 22.9℃ 입니다.\n\n**출처:**\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf (페이지 1)', 'messages': [('user', '지금까지 각 월의 평균 기온에 대해 정리해줘'), ('assistant', '주어진 문맥에서 2025년 3월, 4월, 5월, 6월의 평균 기온에 대한 정보가 제공됩니다.\n\n*   **2025년 3월:** 정보가 없습니다.\n*   **2025년 4월:** 평균 기온은 13.1℃ 입니다.\n*   **2025년 5월:** 평균 기온은 16.8℃ 입니다.\n*   **2025년 6월:** 평균 기온은 22.9℃ 입니다.\n\n**출처:**\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf (페이지 1)')]}
```  

## Relevant Checker
`Relevant Checker` 는 `RAG` 파이프라인에서 검색된 문서가 살제로 입력 질의와 얼마나 고나련성이 높은지 판별하는 컴포넌트이다. 
검색 단계에서 단순 벡터 유사도만으로 질의와 비슷하지만 의미적으로 무관한 문서가 섞여 들어올 수 있기 때문에, 추가적으로 관련성 판별 과정을 두어 검색 품질을 높인다.  

예제에서는 앞서 구현한 `Naive RAG` 에 `Relevant Checker` 를 추가하여 검색된 문서가 입력 질의와 얼마나 관련성이 높은지 판별하는 과정을 추가한다.  

### Define Relevant Checker Class
관련성 체크를 위한 클래스를 정의한다. 이 클래스는 검색된 문서와 입력 질의를 받아 관련성 점수를 계산하고, 이를 기반으로 문서를 필터링할 수 있다.  

```python
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import PromptTemplate
from enum import Enum


class QuestionScore(BaseModel):
  """
  Binary scroes for relevance checks
  """

  score: str = Field(
      description="relevant or not relevant, Answer 'yes' if the anwser is relevant to the question else answer 'no'"
  )

class AnswerRetrievalScore(BaseModel):
  """
  Binary scores for relevance checks
  """

  score: str = Field(
      description="relevant or not relevant, Answer 'yes', if the answer is relevant to the retrieved document else answer 'no'"
  )

class QuestionRetrievalScore(BaseModel):
  """
  Binary socres for relevance checks
  """

  score: str = Field(
      description="relevant or not relevant, Answer 'yes' if the question is relevant to the retrieved document else answer 'no'"
  )

class EnumOutputStructured(Enum):
  # RETRIEVAL_QUESTION()
  RETRIEVAL_ANSWER = (AnswerRetrievalScore, PromptTemplate(
      template="""
      You are a grader assessing relevance of a retrieved document to a user question.

      Here is the retrieved document:
      {context}

      Here is the answer:
      {answer}

      IF the document contains keyword(s) or semantic meaning related to the user answer, grade it as relevant.

      Give a binary score 'yes' or 'no' score to indicate whether the retrieved document is relevant to the answer.
      """,
      input_variables=["context", "answer"]
  ))
  QUESTION_ANSWER = (QuestionScore, PromptTemplate(
      template="""
      You are a grader assessing whether an answer appropriately addresses the given question.

      Here is the question:
      {question}

      Here is the answer:
      {answer}

      If the answer directly addresses the question and provides relevant information, grade it as relevant.
      Consider both semantic meaning and factual accuracy in your assessment.

      Give a binary score 'yes' or 'no' score to indicate whether the answer is relevant to the question.
      """,
      input_variables=["question", "answer"]
  ))
  QUESTION_RETRIEVAL = (QuestionRetrievalScore, PromptTemplate(
      template="""
      You are a grader assessing whether a retrieved document is relevant th the given question.

      Here is the question:
      {question}

      Here is retrieved document:
      {context}


      If the document contains information that could help answer the question, grade it as relevant.
      Consider both semantic meaning and potential usefulness for answering the question.

      Give a binary score 'yes' or 'no' score to indicate whether the retrieved document is relevant to the question.
      """,
      input_variables=['question', 'context']
  ))

  def __init__(self, output_class, prompt):
    self.output_class = output_class
    self.prompt = prompt


class RelevanceGrader:
  def __init__(self, llm, target_enum : EnumOutputStructured):
    self.llm = llm
    self.target_enum = target_enum

  def create(self):
    llm = self.llm.with_structured_output(self.target_enum.output_class)


    prompt = self.target_enum.prompt
    chain = prompt | llm

    return chain
```  




### Build Graph
구현한 `Relevant Checker` 클래스를 사용해서 `RAG` 구조를 그래프로 정의한다.  

먼저 그래프 정의에 필요한 상태와 노드를 정의한다.  

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

# 상태 정의
class GraphState(TypedDict):
  question: Annotated[str, 'Question']
  context: Annotated[str, 'Context']
  answer: Annotated[str, 'Answer']
  messages: Annotated[list, add_messages]
  relevance: Annotated[str, 'Relevance']

# 관련성 체크 노드
def relevance_check(state: GraphState) -> GraphState:
    question_answer_relevant = RelevanceGrader(llm, EnumOutputStructured.QUESTION_RETRIEVAL).create()

    response = question_answer_relevant.invoke(
        {
            'question' : state['question'],
            'context' : state['context']
        }
    )

    print('===== relevant check =====')
    print(response.score)

    return GraphState(relevance=response.score)


def is_relevant(state: GraphState) -> GraphState:
    return state['relevance']
```  

이제 최종적으로 `Relevant Checker` 가 추가된 `RAG` 구조를 그래프로 정의한다.  

```python
# 그래프 구성
from langgraph.graph import START, END, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from IPython.display import Image, display

graph_builder = StateGraph(GraphState)

graph_builder.add_node('retrieve', retrieve_document)
graph_builder.add_node('relevance_check', relevance_check)
graph_builder.add_node('llm_answer', llm_anwser)

graph_builder.add_edge('retrieve', 'relevance_check')
graph_builder.add_conditional_edges(
    'relevance_check',
    is_relevant,
    {
        'yes': 'llm_answer',
        'no' : 'retrieve'
    }
)

graph_builder.set_entry_point('retrieve')

memory = MemorySaver()

graph = graph_builder.compile(memory)

# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/naive-rag-relevant-2.png)

