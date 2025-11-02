--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Agent Memory and Stream"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LangGraph 에서 주어진 목표를 수행하는 Agent 구현과정에서 Agent Memory 와 Stream 기능에 대해 알아본다.'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - Agent
    - Memory
    - Stream
    - Tool
    - Persistent Checkpoint
    - DuckDuckGo
    - Google Gemini
    - LLM
toc: true
use_math: true
---  

## LangGraph Agent Memory and Stream
`LangGraph` 에서 `Agent` 는 특정 작업을 수행하는 독립적인 실행 단위로, 
각 `Agent` 는 자신의 상태(`State`)를 유지하고, 다른 `Agent` 와 상호작용하며, 
작업을 수행하는 데 필요한 도구(`Tool`)를 사용할 수 있다. 

`Agent` 는 `LLM` 에이전트, 데이터베이스 쿼리 에이전트, 파일 처리 에이전트 등 다양한 형태로 구현될 수 있으며, 
각 `Agent` 는 자신의 작업을 정의하고, 다른 `Agent` 와 협력하거나 경쟁하여
목표를 달성한다.

`Agent` 의 가장 큰 특징은 스스로 생각하고, 결정하며, 행동하는 `AI` 비서에 가까우 개념이다. 
단순히 텍스트를 생성하는 것이 아니라, 주어진 목표에 맞춰 논리적으로 추론하고 결정을 내릴 수 있다. 
그리고 다양한 상황과 맥락에 맞춰 행동을 조정하며 사용자 요구나 환경 변화에 유연하게 대응한다. 
그러므로 사용자의 개입이 최소화 될 수 있고, 목표 달성을 위해 필요한 작업들을 스스로 계획하고 설행한다.  

예제 진행을 위해 구현할 `Agent` 는 검색 `API` 를 활요앟여 검색 기능을 구현한 도구를 사용한다. 
검색 `API` 는 [DuckDuckGo](https://python.langchain.com/docs/integrations/tools/ddg/)
를 사용해 `Agent` 에서 사용할 [Tool](https://python.langchain.com/docs/concepts/tools/)
을 정의한다.  

그리고 정의된 `Tool` 을 바탕으로 `LangGraph` 의 `Persistent Checkpoint` 기능을 사용하여 
대화의 `Context` 상태를 추적할 수 있도록해 `Multi-Turn` 대화가 가능하도록 개선한다.  

마지막으로는 `LangGraph` 의 출력함수인 `stream()` 함수에 대해 좀 더 자세하게 어떠한 추가 기능을 수행할 수 있는지 알아본다.  


### LangGraph Agent
먼저 `Agent` 에서 사용할 상태와 검색 도구를 정의해 `LLM` 과 바인딩한다. 
그리고 노드를 정의한 뒤 상태 그래프를 초기화하고 노드를 추가한다.  

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages
from langchain_community.tools import DuckDuckGoSearchRun
from langgraph.graph import StateGraph

import os

os.environ["GOOGLE_API_KEY"] = "api key"
model = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

# 상태 정의
class ChatbotState(TypedDict):
    messages: Annotated[list, add_messages]

search_tool = DuckDuckGoSearchRun()
tools = [search_tool]

# llm 과 도구 바인딩
model_with_tools = model.bind_tools(tools)

# 노드 정의
def chatbot(state: ChatbotState):
    answer = model_with_tools.invoke(state["messages"])

    return {"messages" : [answer]}

# 그래프 생성 및 노드
graph_builder = StateGraph(ChatbotState)
graph_builder.add_node("chatbot", chatbot)

```  

다음으로는 `LLM` 이 외부 도구를 활용해 기능을 수행할 수 있도록해야 한다. 
`LangGraph` 에는 [ToolNode](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.prebuilt.ToolNode.html)
라는 이미 구현된 클래스가 존재하지만 비슷한 역할을 수행하는 `BasicToolNode` 클래스를 구현해 본다. 
이는 가장 최근 메시지를 바탕으로 메시지에 `tool_calls` 가 포함되어 있는 경우 도구를 호출하는 역할을 한다.  

```python

import json
from langchain_core.messages import ToolMessage

# 직접 구현했지만 LangGraph 의 ToolNode 를 사용할 수 있음
# https://langchain-ai.github.io/langgraph/reference/agents/#langgraph.prebuilt.chat_agent_executor.create_react_agent

class BasicToolNode:
  def __init__(self, tools: list) -> None:
    # 도구 리스트
    self.tools_list = {tool.name: tool for tool in tools}

  def __call__(self, inputs: dict):

    # 메시지가 있는 경우 가장 최근 메시지
    if messages := inputs.get("messages", []):
      message = messages[-1]
    else:
      raise ValueError("No messages found in input")

    outputs = []
    for tool_call in message.tool_calls:
      # 도구 호출 결과
      tool_result = self.tools_list[tool_call["name"]].invoke(tool_call["args"])


      outputs.append(
          # 도구 결과를 메시지에 저장
          ToolMessage(
              content=json.dumps(tool_result, ensure_ascii=False),
              name=tool_call["name"],
              tool_call_id=tool_call["id"]
          )
      )


    return {"messages" : outputs}


# 도구 노드 생성
tool_node = BasicToolNode(tools=tools)

# 도구 노드 추가
graph_builder.add_node("tools", tool_node)
```  

`LLM` 이 도구를 사용할 수 있도록 구현이 되었다면, 
이제 어떤 `ToolNode` 를 사용할지 결정하는 조건 또는 분기 로직을 정의해야 한다. 
여러 개의 `ToolNode` 가 있는 상황에서, 현재 입력이나 상태에 따라 특정 `ToolNode` 만 
실행되도록 하는 조건식 역할을 한다. 
주로 그래프 내에서 분기(`Branch`) 지점에서 사용되고, `LLM` 의 추론 결과나 입력 데이터에 따라 워크플로우의 흐름을 동적 제어하는 경우 사용한다. 
이러한 조건 분기의 내용은 그래프에서 `add_conditional_edges()` 함수를 사용해 그래프에 추가할 수 있다.  

이 또한 `LangGraph` 에서는 [tools_condition](https://langchain-ai.github.io/langgraph/reference/agents/#langgraph.prebuilt.tool_node.InjectedStore)
이라는 함수로 이미 구현돼 있지만 예제에서는 비슷한 기능을 직접 구현해 본다.  

```python
# 직접 구현하지만 langgraph 의 tools_condition 을 사용할 수 있음
# https://langchain-ai.github.io/langgraph/reference/agents/#tools_condition

from langgraph.graph import START, END

def route_tools(state: ChatbotState):
  if messages := state.get("messages", []):
    ai_message = messages[-1]
  else:
    raise ValueError(f"No messages found in input state to tool_edge: {state}")

  if hasattr(ai_message, "tool_calls") and len(ai_message.tool_calls) > 0:
    return "tools"

  return END


# tools_condition 함수는 챗봇이 도구 사용으 요청하면 tools 를 반환하고, 직접 응답이 가능한 경우 END 를 반환
graph_builder.add_conditional_edges(
    source="chatbot",
    path=route_tools,
    path_map={"tools": "tools", END:END}
)

# tools > chatbot
graph_builder.add_edge("tools", "chatbot")

# START > chatbot
graph_builder.add_edge(START, "chatbot")

# 그래프 컴파일
graph = graph_builder.compile()
```  

`add_conditional_edges()` 함수는 다은과 같은 메개변수를 갖는다.  

- `source` : 조건부 에싲가 출발하는 `시작 노드` 의 이름이다. 해당 노드에서 워크플로우가 분기되며, 조건에 따라 다음 노드가 다르게 결정된다. 
- `path` : 어떤 조건에서 어느 노드로 이동할지를 결정하는 함수(또는 `Runnable`)이다. 입력값에 따라 이동할 다음 노드의 이름(또는 키값)을 반환하는 함수이다. `path` 가 결정한 값에 따라, 그래프가 다음에 실행할 노드를 결정하는 역할을 한다. (`END` 를 뱐환시 그래프 종료)
- `path_map` : `path` 함수가 반환한 값을 실제 그래프의 노드 이름과 매핑하는 역할을 한다. 이는 `path` 반환값이 노드 이름이 아닌 경우 `path_map` 에 매핑될 노드의 이름을 정의해 사용하면 된다. 
- `then` : 조건에 따라 선택된 노드에 이어 추가로 실행할 노드의 이름을 지정한다. 분기로 간 뒤 반드시 이어 실행할 후속 노드가 있는 경우 사용할 수 있다. 

이제 `LangGraph` 를 바탕으로 도구를 사용할 수 있는 `Agent` 가 구현되었다. 
이를 그래프로 시각화하면 아래와 같다.  

```python
from IPython.display import Image, display


# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  


.. 그림 ..
agent-memory-stream-1.png


`Agent` 가 잘 동작하는지 테스트해 보면 아래와 같다.  

```python
user_input = "현재 가장 youtube 구독자가 많은 채널을 알려줘"

for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
    for value in event.values():
        print("Assistant:", value["messages"][-1].content)

# Assistant:
# Assistant: "Even though the Viacom/YouTube lawsuits have been settled for a decade now, YouTube is still a battleground where major entertainment companies compete with silly imitations for millions of eyeballs. Channel Distribution. Here's a breakdown of the Top 100 Most Subscribed channels this month in terms of their countries of origin: India: 35 Discover the most subscribed YouTube channels in the world as of August 14, 2024, including T-Series, MrBeast, Cocomelon, and more. Learn about their journeys, content, and popularity in this article by Forbes India. Discover the global creators and brands dominating YouTube with massive follower counts. From music and entertainment to kids' content, these channels showcase the diversity and popularity of YouTube's audience. Find out which YouTube channel has the most subscribers as of June 2024, and how much money the top creators earn from their videos. See the ranking of the most popular YouTube channels by number of subscribers, and learn more about the platform's history and features. Find out which YouTubers have the most subscribers in 2025, based on the latest data. See the complete list of the top 50 channels, updated weekly, and learn from their strategies and content."
# Assistant:
# Assistant: "That strategy led Leonardo to a fifth-place finish in the Global Top 100 for January 2025.He got 4.7 million new subscribers during the month, bringing his lifetime total up to 44.5 million on his primary channel.That's a lot of congregation members who are hanging onto every word Leonardo uploads. Then there's Bapuji Dashrathbhai Patel, a spiritual leader from India who also cracked this ... Find out which YouTubers have the most subscribers in 2025, based on the latest data from YouTube. See the complete list of the top 50 channels, updated weekly, and learn from their strategies and content. See the ranking of the most popular YouTube channels as of January 2025, based on the number of subscribers. MrBeast topped the list with 343 million subscribers, followed by T-Series and Cocomelon. Scarlett Adams 07 May 2025. Most Subscribed YouTube Channels highlight the global creators and brands dominating the platform with massive follower counts. From entertainment and education to music and kids' content, these channels showcase the diversity of YouTube's audience. Explore what makes them popular, their unique content strategies ... With a subscriber count of around 394 million, MrBeast is the most subscribed YouTube channel as of 2025. He gained huge attention for his remake of Netflix's Squid Game show on his channel. Today, he is the richest YouTuber with a net worth of around $700 million. 2. T-Series: 294M. Total subscribers:"
# Assistant: 현재 가장 유튜브 구독자가 많은 채널은 MrBeast로, 약 394백만 명의 구독자를 보유하고 있습니다.
```

