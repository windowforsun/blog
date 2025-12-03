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

### ToolNode with LangGraph(Agent)
`LangGraph` 에서 `ToolNode` 는 그래프구 구조에 툴 노드를 적절하게 구성해 사용할 수 있다. 
`Agent` 는 사용자 쿼리를 입력을 받아, 쿼리 해결에 필요한 충분한 정보를 얻을 떄까지 반복적으로 도구를 호출하도록 할 수 있다.  

```python
from IPython.display import Image, display
from langgraph.graph import StateGraph, MessagesState, START, END

# llm 모델을 사용해 메시지 처리 및 응답 생성, 도구 호출이 포함된 응답 반환
def call_model(state: MessagesState):
  messages = state["messages"]
  response = llm_with_tools.invoke(messages)

  return {"messages" : [response]}


# 그래프 초기화
graph_builder = StateGraph(MessagesState)

# 에이전트 와 도구 노드 정의
graph_builder.add_node("agent", call_model)
graph_builder.add_node("tools", tool_node)

# 에이전트 시작점에 에이전트 노드 연결
graph_builder.add_edge(START, "agent")

# 에이전트 노드의 조건부 분기 설정, 도구 또는 종료 지점 연결
graph_builder.add_conditional_edges("agent", tools_condition)

# 도구 노드에서 에이전트 노드로 순환 연결
graph_builder.add_edge("tools", "agent")

# 에이전트 노드에서 종료 지점으로 연결
graph_builder.add_edge("agent", END)

# 그래프를 컴파일해 에이전트 앱 생성
agent = graph_builder.compile()

# 그래프 시각화
try:
    display(Image(agent.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/toolnode-removemessage-1.png)


먼저 `LangGraph` 로 구현된 `Agent` 에 파이썬 코드에 대한 질의를 수행하면 아래와 같다.  

```python
# 파이썬 코드 질문
for chunk in agent.stream(
    {"messages" : ["human", "가장 작은 소수 10개를 출력하는 python code 작성해줘"]},
    stream_mode="values"
):
  chunk["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# 가장 작은 소수 10개를 출력하는 python code 작성해줘
# ================================== Ai Message ==================================
# Tool Calls:
# python_code_interpreter (dddaf6e1-d8fc-4a73-9895-2c6ed403e914)
# Call ID: dddaf6e1-d8fc-4a73-9895-2c6ed403e914
# Args:
# code:
# def is_prime(n):
#     if n <= 1:
#         return False
#     for i in range(2, int(n**0.5) + 1):
#         if n % i == 0:
#             return False
#     return True
# 
# primes = []
# num = 2
# while len(primes) < 10:
#     if is_prime(num):
#         primes.append(num)
#     num += 1
# 
# print(primes)
# ================================= Tool Message =================================
# Name: python_code_interpreter
# 
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
# 
# ================================== Ai Message ==================================
# 
# 가장 작은 소수 10개는 [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]입니다.
```  

웹 검색관련 질의를 하면 다음과 같다.  

```python
# 검색 질문

for chunk in agent.stream(
    {"messages" : ["human", "langgraph 의 최신 정보를 바탕으로 기본개념에 대해 설명해줘"]},
    stream_mode="values"
):
  chunk["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# langgraph 의 최신 정보를 바탕으로 기본개념에 대해 설명해줘
# ================================== Ai Message ==================================
# Tool Calls:
# search_web (1c2699b4-5f11-42fd-896f-455f78c2ef99)
# Call ID: 1c2699b4-5f11-42fd-896f-455f78c2ef99
# Args:
# query: langgraph basic concepts
# ================================= Tool Message =================================
# Name: search_web
# 
# {"searchParameters": {"q": "langgraph basic concepts", "gl": "us", "hl": "en", "type": "search", "num": 10, "engine": "google"}, "organic": [{"title": "Concepts - LangGraph - GitHub Pages", "link": "https://langchain-ai.github.io/langgraph/concepts/", "snippet": "This guide provides explanations of the key concepts behind the LangGraph framework and AI applications more broadly.", "sitelinks": [{"title": "Overview", "link": "https://langchain-ai.github.io/langgraph/concepts/why-langgraph/"}, {"title": "Graph API", "link": "https://langchain-ai.github.io/langgraph/concepts/low_level/"}, {"title": "Why LangGraph?", "link": "https://langchain-ai.github.io/langgraph/concepts/high_level/"}, {"title": "Persistence", "link": "https://langchain-ai.github.io/langgraph/concepts/persistence/"}], "position": 1}, {"title": "Conceptual Guide - LangGraph", "link": "https://langchain-ai.github.io/langgraphjs/concepts/", "snippet": "This guide provides explanations of the key concepts behind the LangGraph framework and AI applications more broadly.", "position": 2}, {"title": "LangGraph - LangChain", "link": "https://www.langchain.com/langgraph", "snippet": "LangGraph is a stateful, orchestration framework that brings added control to agent workflows. LangGraph Platform is a service for deploying and scaling ...", "sitelinks": [{"title": "LangGraph Academy Course", "link": "https://academy.langchain.com/courses/intro-to-langgraph"}, {"title": "Built with LangGraph", "link": "https://www.langchain.com/built-with-langgraph"}, {"title": "LangGraph Platform", "link": "https://www.langchain.com/langgraph-platform"}, {"title": "Experts", "link": "https://www.langchain.com/experts"}], "position": 3}, {"title": "LangGraph Tutorial: What Is LangGraph and How to Use It?", "link": "https://www.datacamp.com/tutorial/langgraph-tutorial", "snippet": "The core concepts of LangGraph include: graph structure, state management, and coordination.", "date": "Jun 26, 2024", "position": 4}, {"title": "Introduction to LangGraph - LangChain Academy", "link": "https://academy.langchain.com/courses/intro-to-langgraph", "snippet": "LangGraph Platform is a platform for deploying AI agents that can scale with production volume. It offers easy-to-use APIs for managing agent state, memory, and ...", "position": 5}, {"title": "LangGraph for Beginners, Part 1: Create a simple Graph. | AI Agents", "link": "https://medium.com/ai-agents/langgraph-for-beginners-basics-f8efe7c8acce", "snippet": "In this series, we will cover the basic concepts of LangGraph that is easy to understand even for beginners and in this article we will ...", "date": "Oct 19, 2024", "position": 6}, {"title": "From Basics to Advanced: Exploring LangGraph", "link": "https://towardsdatascience.com/from-basics-to-advanced-exploring-langgraph-e8c1cf4db787/", "snippet": "Another important concept is the state of the graph. The state serves as a foundational element for collaboration among the graph's components. ...", "date": "Aug 15, 2024", "position": 7}, {"title": "LangGraph Tutorial: A Comprehensive Guide for Beginners", "link": "https://blog.futuresmart.ai/langgraph-tutorial-for-beginners", "snippet": "At the heart of LangGraph's design lies a graph-based representation of the application's workflow. This graph comprises two primary elements:.", "date": "Oct 1, 2024", "sitelinks": [{"title": "Key Concepts", "link": "https://blog.futuresmart.ai/langgraph-tutorial-for-beginners#heading-key-concepts"}, {"title": "Getting Started with LangGraph", "link": "https://blog.futuresmart.ai/langgraph-tutorial-for-beginners#heading-getting-started-with-langgraph"}, {"title": "Advanced LangGraph...", "link": "https://blog.futuresmart.ai/langgraph-tutorial-for-beginners#heading-advanced-langgraph-techniques"}], "position": 8}, {"title": "Introduction to LangGraph: A Beginner's Guide - Medium", "link": "https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141", "snippet": "In this article, we'll introduce LangGraph, walk you through its basic concepts, and share some insights and common points of confusion for ...", "date": "Feb 14, 2024", "sitelinks": [{"title": "Key Concepts", "link": "https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141#:~:text=Key%20Concepts"}, {"title": "Common Confusions", "link": "https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141#:~:text=Common%20Confusions"}, {"title": "Conclusion", "link": "https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141#:~:text=Conclusion"}], "position": 9}, {"title": "Introduction to LangGraph: A Quick Dive into Core Concepts", "link": "https://www.youtube.com/watch?v=J5d1l6xgQBc", "snippet": "In this video, we'll explore LangGraph, a powerful tool that enables agentic workflows with language learning models (LLMs) through cycles, ...", "date": "May 27, 2024", "position": 10}], "peopleAlsoAsk": [{"question": "What is the basics of LangGraph?", "snippet": "LangGraph Platform is a platform for deploying AI agents that can scale with production volume. It offers easy-to-use APIs for managing agent state, memory, and user interactions— which makes building dynamic experiences more accessible.", "title": "Introduction to LangGraph - LangChain Academy", "link": "https://academy.langchain.com/courses/intro-to-langgraph"}, {"question": "What are the basic concepts of graph?", "snippet": "A graph is determined as a mathematical structure that represents a particular function by connecting a set of points. It is used to create a pairwise relationship between objects. The graph is made up of vertices (nodes) that are connected by the edges (lines).", "title": "Graph Theory - BYJU'S", "link": "https://byjus.com/maths/graph-theory/"}, {"question": "What is the function of LangGraph?", "snippet": "The Functional API in LangGraph provides a flexible approach to building AI workflows, with powerful features like human-in-the-loop interactions, state management, persistence, and streaming. These capabilities enable developers to create sophisticated applications that effectively combine automation with human input.", "title": "Introducing the LangGraph Functional API - LangChain Blog", "link": "https://blog.langchain.com/introducing-the-langgraph-functional-api/"}, {"question": "Is LangGraph easy to use?", "snippet": "LangGraph is more complex. It takes more time to learn, and the code can be harder to write and maintain. But this complexity is a trade-off for the additional power and flexibility it provides. If you're building a complex workflow, the extra effort might be worth it.", "title": "LangChain vs. LangGraph: Choosing the Right Framework | by Tahir", "link": "https://medium.com/@tahirbalarabe2/langchain-vs-langgraph-choosing-the-right-framework-0e393513da3d"}], "relatedSearches": [{"query": "Langgraph basic concepts pdf"}, {"query": "Langgraph basic concepts cheat sheet"}, {"query": "Langgraph basic concepts examples"}, {"query": "Langgraph basic concepts github"}, {"query": "LangGraph Studio"}, {"query": "LangGraph documentation"}, {"query": "Langgraph github"}, {"query": "LangGraph examples"}], "credits": 1}
# ================================== Ai Message ==================================
# 
# LangGraph는 AI 에이전트 워크플로우를 오케스트레이션하기 위한 프레임워크입니다. 주요 개념은 다음과 같습니다:
# 
# *   **그래프 구조:** LangGraph는 애플리케이션의 워크플로우를 그래프로 표현합니다. 이 그래프는 노드와 엣지로 구성됩니다. 노드는 워크플로우의 단계를 나타내고, 엣지는 단계 간의 전환을 나타냅니다.
# *   **상태 관리:** LangGraph는 그래프의 상태를 관리합니다. 상태는 그래프의 컴포넌트 간의 협업을 위한 기본 요소 역할을 합니다.
# *   **노드:** 그래프의 노드는 상태를 변경하는 행위자입니다. 노드는 LLM 체인, 함수 호출 또는 라우터일 수 있습니다.
# *   **엣지:** 그래프의 엣지는 노드 간의 연결을 정의합니다. 엣지는 조건부일 수 있으며, 이는 상태에 따라 다른 노드로 라우팅될 수 있음을 의미합니다.
# *   **에이전트 워크플로우:** LangGraph는 에이전트 워크플로우를 오케스트레이션하는 데 사용됩니다. 에이전트 워크플로우는 여러 단계를 거쳐 작업을 완료하는 에이전트의 시퀀스입니다.
# *   **상태 저장:** LangGraph는 상태 저장 프레임워크입니다. 즉, 그래프의 상태가 유지됩니다. 이를 통해 에이전트가 이전 단계의 결과를 기반으로 결정을 내릴 수 있습니다.
# *   **병렬 처리:** LangGraph는 병렬 처리를 지원합니다. 이를 통해 여러 단계를 동시에 실행하여 워크플로우의 성능을 향상시킬 수 있습니다.
# 
# LangGraph는 복잡한 에이전트 워크플로우를 구축하고 관리하는 데 유용한 도구입니다. LangChain과 비교했을 때 더 복잡하지만, 더 강력하고 유연합니다.
```  

도구 호출이 불필요한 질문은 다음과 같다.  

```python
# 도구가 필요없는 질문

for chunk in agent.stream(
    {"messages" : ["human", "langgraph 의 기본개념에 대해 설명해줘"]},
    stream_mode="values"
):
  chunk["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# langgraph 의 기본개념에 대해 설명해줘
# ================================== Ai Message ==================================
# 
# LangGraph는 LangChain을 기반으로 하는 라이브러리로, LLM(Large Language Model)을 사용하여 복잡한 multi-agent 워크플로우를 구축하는 데 특화되어 있습니다. 기존의 순차적인 체인 방식과는 달리, 그래프 구조를 사용하여 에이전트 간의 상호작용과 흐름을 정의하고 관리합니다.
# 
# **LangGraph의 핵심 개념:**
# 
# 1.  **Nodes (노드):** 그래프의 기본적인 구성 요소이며, 에이전트, 도구(tools), LLM 호출 또는 사용자 정의 함수와 같은 실행 가능한 단위를 나타냅니다. 각 노드는 특정 작업을 수행하고 결과를 다음 노드로 전달합니다.
# 
# 2.  **Edges (엣지):** 노드 간의 연결을 나타내며, 워크플로우의 흐름을 정의합니다. 엣지는 조건부 분기, 루프, 병렬 실행 등 다양한 흐름 제어를 지원합니다.
# 
# 3.  **State (상태):** 그래프 전체에서 공유되는 데이터 컨테이너입니다. 노드는 상태를 읽고 수정하여 정보를 공유하고 워크플로우의 진행 상황을 추적할 수 있습니다. 상태는 에이전트 간의 협업과 의사 결정을 가능하게 합니다.
# 
# 4.  **Graph (그래프):** 노드와 엣지의 집합으로, 전체 워크플로우를 나타냅니다. 그래프는 시작 노드에서 시작하여 엣지를 따라 다른 노드로 이동하면서 작업을 수행하고, 최종적으로 종료 노드에서 완료됩니다.
# 
# **LangGraph의 장점:**
# 
# *   **복잡한 워크플로우 관리:** 그래프 구조를 통해 복잡한 multi-agent 상호작용을 시각적으로 표현하고 관리할 수 있습니다.
# *   **유연성:** 다양한 유형의 노드와 엣지를 사용하여 워크플로우를 사용자 정의할 수 있습니다.
# *   **확장성:** 필요에 따라 노드와 엣지를 추가하거나 수정하여 워크플로우를 확장할 수 있습니다.
# *   **재사용성:** 그래프를 모듈화하여 재사용 가능한 워크플로우를 구축할 수 있습니다.
# *   **관찰 가능성:** 워크플로우의 실행 과정을 추적하고 디버깅할 수 있습니다.
# 
# **LangGraph의 활용 예시:**
# 
# *   **자동화된 고객 지원:** 여러 에이전트가 협력하여 고객 문의를 처리하고 해결하는 워크플로우
# *   **연구 및 개발:** 여러 연구원이 협력하여 데이터를 분석하고 가설을 검증하는 워크플로우
# *   **콘텐츠 생성:** 여러 에이전트가 협력하여 글쓰기, 편집, 디자인 등의 작업을 수행하는 워크플로우
# *   **코드 생성:** 여러 에이전트가 협력하여 코드를 작성, 테스트, 디버깅하는 워크플로우
# 
# LangGraph는 복잡한 multi-agent 시스템을 구축하고 관리하는 데 매우 유용한 도구입니다. 기존의 체인 방식의 한계를 극복하고 더욱 강력하고 유연한 워크플로우를 구축할 수 있도록 지원합니다.
```  

여러 도구가 필요한 질의는 아래와 같다.  

```python
# 여러 도구를 사용하는 질문

for chunk in agent.stream(
    {"messages" : ["human", "Microsoft 의 최근 7일 주가의 이동평균선을 python 코드로 구해 알려줘"]},
    stream_mode="values"
):
  chunk["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# Microsoft 의 최근 7일 주가의 이동평균선을 python 코드로 구해 알려줘
# ================================== Ai Message ==================================
# Tool Calls:
# search_web (ec21ed95-6640-489f-b5d3-831f17ee4ca8)
# Call ID: ec21ed95-6640-489f-b5d3-831f17ee4ca8
# Args:
# query: Microsoft stock price past 7 days moving average
# ================================= Tool Message =================================
# Name: search_web
# 
# {"searchParameters": {"q": "Microsoft stock price past 7 days moving average", "gl": "us", "hl": "en", "type": "search", "num": 10, "engine": "google"}, "organic": [{"title": "MSFT Technical Analysis for Microsoft Corp Stock - Barchart.com", "link": "https://www.barchart.com/stocks/quotes/MSFT/technical-analysis", "snippet": "Period, Moving Average, Price Change, Percent Change, Average Volume. 5-Day, 495.07, +1.39, +0.28%, 22,631,561. 20-Day, 483.21, +34.97, +7.54%, 20,248,770.", "position": 1}, {"title": "Microsoft Corporation (MSFT) Stock Historical Prices & Data - Yahoo ...", "link": "https://finance.yahoo.com/quote/MSFT/history/", "snippet": "Discover historical prices for MSFT stock on Yahoo Finance. View daily, weekly or monthly format back to when Microsoft Corporation stock was issued.", "attributes": {"Missing": "moving | Show results with:moving"}, "position": 2}, {"title": "Microsoft Corp. Advanced Charts - MSFT - MarketWatch", "link": "https://www.marketwatch.com/investing/stock/msft/charts", "snippet": "Microsoft Corp. advanced stock charts by MarketWatch. View MSFT historial stock data and compare to other stocks and exchanges.", "attributes": {"Missing": "7 | Show results with:7"}, "position": 3}, {"title": "MSFT Stock Price Chart Technical Analysis - Financhill", "link": "https://financhill.com/stock-price-chart/msft-technical-analysis", "snippet": "Microsoft share price is 498.84 while MSFT 8-day exponential moving average is 492.70, which is a Buy signal. · The stock price of MSFT is 498.84 while Microsoft ...", "position": 4}, {"title": "Microsoft Stock Price History - Investing.com", "link": "https://www.investing.com/equities/microsoft-corp-historical-data", "snippet": "Access Microsoft stock price history with daily data, historical prices, all-time highs, and stock chart history. Download and analyze trends easily.", "position": 5}, {"title": "MSFT Stock Chart and Price — Microsoft (NASDAQ) - TradingView", "link": "https://www.tradingview.com/symbols/NASDAQ-MSFT/", "snippet": "MSFT stock has risen by 1.19% compared to the previous week, the month change is a 8.10% rise, over the last year Microsoft Corp. has showed a 8.87% increase.", "attributes": {"Missing": "average | Show results with:average"}, "sitelinks": [{"title": "MSFT technical analysis", "link": "https://www.tradingview.com/symbols/NASDAQ-MSFT/technicals/"}, {"title": "Price to earnings Ratio (TTM)", "link": "https://www.tradingview.com/symbols/NASDAQ-MSFT/financials-statistics-and-ratios/price-earnings/"}, {"title": "Forecast", "link": "https://www.tradingview.com/symbols/NASDAQ-MSFT/forecast/"}, {"title": "Financials", "link": "https://www.tradingview.com/symbols/NASDAQ-MSFT/financials-overview/"}], "position": 6}, {"title": "Microsoft (MSFT) Technical Analysis - TipRanks.com", "link": "https://www.tipranks.com/stocks/msft/technical-analysis", "snippet": "Microsoft's (MSFT) 20-Day exponential moving average is 481.36, while Microsoft's (MSFT) share price is $491.09, making it a Buy.", "date": "2 days ago", "attributes": {"Missing": "7 | Show results with:7"}, "position": 7}, {"title": "MSFT Price History for Microsoft Corp Stock - Barchart.com", "link": "https://www.barchart.com/stocks/quotes/MSFT/price-history/historical", "snippet": "The historical data and Price History for Microsoft Corp (MSFT) with Intraday, Daily, Weekly, Monthly, and Quarterly data available for download.", "position": 8}, {"title": "150 Day Moving Avg For Microsoft Corporation (MSFT) - Finbox", "link": "https://finbox.com/NASDAQGS:MSFT/explorer/asset_price_avg_150d/", "snippet": "Microsoft's 150 day moving avg is 420.18.. View Microsoft Corporation's 150 Day Moving Avg trends, charts, and more.", "position": 9}, {"title": "MSFT Microsoft Corporation Stock Price & Overview - Seeking Alpha", "link": "https://seekingalpha.com/symbol/MSFT/momentum/performance", "snippet": "Microsoft Corporation (MSFT) stock price is 494.84 and Microsoft Corporation (MSFT) 200-day simple moving average is 424.43. MSFT Relative Strength.", "attributes": {"Missing": "7 | Show results with:7"}, "position": 10}], "relatedSearches": [{"query": "Microsoft stock price history 1980"}, {"query": "Microsoft stock price 1994"}, {"query": "Microsoft stock price 1990"}, {"query": "MSFT 200-day Moving Average"}, {"query": "What was the closing price of msft on march 31 2025"}, {"query": "MSFT 50-day Moving Average"}, {"query": "MSFT 100 day moving average"}, {"query": "Microsoft stock price in 2000"}], "credits": 1}
# ================================== Ai Message ==================================
# Tool Calls:
# python_code_interpreter (06b9981d-a400-45ec-a6ad-3337d73380b4)
# Call ID: 06b9981d-a400-45ec-a6ad-3337d73380b4
# Args:
# code:
# import yfinance as yf
# 
# msft = yf.Ticker("MSFT")
# history = msft.history(period="7d")
# 
# prices = history["Close"].tolist()
# 
# if len(prices) > 0:
#     moving_average = sum(prices) / len(prices)
#     print(f"Microsoft의 최근 7일 주가 이동평균선: {moving_average:.2f}")
# else:
#     print("지난 7일 동안의 주가 데이터를 가져올 수 없습니다.")
# ================================= Tool Message =================================
# Name: python_code_interpreter
# 
# Microsoft의 최근 7일 주가 이동평균선: 495.01
# 
# ================================== Ai Message ==================================
# 
# Microsoft의 최근 7일 주가 이동평균선은 495.01입니다.
```  

## RemoveMessage
그래프 기반 워크플로우에서 상태는 종종 여러 메시지의 집합으로 구성된다. 
보통은 새로운 메시지를 상태에 추가만 하지만, 필요에 따라 특정 메시지를 제거해야 하는 상황도 발생한다. 
이럴 때 사용할 수 있는 도구가 `RemoveMessage` 수정자 이다. 
`RemoveMessage` 는 `LangGraph` 에서 메시지 상태를 동적으로 관리할 수 있게 해주는 공식적인 방법이며, 
이를 적용하려면 해당 상태가 [reducer](https://langchain-ai.github.io/langgraph/concepts/low_level/?h=messagesstate#reducers) 
를 지원해야 한다.  

`LangGraph` 에서 기본적으로 제공되는 `MessagesState` 는 `messages` 라는 키가 있다. 
이 키는 메시지 리스트를 저장하며, 이 리스트의 `reducer` 는 `RemoveMessage` 와 같은 수정자를 지원한다. 
그러므로 `RemoveMessage` 는 `reucer` 메커니즘을 통해 `messages` 리스트에서 특정 메시지를 찾아 삭제하는 역할을 하며, 
이를 통해 불필요한 메시지나 더 이상 필요하지 않은 데이터를 상태에서 효과적으로 제거할 수 있다.  

진행을 위해 앞선 `ToolNode` 의 전체 코드를 다시 작성해 `RemoveMessage` 예제로 사용한다.   

```python
from typing import Literal
from langchain_core.tools import tool
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState, StateGraph, START, END
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_core.messages import AIMessage
from langchain_experimental.tools.python.tool import PythonAstREPLTool
from typing import List, Dict
from langchain_community.utilities import GoogleSerperAPIWrapper
from langchain_google_genai import ChatGoogleGenerativeAI
import os

os.environ["SERPER_API_KEY"] = "api key"
os.environ["GOOGLE_API_KEY"] = "api key"


# 도구 정의
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

memory = MemorySaver()
llm_with_tools = ChatGoogleGenerativeAI(model="gemini-2.0-flash").bind_tools(tools)

# llm 모델을 사용해 메시지 처리 및 응답 생성, 도구 호출이 포함된 응답 반환
def call_model(state: MessagesState):
  messages = state["messages"]
  response = llm_with_tools.invoke(messages)

  return {"messages" : [response]}


# 그래프 초기화
graph_builder = StateGraph(MessagesState)

# 에이전트 와 도구 노드 정의
graph_builder.add_node("agent", call_model)
graph_builder.add_node("tools", tool_node)

# 에이전트 시작점에 에이전트 노드 연결
graph_builder.add_edge(START, "agent")

# 에이전트 노드의 조건부 분기 설정, 도구 또는 종료 지점 연결
graph_builder.add_conditional_edges("agent", tools_condition)

# 도구 노드에서 에이전트 노드로 순환 연결
graph_builder.add_edge("tools", "agent")

# 에이전트 노드에서 종료 지점으로 연결
graph_builder.add_edge("agent", END)

# 그래프를 컴파일해 에이전트 앱 생성
agent = graph_builder.compile(checkpointer=memory)
```  

구성된 그래프를 시각화하면 아래와 같다.  

```python
from IPython.display import Image, display


# 그래프 시각화
try:
    display(Image(agent.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/toolnode-removemessage-2.png)


간단한 대화를 하고 메시지 목록을 확인한다.  

```python
from langchain_core.messages import HumanMessage

config = {"configurable" : {"thread_id": "1"}}
input_message = HumanMessage(content="안녕 내 이름은 철수야")

for chunk in agent.stream(
    {"messages" : [input_message]},
    config,
    stream_mode="values"
):
  chunk["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# 안녕 내 이름은 철수야
# ================================== Ai Message ==================================
# 
# 안녕하세요 철수 씨! 무엇을 도와드릴까요?


# 후속 질문
input_message = HumanMessage(content="내 이름이 뭐라고?")

for chunk in agent.stream(
        {"messages" : [input_message]},
        config,
        stream_mode="values"
):
    chunk["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# 내 이름이 뭐라고?
# ================================== Ai Message ==================================
# 
# 당신의 이름은 철수입니다.


# 모든 메시지 확인
messages = agent.get_state(config).values["messages"]

for message in messages:
    message.pretty_print()
# ================================ Human Message =================================
# 
# 안녕 내 이름은 철수야
# ================================== Ai Message ==================================
# 
# 안녕하세요 철수 씨! 무엇을 도와드릴까요?
# ================================ Human Message =================================
# 
# 내 이름이 뭐라고?
# ================================== Ai Message ==================================
# 
# 당신의 이름은 철수입니다.
```  
