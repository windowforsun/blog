--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Agent Memory and Stream"
header:
  overlay_image: /img/langchain-bg-2.jpg
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

## LangGraph ToolNode
`ToolNode` 는 `LangGraph` 에서 `LLM` 만으로 처리하기 힘든 외부 기능/도구를
그래프 워크플로우에 결합할 때 사용하는 노드타입이다. 
외부 `API` 호출, 데이터베이스 질의, 계산, 웹 검색 등 다양한 `실행형 도구` 를 `LangChain` 의 툴 시스템과 결합하여 
자동화 워크플로우에 삽입할 수 있게 한다. 
이는 메시지 목록이 포함된 그래프 상태를 입력으로 받아 도구 호출 결과로 상태를 업데이트하는 `LangChain Runnable` 객체이다. 
사전 구축된 `Agnet` 와 즉시 사용할 수 있도록 설계되어 있고, 
상태에 적절한 리듀서가 있는 `messages` 키가 포함된 경우 모든 `StateGraph` 와 함께 작동할 수 있다.  

예제는 웹 검색 툴과 파이썬 코드 생성 툴을 사용한다.  

```python
from langchain_core.messages import AIMessage
from langchain_core.tools import tool
from langchain_experimental.tools.python.tool import PythonAstREPLTool
from typing import List, Dict
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_community.utilities import GoogleSerperAPIWrapper

os.environ["SERPER_API_KEY"] = "api key"


@tool
def python_code_interpreter(code: str):
  """ call to execute python code. """
  return PythonAstREPLTool().invoke(code)

@tool
def search_web(query: str):
  """ search web for given query """
  return GoogleSerperAPIWrapper().results(query)

tools = [search_web, python_code_interpreter]
tool_node = ToolNode(tools)
```  

정의된 툴은 `ToolNode` 객체 자체를 사용해서 아래와 같이 수동으로 호출할 수 있다.  

```python
message_with_single_tool_call = AIMessage(
    content="",
    tool_calls=[
        {
            "name" : "search_web",
            "args" : {"query": "langgraph"},
            "id": "tool_call_id_1",
            "type" : "tool_call"
        }
    ]
)

tool_node.invoke({"messages" : [message_with_single_tool_call]})
# {'messages': [ToolMessage(content='{"searchParameters": {"q": "langgraph", "gl": "us", "hl": "en", "type": "search", "num": 10, "engine": "google"}, "organic": [{"title": "LangGraph - LangChain", "link": "https://www.langchain.com/langgraph", "snippet": "LangGraph sets the foundation for how we can build and scale AI workloads — from conversational agents, complex task automation, to custom LLM-backed ...", "sitelinks": [{"title": "LangGraph Academy Course", "link": "https://academy.langchain.com/courses/intro-to-langgraph"}, {"title": "Built with LangGraph", "link": "https://www.langchain.com/built-with-langgraph"}], "position": 1}, {"title": "langchain-ai/langgraph: Build resilient language agents as graphs.", "link": "https://github.com/langchain-ai/langgraph", "snippet": "LangGraph is a low-level orchestration framework for building, managing, and deploying long-running, stateful agents.", "position": 2}, {"title": "Learn LangGraph basics - Overview", "link": "https://langchain-ai.github.io/langgraph/concepts/why-langgraph/", "snippet": "LangGraph is built for developers who want to build powerful, adaptable AI agents. Developers choose LangGraph for: Reliability and controllability. Steer agent ...", "position": 3}, {"title": "What is LangGraph? - IBM", "link": "https://www.ibm.com/think/topics/langgraph", "snippet": "LangGraph, created by LangChain, is an open source AI agent framework designed to build, deploy and manage complex generative AI agent workflows.", "position": 4}, {"title": "How to Build an Agent with Auth and Payments - LangGraph.js", "link": "https://www.youtube.com/watch?v=4Z2uBtIfmfE", "snippet": "Build a Complete AgentChat App with Auth + Payments using LangGraph.js This full-stack template includes everything you need to build and ...", "date": "5 days ago", "position": 5}, {"title": "LangGraph.js", "link": "https://langchain-ai.github.io/langgraphjs/", "snippet": "LangGraph — used by Replit, Uber, LinkedIn, GitLab and more — is a low-level orchestration framework for building controllable agents.", "position": 6}, {"title": "LangGraph Tutorial - How to Build Advanced AI Agent Systems", "link": "https://www.youtube.com/watch?v=1w5cCXlh7JQ", "snippet": "Check out PyCharm, the Python IDE for data and web professionals: https://jb.gg/try-pycharm-ide Free 3-Month Personal Subscription for ...", "date": "May 5, 2025", "position": 7}, {"title": "Building Ambient Agents with LangGraph - LangChain Academy", "link": "https://academy.langchain.com/courses/ambient-agents", "snippet": "LangGraph Platform is a platform for deploying AI agents that can scale with production volume. It offers easy-to-use APIs for managing agent state, memory, and ...", "position": 8}], "peopleAlsoAsk": [{"question": "Is LangGraph free or paid?", "snippet": "If you want to try out a basic version of our LangGraph server in your environment, you can also self-host on our Developer plan and get up to 100k nodes executed per month for free. Great for running hobbyist projects, with fewer features are available than in paid plans.", "title": "LangGraph Platform - LangChain", "link": "https://www.langchain.com/langgraph-platform"}, {"question": "Does ChatGPT use LangGraph?", "snippet": "By structuring interactions as a dynamic graph, LangGraph enables seamless collaboration between multiple LLMs, allowing ChatGPT to refine queries while Perplexity fetches real-world insights.", "title": "Combining ChatGPT and Perplexity with LangGraph | by Fateh Ali Aamir", "link": "https://fatehaliaamir.medium.com/combining-chatgpt-and-perplexity-with-langgraph-49233e74b8ab"}], "relatedSearches": [{"query": "LangGraph Studio"}, {"query": "Langgraph github"}, {"query": "LangGraph Python"}, {"query": "LangGraph JS"}, {"query": "LangGraph Academy"}, {"query": "LangGraph course"}, {"query": "LangGraph vs LangChain"}, {"query": "LangGraph documentation"}], "credits": 1}', name='search_web', tool_call_id='tool_call_id_1')]}
```  

`tool_calls` 에 여러 툴 호출을 정의해 잔달하면, 병렬로 도구를 호출할 수 있다.  

```python
# 다중 도구 호출
message_with_multiple_tool_calls = AIMessage(
    content="",
    tool_calls=[
        {
            "name" : "search_web",
            "args" : {"query" : "langgraph"},
            "id" : "tool_call_id_1",
            "type" : "tool_call"
        },
        {
            "name" : "python_code_interpreter",
            "args" : {"code" : "print(1 * 2 * 3 * 4)"},
            "id" : "tool_call_id_1",
            "type" : "tool_call"
        }
    ]
)

tool_node.invoke({"messages" : [message_with_multiple_tool_calls]})
# {'messages': [ToolMessage(content='{"searchParameters": {"q": "langgraph", "gl": "us", "hl": "en", "type": "search", "num": 10, "engine": "google"}, "organic": [{"title": "LangGraph - LangChain", "link": "https://www.langchain.com/langgraph", "snippet": "LangGraph sets the foundation for how we can build and scale AI workloads — from conversational agents, complex task automation, to custom LLM-backed ...", "position": 1}, {"title": "langchain-ai/langgraph: Build resilient language agents as graphs.", "link": "https://github.com/langchain-ai/langgraph", "snippet": "LangGraph is a low-level orchestration framework for building, managing, and deploying long-running, stateful agents.", "position": 2}, {"title": "Learn LangGraph basics - Overview", "link": "https://langchain-ai.github.io/langgraph/concepts/why-langgraph/", "snippet": "Overview¶. LangGraph is built for developers who want to build powerful, adaptable AI agents. Developers choose LangGraph for:.", "position": 3}, {"title": "What is LangGraph? - IBM", "link": "https://www.ibm.com/think/topics/langgraph", "snippet": "LangGraph, created by LangChain, is an open source AI agent framework designed to build, deploy and manage complex generative AI agent workflows.", "position": 4}, {"title": "How to Build an Agent with Auth and Payments - LangGraph.js", "link": "https://www.youtube.com/watch?v=4Z2uBtIfmfE", "snippet": "Build a Complete AgentChat App with Auth + Payments using LangGraph.js This full-stack template includes everything you need to build and ...", "date": "5 days ago", "position": 5}, {"title": "LangGraph.js", "link": "https://langchain-ai.github.io/langgraphjs/", "snippet": "LangGraph — used by Replit, Uber, LinkedIn, GitLab and more — is a low-level orchestration framework for building controllable agents ...", "position": 6}, {"title": "ReAct agent from scratch with Gemini 2.5 and LangGraph", "link": "https://ai.google.dev/gemini-api/docs/langgraph-example", "snippet": "LangGraph is a framework for building stateful LLM applications, making it a good choice for constructing ReAct (Reasoning and Acting) Agents.", "position": 7}, {"title": "LangGraph Tutorial - How to Build Advanced AI Agent Systems", "link": "https://www.youtube.com/watch?v=1w5cCXlh7JQ", "snippet": "For polished writing, BeLikeNative offers an excellent paraphrasing tool. It refines your sentences while keeping the original meaning ...", "date": "May 5, 2025", "position": 8}], "relatedSearches": [{"query": "langgraph studio"}, {"query": "langgraph github"}, {"query": "langgraph js"}, {"query": "langgraph python"}, {"query": "langgraph academy"}, {"query": "langgraph documentation"}, {"query": "langgraph course"}, {"query": "langgraph vs langchain"}], "credits": 1}', name='search_web', tool_call_id='tool_call_id_1'),
# ToolMessage(content='24\n', name='python_code_interpreter', tool_call_id='tool_call_id_1')]}
```  

### ToolNode with LLM
`ToolNode` 는 `LLM` 과 함께 사용할수 있는데 아래는 `LLM` 에 사용자 쿼리를 전달하고, 
이를 바탕으로 적절한 툴을 호출해 최종 결과를 반환하는 예시이다.  

```python
from langchain_google_genai import ChatGoogleGenerativeAI
import os

os.environ["GOOGLE_API_KEY"] = "api key"
# llm 모델에 정의된 툴 바인딩
llm_with_tools = ChatGoogleGenerativeAI(model="gemini-2.0-flash").bind_tools(tools)
```  

사용자 질의를 전달하면 `LLM` 이 도구 호출에 필요한 파라미터를 생성한다.  

```python
llm_with_tools.invoke("가장 작은 소수 10개를 출력하는 python code 작성해줘").tool_calls
# [{'name': 'python_code_interpreter',
# 'args': {'code': '\ndef is_prime(n):\n    if n <= 1:\n        return False\n    for i in range(2, int(n**0.5) + 1):\n        if n % i == 0:\n            return False\n    return True\n\nprimes = []\nnum = 2\nwhile len(primes) < 10:\n    if is_prime(num):\n        primes.append(num)\n    num += 1\n\nprint(primes)\n'},
# 'id': 'a1e99add-2b5d-4b6f-8c45-c9906a0e5ab8',
# 'type': 'tool_call'}]
```  

이를 `ToolNode` 와 함께 연결해 사용하면 아래와 같다.  

```python
# 도구 노드를 통한 메시지 처리 및 llm 모델의 도구 기반 응답 생성
tool_node.invoke(
    {
        "messages" : [
            llm_with_tools.invoke("가장 작은 소수 10개를 출력하는 python code 작성해줘")
        ]
    }
)
# {'messages': [ToolMessage(content='[2, 3, 5, 7, 11, 13, 17, 19, 23, 29]\n', name='python_code_interpreter', tool_call_id='b9d1c94e-6caa-4191-ac4e-83cd98fdc7f1')]}
```  
