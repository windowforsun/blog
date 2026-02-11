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
