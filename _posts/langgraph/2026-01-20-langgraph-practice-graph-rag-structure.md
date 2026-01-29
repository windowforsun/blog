--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph RAG Structure"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LangGraph 에서 RAG 시스템을 구축하는 구조에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - RAG
    - VectorStore
    - Retrieval
toc: true
use_math: true
---  

## LangGraph RAG Structure
`LangGraph` 는 복잡한 `AI Workflow` 를 그래프 형태로 구축할 수 있도록 지원하는 프레임워크이다. 
각 워크플로우의 구성 요소(노드)를 명확하게 분리하고, 이들 간의 데이터 흐름(엣지)을 정의함으로써, 유연하고 확장 가능한 `AI Workflow` 를 구축할 수 있다.  

이러한 `LangGraph` 의 특징을 사용해 `RAG (Retrieval-Augmented Generation)` 시스템을 구축하면 복잡하고 다양한/추가적인 기능을 가진 
워크플로우를 좀 더 구조적으로 구현할 수 있다. 

- 모둘화/재사용성 : 각 역할별 노드가 분리되어, 원하는 부분만 독립적으로 수행 및 재사용 가능
- 확장성 : 새로운 기능(추가 검색, 평가 노드 등)을 손쉽게 추가 가능
- 유연성 : 다양한 `RAG` 전략을 조합하여 실험적 워크플로우 구축 가능
- 가독성과 유지보수성 : 그래프 기반 시각화로 시스템 구조 파악 및 유지보수가 쉬움

이렇게 `LangGraph` 를 활용하면 `RAG` 시스템을 단순한 파이프라인이 아니라, 
각 기능 단위를 모듈화하여 유연하게 조합할 수 있다. 
이를 통해 복잡한 요구사항에도 대응하고, 새로운 아이디어나 전략을 빠르게 실험할 수 있는 강력한 `RAG` 플랫폼을 구축할 수 있다.  

본 포스팅에서는 실제 세부 구현은 다루지 않는다. 
`LangGraph` 사용해 `RAG` 를 구축할 때 어떠한 구조와 흐름으로 구성할 수 있는지에 대해 알아보고자 한다.  


### Normal RAG
가장 먼저 기본적인 `RAG` 시스템을 구축해 본다. 
`Retrieval` 을 통해 문서를 검색하고 이를 `llm` 에 던져 결과를 만들어 낸다. 
그리고 관련성 검증을 한 뒤 관련성이 낮은 경우 재검색을 수행하는 흐름의 그래프이다.  

먼저 그래프에서 사용할 상태와 노드로 사용할 각 함수를 정의한다.  

```python
# 상태와 함수 노드 정의

from typing import TypedDict, Annotated, List
from langchain_core.documents import Document
import operator

class GraphState(TypedDict):
  context: Annotated[List[Document], operator.add]
  answer: Annotated[List[Document], operator.add]
  question: Annotated[str, operator.add]
  binary_score: Annotated[str, operator.add]

def retrieve(state: GraphState) -> GraphState:
  # retrieve 검색
  documents = [Document(page_content="retrieve 검색 결과")]
  return GraphState(context=documents)

def rewrite_question(state: GraphState) -> GraphState:
  # 재검색을 위해 쿼리 재작성
  documents = [Document(page_content="질문 재작성")]
  return GraphState(context=documents)

def llm_llama_execute(state: GraphState) -> GraphState:
  # llama llm 실행
  answer = [Document(page_content="llama 답변")]
  return GraphState(answer=answer)

def llm_gemini_execute(state: GraphState) -> GraphState:
  # gemini llm 실행
  answer = [Document(page_content="gemini 답변")]
  return GraphState(answer=answer)

def relevance_check(state: GraphState) -> GraphState:
  # 관련성 확인
  binary_score = "관련성 점수"
  return GraphState(binary_score=binary_score)

def sum_up(state: GraphState) -> GraphState:
  # 결과 종합
  answer = [Document(page_content="종합 답변")]
  return GraphState(answer=answer)

def handle_error(state: GraphState) -> GraphState:
  # 에러 처리
  return GraphState(context=[], answer=[], question="", binary_score="")


def decision(state: GraphState) -> str:
  # 결과 종합 및 판단
  if random.randint(1, 3) % 3 == 0:
    decision = "ok"
  else:
    decision = "research"
  return decision
```  

그리고 정의된 내용들을 바탕으로 그래프를 정의하고 컴파일해 구조를 살펴본다.  

```python
# 일반 rag 그래프 정의
# 사용자 질의 -> conventional rag -> 멀티 LLM -> 관련성 확인 -> 종합 -> 재검색 or 종료 

from langgraph.graph import START, END, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from IPython.display import Image, display

graph_builder = StateGraph(GraphState)

# 노드 추가
graph_builder.add_node("retrieve rag", retrieve)
graph_builder.add_node("rewrite question", rewrite_question)
graph_builder.add_node("llama", llm_llama_execute)
graph_builder.add_node("gemini", llm_gemini_execute)
graph_builder.add_node("llama relevance check", relevance_check)
graph_builder.add_node("gemini relevance check", relevance_check)
graph_builder.add_node("result aggregation", sum_up)

# 노드 연결
# graph_builder.add_edge(START, "문서 검색")
graph_builder.add_edge("retrieve rag", "llama")
graph_builder.add_edge("retrieve rag", "gemini")
graph_builder.add_edge("llama", "llama relevance check")
graph_builder.add_edge("gemini", "gemini relevance check")
graph_builder.add_edge("llama relevance check", "result aggregation")
graph_builder.add_edge("gemini relevance check", "result aggregation")
graph_builder.add_edge("rewrite question", "retrieve rag")

# 재검색에 대한 조건부 엣지 추가
graph_builder.add_conditional_edges(
    source="result aggregation",
    path=decision,
    path_map={
        "research": "rewrite question",
        "ok": END
    }
)

# 시작점 설정 및 컴파일
graph_builder.set_entry_point("retrieve rag")
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)


# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/graph-rag-intro-1.png)



### RAG with SQL
이번에는 `Retrieval` 의 검색과 데이터베이스에서 `SQL` 을 통해 검색하는 흐름을 추가해 본다.  
`SQL` 에서 사용할 추가 상태와 함수 노드를 정의한다.  

```python
# sql 함수 노드 추가 정의

class GraphState(TypedDict):
  context: Annotated[List[Document], operator.add]
  answer: Annotated[List[Document], operator.add]
  question: Annotated[str, operator.add]
  sql_query: Annotated[str, operator.add]
  binary_score: Annotated[str, operator.add]

def get_table_info(state: GraphState) -> GraphState:
  table_info = [Document(page_content="테이블 정보")]
  return GraphState(context=table_info)

def generate_sql_query(state: GraphState) -> GraphState:
  sql_query = "SQL 쿼리"
  return GraphState(sql_query=sql_query)

def execute_sql_query(state: GraphState) -> GraphState:
  sql_result = [Document(page_content="SQL 결과")]
  return GraphState(context=sql_result)

def validate_sql_query(state: GraphState) -> GraphState:
  binary_score = "SQL 결과 검증"
  return GraphState(binary_score=binary_score)

def rewrite_query(state: GraphState) -> GraphState:
  # 재검색을 위해 쿼리 재작성
  documents = [Document(page_content="Query 재작성")]
  return GraphState(context=documents)

def search_on_web(state: GraphState) -> GraphState:
  # search web
  documents = state.get('context', [])
  searched_documents = [Document(page_content="웹 검색 문서")]
  documents.extend(searched_documents)
  return GraphState(context=documents)

def middle_node(state: GraphState) -> GraphState:
  return state;

def agg_search_result(state: GraphState) -> GraphState:
  # 각 검색 결과 종합
  return state;

def decision_sql(state: GraphState) -> str:
  # sql 결과 검증 결과
  rand = random.randint(1, 9) 
  if rand % 3 == 1:
    decision = "query error"
  elif rand % 3 == 2:
    decision = "context error"
  else:
    decision = "ok"
  return decision

```  

그리고 위에서 정의한 상태와 함수 노드를 바탕으로 그래프를 정의하고 컴파일한다.  

```python
# sql rag 그래프 정의

graph_builder = StateGraph(GraphState)


# sql 노드 추가
graph_builder.add_node("retrieve rag", retrieve)
graph_builder.add_node("rewrite query", rewrite_query)
graph_builder.add_node("rewrite question", rewrite_question)
graph_builder.add_node("table info", get_table_info)
graph_builder.add_node("generate sql", generate_sql_query)
graph_builder.add_node("execute sql", execute_sql_query)
graph_builder.add_node("validate sql", validate_sql_query)
graph_builder.add_node("llama", llm_llama_execute)
graph_builder.add_node("gemini", llm_gemini_execute)
graph_builder.add_node("llama relevance check", relevance_check)
graph_builder.add_node("gemini relevance check", relevance_check)
graph_builder.add_node("result aggregation", sum_up)
graph_builder.add_node("middle node", middle_node)

# sql 노드 연결
graph_builder.add_edge("retrieve rag", "table info")
graph_builder.add_edge("table info", "generate sql")
graph_builder.add_edge("generate sql", "execute sql")
graph_builder.add_edge("execute sql", "validate sql")
# 재검색에 대한 조건부 엣지 추가
graph_builder.add_conditional_edges(
    "validate sql",
    decision_sql,
    {
        "query error": "rewrite query",
        "context error": "rewrite question",
        "ok": "middle node"
    }
)
graph_builder.add_edge('rewrite query', 'execute sql')
graph_builder.add_edge('rewrite question', 'rewrite query')
graph_builder.add_edge('middle node', 'gemini')
graph_builder.add_edge('middle node', 'llama')
graph_builder.add_edge('gemini', 'gemini relevance check')
graph_builder.add_edge('llama', 'llama relevance check')
graph_builder.add_edge('gemini relevance check', 'result aggregation')
graph_builder.add_edge('llama relevance check', 'result aggregation')
graph_builder.add_edge('result aggregation', END)


# 시작점 설정 및 컴파일
graph_builder.set_entry_point("retrieve rag")
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)


# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```   

![그림 1]({{site.baseurl}}/img/langgraph/graph-rag-intro-2.png)




---  
## Reference
[LangGraph-Building-Graphs](https://langchain-opentutorial.gitbook.io/langchain-opentutorial/17-langgraph/02-structures/01-langgraph-building-graphs)  


