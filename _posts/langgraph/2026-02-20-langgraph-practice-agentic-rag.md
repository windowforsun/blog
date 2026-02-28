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

### Build Graph
이제 정의된 툴, 상태, 노드들을 모두 사용해 그래프를 구성한다.  

```python
from langgraph.graph import END, StateGraph, START
from langgraph.prebuilt import ToolNode
from langgraph.prebuilt import tools_condition
from IPython.display import Image, display

graph_builder = StateGraph(AgenticRagState)
graph_builder.add_node('agent', agent)
graph_builder.add_node('retrieve', tool_node)
graph_builder.add_node('rewrite', rewrite)
graph_builder.add_node('generate', generate)

graph_builder.add_edge(START, 'agent')

graph_builder.add_conditional_edges(
    "agent",
    tools_condition,
    {
        'tools' : 'retrieve',
        END : END
    }
)


graph_builder.add_conditional_edges(
    'retrieve',
    grade_documents,
)

graph_builder.add_edge('generate', END)
graph_builder.add_edge('rewrite', 'agent')

graph = graph_builder.compile()


# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass


```  

![그림 1]({{site.baseurl}}/img/langgraph/agentic-rag-1.png)


구현된 `Agentic RAG` 그래프를 사용해 가장 먼저 문서 검색기를 활용할 수 있는 질문을 입력해 본다. 
아래 결과를보면 `pdf_retriever` 도구를 사용해 최종 답변이 출력되는 것을 확인할 수 있다. 

```python
# retrieve 수행

from langchain_core.runnables import RunnableConfig

def execute_graph(graph, config, inputs):
    for event in graph.stream(inputs, config, stream_mode="updates"):
        for k, v in event.items():
            print(k)
            print(v)

config = RunnableConfig(recursion_limit=20, configurable={'thread_id' : '4'})
question = '한국 6월 날씨를 분석해줘'

inputs = inputs = {
    "messages": [
        ("user", question),
    ]
}

execute_graph(graph, config, inputs)
agent
# {'messages': [AIMessage(content='', additional_kwargs={'function_call': {'name': 'pdf_retriever', 'arguments': '{"query": "\\ud55c\\uad6d 6\\uc6d4 \\ub0a0\\uc528"}'}}, response_metadata={'prompt_feedback': {'block_reason': 0, 'safety_ratings': []}, 'finish_reason': 'STOP', 'model_name': 'gemini-2.0-flash', 'safety_ratings': []}, id='run--d98ddf46-70fd-4c6c-abf7-5c5fd227d817-0', tool_calls=[{'name': 'pdf_retriever', 'args': {'query': '한국 6월 날씨'}, 'id': 'a61748b4-7308-4f8c-baef-e3d7d0c33ae3', 'type': 'tool_call'}], usage_metadata={'input_tokens': 71, 'output_tokens': 11, 'total_tokens': 82, 'input_token_details': {'cache_read': 0}})]}
# ===== docs relevant =====
# retrieve
# {'messages': [ToolMessage(content='<document><context>[기온]\x01올해(7.6℃)\x01vs\x01작년(6.9℃)\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\n전국적으로\x01작년보다\x01기온이\x01높았으며,\x01작년\x01대비\x01+0.3~+1.1℃\x01기온\x01분포를\x01보였습니다.</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>3</page></metadata></document>\n\n<document><context>다.\x016월에는\x01우리나라\x01남동쪽에\x01고기압이\x01발달하면서\x01남서풍이\x01주로\x01불어\x01기온이\x01평년보다\x01높은\x01날이\x01많았고,\x01특히\x0127~30일에는\x01북\n태평양고기압\x01가장자리를\x01따라\x01덥고\x01습한\x01공기가\x01유입되고\x01낮\x01동안\x01햇볕이\x01더해지면서\x01폭염과\x01열대야가\x01발생했습니다.\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\n※\x01평년\x01비슷범위:\x0121.1~21.7℃\n2025년\x016월\x01평균기온/평균\x01최고기온/평균\x01최저기온\x01(1973년\x01이후\x01전국평균)\n2025년\x016월\n구분\n평균값\x01(℃) 평년값\x01(℃) 평년편차\x01(℃) 순위(상위)\n평균기온 22.9 21.4 +1.5 1위</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>0</page></metadata></document>\n\n<document><context>7</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>6</page></metadata></document>\n\n<document><context>을\x01보이다가\x0116일에\x01우리나라\x01주변에\x01상층\x01찬\x01기압골이\x01급격하게\x01발달하여\x0116~19일에\x01기온이\x01일시적으로\x01크게\x01떨어졌고,\x01하순에\n는\x01중국\x01내륙의\x01따뜻하고\x01건조한\x01공기가\x01서풍을\x01타고\x01유입되면서\x01고온이\x01지속되었습니다.\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\n※\x01평년\x01비슷범위:\x015.6~6.6℃\n2025년\x013월\x01평균기온/평균\x01최고기온/평균\x01최저기온\x01(1973년\x01이후\x01전국평균)\n2025년\x013월\n구분</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>0</page></metadata></document>\n\n<document><context>[기온]\x01올해(22.9℃)\x01vs\x01작년(22.7℃)\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\n수도권을\x01제외한\x01전국\x01대부분의\x01지역에서\x01기온이\x01작년과\x01비슷하거나\x01높았으며,\x01작년\x01대비\x01-0.3~+0.6℃\x01기온\x01분포를\x01보였습니다.</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>3</page></metadata></document>\n\n<document><context>5월\x01전국\x01평균기온은\x01작년보다\x010.9℃\x01낮았고,\x01강수량은\x01작년과\x01비슷하였습니다.\n[기온]\x01올해(16.8℃)\x01vs\x01작년(17.7℃)\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\n전국적으로\x01작년보다\x01기온이\x01낮았으며,\x01작년\x01대비\x01-1.3~-0.6℃\x01기온\x01분포를\x01보였습니다.</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf</source><page>3</page></metadata></document>\n\n<document><context>\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01※\x011973년\x01이후부터\x01연속적으로\x01관측한\x0162개\x01지점과\x01제주\x014개\x01지점을\x01포함한\x0166개\x01지점의\x01관측자료를\x01활용\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01(1973~1989년)전국\x0156개+제주\x012개,\x01(1990~2025년)전국\x0162개+제주\x014개\n현황\n3월\x01전국\x01평균기온은\x017.6℃로\x01평년(6.1℃)보다\x011.5℃\x01높았고,\x01작년(6.9℃)보다\x010.7℃\x01높았습니다.\x01전반에는\x01대체로\x01평년\x01수준의\x01기온</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>0</page></metadata></document>\n\n<document><context>-----\x01\x01\x01\x01\x01기기기기기후후후후후분분분분분석석석석석정정정정정보보보보보\x01\x01\x01\x01\x0122222000002222255555년년년년년66666월월월월월호호호호호\x01\x01\x01\x01\x01-----\n주요\x01기후요소\x01비교\x01-\x01기온·강수량\n작년\x01비교\n6월\x01전국\x01평균기온은\x01작년보다\x010.2℃\x01높았고,\x01강수량은\x01작년보다\x0156.9mm\x01많았습니다.</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>3</page></metadata></document>\n\n<document><context>한\x01날씨가\x01지속되었습니다.\n하층에서는\x01우리나라\x01남쪽에\x01이동성고기압이\x01느리게\n이동한\x01가운데\x01북쪽으로\x01저기압이\x01통과하면서\x01중국\x01내\n륙의\x01고온\x01건조한\x01공기가\x01강한\x01서풍을\x01타고\x01우리나라\n로\x01유입되었습니다.\x01이에\x01따라\x01기온이\x01크게\x01상승하여\x013\n월\x01일최고기온\x01극값을\x01경신한\x01지점이\x01많았습니다.\n3월\x01하순\x01상대습도는\x01평년(59%)보다\x01낮은\x01날이\x01많았\n으며,\x01특히\x0121~26일에\x01경북지역을\x01중심으로\x01상대습\n도가\x01평년\x01대비\x0115%p\x01이상\x01낮았습니다.\n강수량\n강수\x01현황\n2025년\x013월\x01전국\x01강수량(㎜)과\x01퍼센타일(%ile)</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>1</page></metadata></document>\n\n<document><context>\x01\x01\x01\x01\x01\x01\x01((1973~1989년)\x0156개\x01지점,\x01(1990~2025년)\x0162개\x01지점)\n우리나라\x01월별\x01평균기온\x01평년편차와\x01순위\x01(2024년\x016월\x01~\x012025년\x015월)\n2024년 2025년\n년/월 기준\n6월 7월 8월 9월 10월 11월 12월 1월 2월 3월 4월 5월\n월평균(℃) 22.7 26.2 27.9 24.7 16.1 9.7 1.8 -0.2 -0.5 7.6 13.1 16.8</context><metadata><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf</source><page>4</page></metadata></document>', name='pdf_retriever', id='04ffc045-a1fc-43c5-98ba-530c7e997688', tool_call_id='a61748b4-7308-4f8c-baef-e3d7d0c33ae3')]}
# generate
# {'messages': ['2025년 6월 전국 평균기온은 22.9℃로, 평년값 21.4℃보다 +1.5℃ 높아 1위를 기록했습니다. 6월에는 우리나라 남동쪽에 고기압이 발달하면서 남서풍이 주로 불어 기온이 평년보다 높은 날이 많았고, 특히 27~30일에는 북태평양고기압 가장자리를 따라 덥고 습한 공기가 유입되고 낮 동안 햇볕이 더해지면서 폭염과 열대야가 발생했습니다. 6월 전국 평균기온은 작년보다 0.2℃ 높았고, 강수량은 작년보다 56.9mm 많았습니다.\n\n**출처**\n- /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf']}
```  

이번에는 실시간 검색이 필요한 질문을 입력해 본다. 
아래 결과를보면 `search_llm` 도구를 사용해 최종 답변이 출력되는 것을 확인할 수 있다.


```python
config = RunnableConfig(recursion_limit=20, configurable={'thread_id' : '4'})
question = '현재 미국 수도의 날씨를 알려줘'

inputs = inputs = {
    "messages": [
        ("user", question),
    ]
}

execute_graph(graph, config, inputs)
agent
# {'messages': [AIMessage(content='', additional_kwargs={'function_call': {'name': 'search_llm', 'arguments': '{"question": "\\ud604\\uc7ac \\ubbf8\\uad6d \\uc218\\ub3c4\\uc758 \\ub0a0\\uc528"}'}}, response_metadata={'prompt_feedback': {'block_reason': 0, 'safety_ratings': []}, 'finish_reason': 'STOP', 'model_name': 'gemini-2.0-flash', 'safety_ratings': []}, id='run--b12d24bc-bb8c-4207-80c4-80d203937859-0', tool_calls=[{'name': 'search_weather', 'args': {'question': '현재 미국 수도의 날씨'}, 'id': '7d2d35db-5997-442d-b038-357efd2bee59', 'type': 'tool_call'}], usage_metadata={'input_tokens': 73, 'output_tokens': 12, 'total_tokens': 85, 'input_token_details': {'cache_read': 0}})]}
# [{'title': '워싱턴, DC, 미국 일기예보 및 날씨 | Weather.com', 'link': 'https://weather.com/ko-KR/weather/today/l/Washington+DC+United+States?canonicalCityId=5449cc9af33d6584872016be78d0340d', 'snippet': '워싱턴, DC, 미국. 01:26 EDT 기준. 23°. 대체로 흐림. 최고 32° • 최저 22°. 오늘 워싱턴, DC, 미국의 날씨 예보. 체감온도23°. 05:55. 20:32. 최고/최저. --/22°.', 'position': 1}, {'title': '워싱턴, DC 현재 날씨 - AccuWeather', 'link': 'https://www.accuweather.com/ko/us/washington/20006/current-weather/327659', 'snippet': '현재 기상 ; RealFeel®. 82° ; 바람. 북북동 1mi/h ; 돌풍. 1mi/h ; 습도. 96% ; 이슬점. 72° F.', 'position': 2}, {'title': '전국 현재 날씨 - AccuWeather', 'link': 'https://www.accuweather.com/ko/us/united-states-weather', 'snippet': '뉴욕 76° 댈러스 78° 덴버 77° 로스앤젤레스 64° 맨해튼 76° 보스턴 79° 브롱크스 75° 브루클린 79° 산호세 58° 샌디에이고 66° 샌안토니오 81° 시카고 80° 애틀랜타 ...', 'attributes': {'Missing': '수도 | Show results with:수도'}, 'position': 3}, {'title': '댈러스, TX, 미국 일기예보 및 날씨 | Weather.com', 'link': 'https://weather.com/ko-KR/weather/today/l/Dallas+TX+United+States?canonicalCityId=3bef7f8bb00708145ceebe387a6de1b2098d40101d65836dd79c94d1dfe0c20b', 'snippet': '오늘 댈러스, TX, 미국의 날씨 예보 ; 최고/최저. 29°/20° ; 바람. 10 km/h ; 습도. 72% ; 이슬점. 18° ; 기압. 1017.3 mb.', 'position': 4}, {'title': '워싱턴 일기 예보', 'link': 'https://ko.meteotrend.com/forecast/us/washington/', 'snippet': '새벽00:01~06:00, 부분적으로 흐린 날씨 +23...+26 °C 공기의 온도가 내려갑니다 부분적으로 흐린 날씨. 서양의. 바람: 남실바람, 서양의, 속도 2-3 초당 미터', 'position': 5}, {'title': '미국, 워싱턴, 시애틀 일기예보 | MSN 날씨', 'link': 'https://www.msn.com/ko-kr/weather/forecast/in-Seattle,WA', 'snippet': '최소 2 시간 동안 비나 눈이 내리지 않습니다. 지도 열기. 상태 및 활동. 야외. 매우 좋지 않음. 옷. 가벼운 자켓. 냉풍. 안전. 우산. 필요 없음. 자외선 지수. 낮음 ...', 'position': 6}, {'title': '일주일 동안 워싱턴의 날씨 - 일기 예보 및 기상 조건', 'link': 'https://ko.meteocast.net/week-forecast/us/washington_8/', 'snippet': '워싱턴의 현재 시간: ; 새벽00:01~06:00, 흐린 · + · 34 °C ; 아침06:01~12:00, 맑은 하늘 · + · 35 °C ; 오후12:01~18:00, 맑은 하늘 · + · +40 °C ...', 'position': 7}, {'title': '워싱턴 DC 14일 날씨 예측', 'link': 'https://meteodays.com/ko/weather/14days/washington', 'snippet': '상세한 14일간의 예보: 매일의 날씨, 기온, 강수 확률, 풍속, 습도 등의 상세 정보. ; 최신 날씨 데이터: 실시간으로 업데이트되는 정확한 날씨 정보. ; 전국적인 범위: 주요 ...', 'position': 8}, {'title': '워싱턴 주의 달별 기후, 평균 온도 (미국) - Weather Spark', 'link': 'https://ko.weatherspark.com/countries/US/WA', 'snippet': '워싱턴 주의 연중 기후 및 평균 날씨 미국 ; 높은 · 9°C · 10°C · 13°C · 16°C ; 낮은 · 3°C · 4°C · 5°C · 7°C ; 맑은 하늘 · 29%, 31%, 33%, 39% ...', 'position': 9}, {'title': '미국, 워싱턴, 시애틀 일기예보 | MSN 날씨', 'link': 'https://www.msn.com/ko-kr/weather/forecast/in-%EC%8B%9C%EC%95%A0%ED%8B%80,%EC%9B%8C%EC%8B%B1%ED%84%B4', 'snippet': 'MSN 날씨과(와) 함께 미국, 워싱턴, 시애틀의 오늘, 오늘 밤, 내일에 대한 ... 현재 날씨. 5:00 PM. 맑음 19°C. 체감 18°. 화창. 대기질. 37 · 바람. 4 km/h · 습도. 73 ...', 'position': 10}]
# ===== docs relevant =====
# retrieve
# {'messages': [ToolMessage(content='[{"title": "워싱턴, DC, 미국 일기예보 및 날씨 | Weather.com", "link": "https://weather.com/ko-KR/weather/today/l/Washington+DC+United+States?canonicalCityId=5449cc9af33d6584872016be78d0340d", "snippet": "워싱턴, DC, 미국. 01:26 EDT 기준. 23°. 대체로 흐림. 최고 32° • 최저 22°. 오늘 워싱턴, DC, 미국의 날씨 예보. 체감온도23°. 05:55. 20:32. 최고/최저. --/22°.", "position": 1}, {"title": "워싱턴, DC 현재 날씨 - AccuWeather", "link": "https://www.accuweather.com/ko/us/washington/20006/current-weather/327659", "snippet": "현재 기상 ; RealFeel®. 82° ; 바람. 북북동 1mi/h ; 돌풍. 1mi/h ; 습도. 96% ; 이슬점. 72° F.", "position": 2}, {"title": "전국 현재 날씨 - AccuWeather", "link": "https://www.accuweather.com/ko/us/united-states-weather", "snippet": "뉴욕 76° 댈러스 78° 덴버 77° 로스앤젤레스 64° 맨해튼 76° 보스턴 79° 브롱크스 75° 브루클린 79° 산호세 58° 샌디에이고 66° 샌안토니오 81° 시카고 80° 애틀랜타 ...", "attributes": {"Missing": "수도 | Show results with:수도"}, "position": 3}, {"title": "댈러스, TX, 미국 일기예보 및 날씨 | Weather.com", "link": "https://weather.com/ko-KR/weather/today/l/Dallas+TX+United+States?canonicalCityId=3bef7f8bb00708145ceebe387a6de1b2098d40101d65836dd79c94d1dfe0c20b", "snippet": "오늘 댈러스, TX, 미국의 날씨 예보 ; 최고/최저. 29°/20° ; 바람. 10 km/h ; 습도. 72% ; 이슬점. 18° ; 기압. 1017.3 mb.", "position": 4}, {"title": "워싱턴 일기 예보", "link": "https://ko.meteotrend.com/forecast/us/washington/", "snippet": "새벽00:01~06:00, 부분적으로 흐린 날씨 +23...+26 °C 공기의 온도가 내려갑니다 부분적으로 흐린 날씨. 서양의. 바람: 남실바람, 서양의, 속도 2-3 초당 미터", "position": 5}, {"title": "미국, 워싱턴, 시애틀 일기예보 | MSN 날씨", "link": "https://www.msn.com/ko-kr/weather/forecast/in-Seattle,WA", "snippet": "최소 2 시간 동안 비나 눈이 내리지 않습니다. 지도 열기. 상태 및 활동. 야외. 매우 좋지 않음. 옷. 가벼운 자켓. 냉풍. 안전. 우산. 필요 없음. 자외선 지수. 낮음 ...", "position": 6}, {"title": "일주일 동안 워싱턴의 날씨 - 일기 예보 및 기상 조건", "link": "https://ko.meteocast.net/week-forecast/us/washington_8/", "snippet": "워싱턴의 현재 시간: ; 새벽00:01~06:00, 흐린 · + · 34 °C ; 아침06:01~12:00, 맑은 하늘 · + · 35 °C ; 오후12:01~18:00, 맑은 하늘 · + · +40 °C ...", "position": 7}, {"title": "워싱턴 DC 14일 날씨 예측", "link": "https://meteodays.com/ko/weather/14days/washington", "snippet": "상세한 14일간의 예보: 매일의 날씨, 기온, 강수 확률, 풍속, 습도 등의 상세 정보. ; 최신 날씨 데이터: 실시간으로 업데이트되는 정확한 날씨 정보. ; 전국적인 범위: 주요 ...", "position": 8}, {"title": "워싱턴 주의 달별 기후, 평균 온도 (미국) - Weather Spark", "link": "https://ko.weatherspark.com/countries/US/WA", "snippet": "워싱턴 주의 연중 기후 및 평균 날씨 미국 ; 높은 · 9°C · 10°C · 13°C · 16°C ; 낮은 · 3°C · 4°C · 5°C · 7°C ; 맑은 하늘 · 29%, 31%, 33%, 39% ...", "position": 9}, {"title": "미국, 워싱턴, 시애틀 일기예보 | MSN 날씨", "link": "https://www.msn.com/ko-kr/weather/forecast/in-%EC%8B%9C%EC%95%A0%ED%8B%80,%EC%9B%8C%EC%8B%B1%ED%84%B4", "snippet": "MSN 날씨과(와) 함께 미국, 워싱턴, 시애틀의 오늘, 오늘 밤, 내일에 대한 ... 현재 날씨. 5:00 PM. 맑음 19°C. 체감 18°. 화창. 대기질. 37 · 바람. 4 km/h · 습도. 73 ...", "position": 10}]', name='search_weather', id='670e37c3-f7ed-4d43-9cab-32f01c3611ab', tool_call_id='7d2d35db-5997-442d-b038-357efd2bee59')]}
# generate
# {'messages': ['현재 워싱턴 DC의 날씨는 23°이며 대체로 흐립니다. 최고 기온은 32°, 최저 기온은 22°입니다. 체감온도는 23°입니다.\n        \n        **출처:**\n        * weather.com']}
```  

`llm` 모델에서 답변할 수 있는 일반적인 질문을 입력해 본다.  
아래 결과를보면 `llm_tool` 도구를 사용해 최종 답변이 출력되는 것을 확인할 수 있다.

```python

config = RunnableConfig(recursion_limit=20, configurable={'thread_id' : '4'})
question = '미국 수도가 어딘지 알려줘'

inputs = inputs = {
    "messages": [
        ("user", question),
    ]
}

execute_graph(graph, config, inputs)
# agent
# {'messages': [AIMessage(content='', additional_kwargs={'function_call': {'name': 'llm_tool', 'arguments': '{"question": "What is the capital of the US?"}'}}, response_metadata={'prompt_feedback': {'block_reason': 0, 'safety_ratings': []}, 'finish_reason': 'STOP', 'model_name': 'gemini-2.0-flash', 'safety_ratings': []}, id='run--c84502c0-74b8-4925-8fbf-8f2081572737-0', tool_calls=[{'name': 'llm_tool', 'args': {'question': 'What is the capital of the US?'}, 'id': 'ba9360fc-ef67-45a9-96d2-70aaa4417728', 'type': 'tool_call'}], usage_metadata={'input_tokens': 72, 'output_tokens': 13, 'total_tokens': 85, 'input_token_details': {'cache_read': 0}})]}
# ===== docs relevant =====
# retrieve
# {'messages': [ToolMessage(content='The capital of the US is Washington, D.C.', name='llm_tool', id='f120f04e-b0d5-4d9d-9332-fa2ac9a18b64', tool_call_id='ba9360fc-ef67-45a9-96d2-70aaa4417728')]}
# generate
# {'messages': ['워싱턴 D.C.는 미국의 수도입니다.\n\n**출처**\n- 해당 맥락에서 제공됨']}
```  
