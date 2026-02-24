--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph RAG Structure"
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

## Agentic RAG
`Agentic RAG` 는 전통적인 `RAG` 시스템에서 `Agent` 의 역할을 추가하여, 
사용자의 질문에 대해 더 깊이 있는 답변을 제공하는 시스템이다. 
기존 `RAG` 에서는 단순 검색과 생성의 파이프라인이 중심이었다면, 
`Agentic RAG` 는 복수의 에이전트들이 협력하며, 다양한 살태와 흐름 제어를 통해 더 복잡한 정보 검색 및 생성 작업을 수생할 수 있다. 

`LangGraph` 기반의 `Agentic RAG` 의 주요 특징은 아래와 같다. 

- `Agentic` 구조 : 단일 `LLM` 에 의존하지 않고, 여러 `Agent`(도구, 함수, `LLM` 등)가 각자 역할을 나누어 협력한다. 각 `Agent` 가 자신의 상태와 맥락에 따라 행동을 결정한다. 
- 동적 워크플로우 : 질의의 난이도, 유형, 중간 결과에 따라 워크플로우가 동적으로 변화한다. 예를 들어 검색 결과가 부족할 때 추가 검색, 요약, 재질문 등 자동 분기가 가능하다. 
- 상태 기반 처리 : 각 단계의 결과와 상태를 기록, 필요 시 이전 단계로 되돌리거나, 추가 작업을 수행할 수 있다. 이를 통해 복잡한 질의에도 유연하게 대응할 수 있다. 
- 모듈화와 확장성 : 각 노드를 별도의 함수/모듈로 분리하여 재사용 및 확장이 용이하다. 새로운 `Agent` 도 쉽게 추가할 수 있다.  

이렇게 `LangGraph` 를 활용한 `Agentic RAG` 는 기존 `RAG` 의 한계르 넘어, 
복잡한 질의와 다양한 처리 흐름을 에이전트와 그래프 기반 상태 관리로 더 유연하고 강력하게 구현할 수 있다. 
이 구조는 확장성과 유지보수가 뛰어나며, 다양한 도메인에 맞춘 맞춤형 `RAG` 시스템 개발에 매우 적합하다.  

본 포스팅에서 진행할 예제는 [이전 예제]({{site.baseurl}}{% link _posts/2026-02-01-langgraph-practice-naive-rag-relevant.md %})
의 내용을 포함하고 하고 있으므로 이전 내용의 숙지가 필요하다.  


### create_retriever_tool
`create_retriever_tool` 는 `LangChain` 함수로 기존의 `retriever` 를 도구(`Tool`)로 래핑하여, 
에이전트가 쉽게 사용할 수 있게 해주는 유틸리티이다. 
기존 검색기를 `Tool` 객체로 변환하여, 
`LangChain` 의 `Agent` 가 여러 도구 중 하나로 선택/활용 하게 만든다. 
`Tool` 로 래핑된 검색기는, `Agent` 의 계획(`Planning`)과 행동(`Action`) 과정에서 자연스럽게 호출된다. 

`create_retriever_tool` 를 사용한 경우와 그렇지 않은 경우를 비교해 정리하면 아래와 같다.  

| 구분             | `create_retriever_tool` 사용 | 미사용(직접 호출)    |
|------------------|-----------------------------|---------------------|
| 통합성           | Agent 워크플로우에 통합      | 개별 기능 호출       |
| 선택적 사용      | Agent가 필요할 때만 사용     | 항상 직접 호출 필요  |
| 복합 도구 조합   | 여러 도구와 조합 가능        | 단일 기능 중심       |
| 계획 및 판단     | Agent가 자연어 설명 기반 선택| 수동 코드 제어       |


이전 예제에서 구현했던 `DemoRetrievalChain` 를 사용해 일반 `retriever` 를 
`create_retriever_tool` 를 사용해 툴로 변환한다.  

```python
from langchain_core.tools.retriever import create_retriever_tool
from langchain_core.prompts import PromptTemplate

# retriever 생성
file_list = [
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf'
]

collection_name='weather-docs-6'
persist_directory = f"/content/drive/MyDrive/Colab Notebooks/data/vectorstore/{collection_name}"

pdf = DemoRetrievalChain(
    source_uri=file_list,
    embeddings=hf_embeddings,
    llm=llm,
    collection_name=collection_name,
    persist_directory=persist_directory
).create_chain()

pdf_retriever = pdf.retriever
pdf_chain = pdf.chain


# retriever를 Tool로 래핑
retriever_tool = create_retriever_tool(
    pdf_retriever,
    "pdf_retriever",
    """
    It contains useful information on 2025 03-06 Korea Climate Analysis and Weather Information
    """,
    document_prompt=PromptTemplate.from_template(
        "<document><context>{page_content}</context><metadata><source>{source}</source><page>{page}</page></metadata></document>"
    )
)
```  

### Tools
`Agentic RAG` 에서는 다양한 도구(`Tool`)를 사용하여, 
사용자의 질문에 대한 답변을 생성한다. 
예제에서 사용할 도구는 문서 검색을 위한 `retriever_tool` 와 
일반적인 `llm` 에서 답변할 수 있는 `llm_tool` 그리고 최신 정보 검색을 위한 `search_tool` 이다.  

```python
from langchain_core.tools import tool
from langchain_community.utilities import GoogleSerperAPIWrapper
import json
from langchain_google_genai import ChatGoogleGenerativeAI
import os

os.environ["SERPER_API_KEY"] = "api_key"
os.environ["GOOGLE_API_KEY"] = "api key"


@tool
def llm_tool(question: str):
    """
    normal question
    """
    llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

    return llm.invoke(question).content



@tool
def search_tool(question: str):
    """
    if need realtime information
    """
    web_search_tool = GoogleSerperAPIWrapper()

    search_query = question

    search_result = web_search_tool.results(search_query)['organic']

    return search_result
```  

정의된 모든 `tool` 을 사용해 `ToolNode` 를 생성한다.  

```python
tools = [retriever_tool, search_tool, llm_tool]
tool_node = ToolNode(
    tools=tools,
)
```  

### Node And GraphState
그래프에서 사용할 상태와 노드 함수를 정의한다. 
먼저 예제 그래프에서 사용할 상태 객체는 아래와 같다.  

```python
from typing import Annotated, Sequence, TypedDict
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

class AgenticRagState(TypedDict):
  messages: Annotated[Sequence[BaseMessage], add_messages]
```  

이제 위 상태를 사용하는 각 노드를 정의한다. 
노드는 기본적으로 필요한 노드를 포함해서, 
질문의 검색 결과와 질문에 대한 관령성 펑가 노드와 
질문을 보다 검색기에서 검색하기 좋은 형태로 변환하는 노드로 구성된다.   

```python
from typing import Literal
from langchain import hub
from langchain_core.messages import HumanMessage
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import PromptTemplate
from pydantic import BaseModel, Field
from langchain.chat_models import init_chat_model
import os

def agent(state):
  messages = state['messages']
  llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

  llm = llm.bind_tools(tools)

  response = llm.invoke(messages)

  return {'messages' : [response]}


class grade(BaseModel):
  """
  A binary score for relevance checks
  """

  binary_score: str = Field(
      description="Response 'yes' if the document is relevant to the question or 'no' if it is not."
  )

def grade_documents(state) -> Literal["generate", "rewrite"]:
  llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

  llm_with_tool = llm.with_structured_output(grade)

  prompt = PromptTemplate(
      template="""
      You are grader assessing relevance of retrieved document to a user question.

      Here is the retrieved document:
      {context}

      Here is the user question:
      {question}

      If the document contains keyword(s) or semantic meaning related to the user question, grade it as relevant.
      Give a binary score 'yes' or 'no' socre to indicate whether the document is relevant to the question.
      """,
      input_variables=["context", "question"]
  )

  chain = prompt | llm_with_tool

  messages = state["messages"]

  last_messages = messages[-1]

  question = messages[0].content

  retrieved_docs = last_messages.content

  scored_result = chain.invoke({
      'question' : question,
      'context' : retrieved_docs
  })

  score = scored_result.binary_score

  if score == 'yes':
    print('===== docs relevant =====')
    return 'generate'
  else:
    print('===== docs not relevant =====')
    return 'rewrite'

def rewrite(state):
  messages = state['messages']
  question = messages[0].content

  prompt = PromptTemplate.from_template(
      template="""
      Look at the input and try to reason about the underlying semantic intent and meaning

      Here is the initial question:
      {question}

      Formulate an improved question:
      """
  )

  llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

  chain = prompt | llm
  response = chain.invoke({'question' : question})

  return {"messages": [response]}

def generate(state):
  messages = state['messages']
  question = messages[0].content

  docs = messages[-1].content
  response = pdf_chain.invoke(
      {
          'question' : question,
          'context' : docs,
          'chat_history' : []
      }
  )

  return {"messages": [response]}
```  
