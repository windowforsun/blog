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


## Web Search
앞서 구현한 [LangGraph RAG]({{site.baseurl}}{% link _posts/langgraph/2026-02-01-langgraph-practice-naive-rag-relevant.md %})
구성에서 관련성 검증에 실패한 경우, 
`Web Search` 를 통해 추가적인 문서를 검색하고, 이를 다시 `llm` 에 던져 답변을 생성하는 흐름을 구현해 본다. 

웹 검색으로는 `GoogleSerperAPIWrapper` 를 사용한다. 
웹 검색 노드 함수를 구현하면 아래와 같다.  

```python
# 웹 검색 노드 추가
from langchain_community.utilities import GoogleSerperAPIWrapper
import json

os.environ["SERPER_API_KEY"] = "api key"

def web_search(state: GraphState) -> GraphState:
  web_search_tool = GoogleSerperAPIWrapper()

  search_query = state['question']

  search_result = web_search_tool.results(search_query)['organic']
  print(search_result)

  return GraphState(context="\n".join(json.dumps(search_result)))
```  

이제 이전에 정의했던 내용들에 웹 검색 노드를 추가하여 전체 그래프를 구성한다. 

```python
# 그래프 구성
from langgraph.graph import START, END, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from IPython.display import Image, display

graph_builder = StateGraph(GraphState)

graph_builder.add_node('retrieve', retrieve_document)
graph_builder.add_node('relevance_check', relevance_check)
graph_builder.add_node('llm_answer', llm_anwser)
graph_builder.add_node('web_search', web_search)

graph_builder.add_edge('retrieve', 'relevance_check')
graph_builder.add_conditional_edges(
    'relevance_check',
    is_relevant,
    {
        'yes': 'llm_answer',
        'no' : 'web_search'
    }
)

graph_builder.add_edge('web_search', 'llm_answer')
graph_builder.set_entry_point('retrieve')

memory = MemorySaver()

graph = graph_builder.compile(memory)



# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/web-search-query-rewrite-1.png)
