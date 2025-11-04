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



### Agent Memory
앞서 구현한 `Agent` 는 실시간 검색 도구를 사용할 수 있는 `Agent` 이지만, 
과거 대화의 내용을 기억하지 못해 `Context` 가 필요한 `Multi-Turn` 대화가 불가능하다.  

이를 해결하기 위해 `LangGraph` 의 `Persistent Checkpoint` 를 사용해 
그래프 호출시 `thread_id` 를 지정해 `thread_id` 별로 상태를 유지할 수 있도록 한다. 
이를 사용하면 `Agent` 가 지금까지 대화 맥락을 이해하고 그에 맞춤 답변을 제공할 수 있다.  

`LangGraph` 에서 제공하는 `Persistent Checkpoint` 기능은 `LangChain` 에서 제공하는 
`Memory` 기능보다 훨씬 강력하다. 
주요 차이점을 표로 정리하면 다음과 같다.  

구분|Persistent Checkpoint (LangGraph)|Memory (LangChain)
---|---|---
지속성|영구적(외부 저장)|임시적(메모리 기반)
복구/재시작|중단/재시작 완벽 지원|세션 종료 시 데이터 소멸
확장성|대용량, 복잡 워크플로도 지원|짧고 단순한 대화에 적합
구현 난이도|상대적으로 복잡(저장소 연동 필요)|상대적으로 간단
관리/운용|상태 관리·운용에 특화|간단한 세션 컨텍스트 관리

`Persistent Checkpoint` 를 사용해 `Multi-Turn` 대화가 가능한 `Agent` 구현은 
이전 `Agent` 에서 언급한 `LangGraph` 에서 제공하는 `ToolNode` 와 `tools_condition` 을 사용한다. 
그리고 간단한 구현을 위해 메모리 기반으로 진행한다.  

`in-memory checkpointer` 를 정의하고 기존 `Agent` 구현을 `ToolNode` 와 
`tools_condition` 기반으로 수정 후 메모리 기능을 `Agent` 에 추가하면 아래와 같다.  

```python
from langgraph.checkpoint.memory import MemorySaver
from typing import Annotated
from typing_extensions import TypedDict
from langchain_community.tools import DuckDuckGoSearchRun
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langchain.chat_models import init_chat_model
from langchain_google_genai import ChatGoogleGenerativeAI
import os

# in-memory checkpointer 정의
memory = MemorySaver()


# 상태 정의
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

# 도구 초기화
tool = DuckDuckGoSearchRun()
tools = [tool]

# LLM 초기화
os.environ["GOOGLE_API_KEY"] = "api key"
model = ChatGoogleGenerativeAI(model="gemini-2.0-flash")


# LLM 도구 바인딩
model_with_tools = model.bind_tools(tools)


# 챗봇 함수 정의
def chatbot(state: AgentState):
    return {"messages" : [model_with_tools.invoke(state["messages"])]}

# 상태 그래프 생성
graph_builder = StateGraph(AgentState)

# 챗봇 노드 추가
graph_builder.add_node("chatbot", chatbot)

# 도구 노드 추가
tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

# 조건부 엣지
graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition
)

# tools -> chatbot 엣지 추가
graph_builder.add_edge("tools", "chatbot")

# START -> chatbot 엣지 추가
graph_builder.add_edge(START, "chatbot")

# chatbot -> END 엣지 추가
graph_builder.add_edge("chatbot", END)

# checkpointer를 사용해 상태 그래프 컴파일
graph = graph_builder.compile(checkpointer=memory)
```  

구현된 그패르르 시각화하면 이전 `Agent` 와 동일하다. 
차이점은 그래프가 각 노드를 처리하면서 `State` 를 체크포인트 하는 것이다.  

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

`checkpointing` 기능을 사용해 `Agent` 를 호출하기 위해서는 `thread_id` 를 지정해야 하기 때문에, 
`RunnableConfig` 를 정의해 사용한다.  

```python
from langchain_core.runnables import RunnableConfig

config = RunnableConfig(
    # 최대 10개 노드 방문
    recursion_limit=10,
    # 대화 세션을 구분할 스레드 ID
    configurable={"thread_id": "1"}
)


def answer_print(question, config):
  for event in graph.stream({"messages": [("user", question)]}, config=config):
      for value in event.values():
          value["messages"][-1].pretty_print()

answer_print("안녕 내이름은 jack 이야 반가워", config)
# ================================== Ai Message ==================================
# 
# 안녕하세요, Jack님. 만나서 반갑습니다! 어떤 도움이 필요하신가요?
```  

동일한 `thread_id` 를 사용하는 경우 이전 대화를 기억해 답변하지만 다르면 기억하지 못한다.  

```python
config = RunnableConfig(
    recursion_limit=10,
    configurable={"thread_id": "2"},
)

answer_print("내 이름이 뭐라고 ?", config)
# ================================== Ai Message ==================================
# 
# 저는 당신의 이름을 모릅니다. 저는 대규모 언어 모델입니다.

config = RunnableConfig(
    recursion_limit=10,
    configurable={"thread_id": "1"},
)

answer_print("내 이름이 뭐라고 ?", config)
# ================================== Ai Message ==================================
# 
# 당신의 이름은 Jack입니다.
```  

저장된 `State` 를 확인하고 싶은 경우 `thread_id` 를 지정해 `get_state()` 함수를 통해 확인해 볼 수 있다.  

```python
from langchain_core.runnables import RunnableConfig

config = RunnableConfig(
    configurable={"thread_id": "1"}
)

snapshot = graph.get_state(config)
# 메시지 확인
snapshot.values["messages"]
# [HumanMessage(content='안녕 내이름은 jack 이야', additional_kwargs={}, response_metadata={}, id='a657b2ed-d7c5-4763-8ea2-67f9ef02dd32'),
#  AIMessage(content='<function=duckduckgo_search({"query": "jack"})</function>', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 18, 'prompt_tokens': 274, 'total_tokens': 292, 'completion_time': 0.065454545, 'prompt_time': 0.017705171, 'queue_time': 0.22546059400000001, 'total_time': 0.083159716}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'stop', 'logprobs': None}, id='run--50fcef24-c195-409c-8c63-fe6c6c10dde2-0', usage_metadata={'input_tokens': 274, 'output_tokens': 18, 'total_tokens': 292}),
#  HumanMessage(content='내 이름이 뭐라고 ?', additional_kwargs={}, response_metadata={}, id='45972f3c-e413-4d20-9463-68c9bde0a059'),
#  AIMessage(content='당신의 이름은 Jack입니다.', additional_kwargs={}, response_metadata={'prompt_feedback': {'block_reason': 0, 'safety_ratings': []}, 'finish_reason': 'STOP', 'model_name': 'gemini-2.0-flash', 'safety_ratings': []}, id='run--bbe24502-92d3-4893-8a04-2b77e5eb8125-0', usage_metadata={'input_tokens': 198, 'output_tokens': 8, 'total_tokens': 206, 'input_token_details': {'cache_read': 0}})]

# 설정된 config 확인
snapshot.config
# {'configurable': {'thread_id': '1',
#                   'checkpoint_ns': '',
#                  'checkpoint_id': '1f049d38-9337-689d-8010-2fbde531c2a3'}}

# 저장된 값 확인
snapshot.values
# {'messages': [HumanMessage(content='안녕 내이름은 jack 이야', additional_kwargs={}, response_metadata={}, id='a657b2ed-d7c5-4763-8ea2-67f9ef02dd32'),
#               AIMessage(content='<function=duckduckgo_search({"query": "jack"})</function>', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 18, 'prompt_tokens': 274, 'total_tokens': 292, 'completion_time': 0.065454545, 'prompt_time': 0.017705171, 'queue_time': 0.22546059400000001, 'total_time': 0.083159716}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'stop', 'logprobs': None}, id='run--50fcef24-c195-409c-8c63-fe6c6c10dde2-0', usage_metadata={'input_tokens': 274, 'output_tokens': 18, 'total_tokens': 292}),
#               HumanMessage(content='내 이름이 뭐라고 ?', additional_kwargs={}, response_metadata={}, id='45972f3c-e413-4d20-9463-68c9bde0a059'),
#              AIMessage(content='당신의 이름은 Jack입니다.', additional_kwargs={}, response_metadata={'prompt_feedback': {'block_reason': 0, 'safety_ratings': []}, 'finish_reason': 'STOP', 'model_name': 'gemini-2.0-flash', 'safety_ratings': []}, id='run--bbe24502-92d3-4893-8a04-2b77e5eb8125-0', usage_metadata={'input_tokens': 198, 'output_tokens': 8, 'total_tokens': 206, 'input_token_details': {'cache_read': 0}})]}

# 다음 노드 확인
# 현재 END 로 종료되어 빈값
snapshot.next
# {}
```  

### LangGraph Stream
지금가지 구현한 `Chatbot`, `Agent` 에서 사용자 질문을 전달하고 출력을 위해 사용한 함수는 `stream()` 이다. 
`stream()` 함수는 단순히 입력을 전달하고 최종 답변을 전달하는 것 외에도 다른 매개변수들이 있는데 이것들에 대해 알아본다. 
사용할 예제는 `checkpointing` 기능을 사용한 `Agent` 구현에서 `State` 정의에 `extra_data` 만 추가한다.  

```python
# 상태 정의
class AgentState(TypedDict):
  extra_data: Annotated[str, "추가 데이터"]
  messages: Annotated[list, add_messages]

# 챗봇 함수 정의
def chatbot(state: AgentState):
    return {
        "messages" : [model_with_tools.invoke(state["messages"])],
        "extra_data" : "chatbot extra data 메시지"
    }
```

`StateGraph` 의 `stream()` 함수는 개르프를 한 번에 끝까지 실행하는 것이 아니라, 
한 단계씩, 실시간으로 결과를 순서대로 내보내는 기능이다. 
즉, `Agent` 의 처리 과정을 스트리밍 형태로 외부에서 받아볼 수 있게 제공한다. 
이는 한 번에 모든 결과를 기다리지 않고, 실행 도중 각 단계별 상태나 결과를 바로바로 확인하거나, 
사용자에게 스트리밍 방식으로 보여주고 싶을 때 사용할 수 있다. 
`stream()` 함수는 다음과 같은 매개변수를 갖는다.  

- `input`(필수) : 그래프 실행을 시작할 때 넣어주는 입력값이다. 딕셔너리 또는 다일 값이 될 수 있다. 
- `config`(선택) : 그래프 실행에 필요한 추가 설정 정보(시간 제한, 환경 변수, 콜백 등)를 담는 객체이다. 일반적으로 특별한 설정이 없으면 생략 가능하다. 
- `stream_mode`(선택) : 스트리밍의 방식을 지정한다. 여러 모드를 조합할 수도 있다. (`["values", "debug"]`)
- `output_key`(선택) : 스트리밍 시, 출력할 데이터의 `키(이름)`를 지정한다. 여러 데이터 중 `answer` 만 스트리밍 하고 싶은 경우 해당 키를 지정해 주면된다. 
- `interrupt_before`(선택) : 지정한 노드(들) 실행 이전에 실행을 잠시 멈추고, 외부에서 개입할 수 있게 하는 옵션이다. `All` 을 지정하면 모든 노드 실행 전마다 중단한다. 
- `interrupt_after`(선택) : 지정한 노드(들) 실행 이후에 실행을 잠시 멈추고, 외부에서 개입할 수 있게 하는 옵션이다. `All` 을 지정하면 모든 노드 실행 후마다 중단한다. 
- `debug`(선택) : `True` 로 설정하면, 실행 과정에서 디버그 정보를 함께 출력한다. 
- `subgraph`(선택, 기본 False) : 그래프 안에 또 다른 하위 그래프가 있을 때, 그 내부의 실행 과정까지 스트리밍할지 여부이다. 

`stream()` 의 반환값은 `Iterator` 형태로, 그래프의 각 단계별 출력 결과를 차례대로 제공한다. 
각 출력값의 형태를 `stream_mode` 설정에 따라 달라진다. 

`stream_mode` 의 종류는 아래와 같은 것들이 있다. 

- `values` : 각 단계에서 현재 전체 상태 값을 출력한다. (지금까지 어떤 데이터가 쌓였는지 전체를 보여 준다.)
- `updates` : 각 단계에서 방금 바뀐 부분만 출력한다. (변경점만 신속하게 확인)
- `debug` : 각 단계에서 디버그 이벤트 정보(어떤 노드에 무슨 일이 있었는지, 오류 등)를 출력한다.  

`stream()` 함수의 사용 예시는 아래와 같다.  

```python

from langchain_core.runnables import RunnableConfig

question = "현재 실시간 비트코인 가격을 알려줘"

input = AgentState(extra_data="추가 데이터", messages=[("user", question)])

config = RunnableConfig(
    recursion_limit=10,
    configurable={"thread_id": "10"},
    tags=["my-tag"]
)

for event in graph.stream(input=input, config=config):
    for key, value in event.items():
        print(f"\n[{key}]\n")

        if "messages" in value:
            messages = value["messages"]
            value["messages"][-1].pretty_print()


# [chatbot]
# 
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (2850865c-b0b7-4c36-ab76-7d959860059d)
# Call ID: 2850865c-b0b7-4c36-ab76-7d959860059d
# Args:
# query: 현재 비트코인 가격
# 
# [tools]
# 
# ================================= Tool Message =================================
# Name: duckduckgo_search
# 
# 106,885.76 btc/usd - 비트코인 시세 실시간 - 코인힐스는 비트코인을 중심으로하는 다양한 디지털 화폐의 시세와 인덱스 정보를 쉽고 편리한 사용자 환경을 통하여, 실시간 제공하고 있습니다. 더블어, 전세계 비트코인 및 디지털 화폐 거래소와 개별 코인의 통계 데이터를 제공합니다. 미래에 비트코인의 가치가 얼마나 될지 알아보고 투자 결정을 내리세요 비트코인 가격 예측 ... 현재 가격은 ₩143,455,835.00입니다. 최근 24시간 동안 가격이 -1.01%에 의해 변경되었습니다. 유통중인 19,881,162.00 코인이 있습니다. 뉴스레터에 가입하세요 ... 비트코인 가격 오늘: 트럼프 상호 관세 앞두고 $86k 아래로 하락; PCE 데이터 주목 ... 비트코인 네트워크는 공공 거래 장부인 "블록체인(Blockchain)을 공유합니다. 이 장부는 처리된 모든 거래를 기록하여 사용자의 컴퓨터가 각 거래의 유효성을 확인할 수 있도록 ... 알려진 최초의 비트코인 상업 거래는 2010년 5월 22일에 프로그래머 라즐로 하넥츠가 10,000개의 비트코인을 피자 두 판과 교환하면서 이루어졌습니다. 2021년 9월 중순 현재 비트코인 가격으로 이 피자는 무려 4억 7,800만 달러의 가치가 있습니다. Trade BTC to USDT and other cryptocurrencies in the world's largest cryptocurrency exchange. Find real-time live price with technical indicators to help you analyze BTC/USDT changes.
# 
# [chatbot]
# 
# ================================== Ai Message ==================================
# 
# 현재 비트코인 가격은 106,885.76 btc/usd 이며, 현재 가격은 ₩143,455,835.00입니다. 최근 24시간 동안 가격이 -1.01% 변동되었습니다.
```  

#### output_keys
`output_keys` 옵션은 스트리밍할 키를 지정하는 옵션이다. 
`list` 형태로 `channels` 에 정의된 키 중 선택해 사용해야 한다.  

```python
# output_keys
# state key 출력
list(graph.channels.keys())
# ['extra_data', 'messages', '__start__', 'branch:to:chatbot', 'branch:to:tools']
```  

아래는 `extra_data` 키를 지정해 스트리밍하는 예시이다. 

```python
from langchain_core.runnables import RunnableConfig

question = "현재 실시간 비트코인 가격을 알려줘"

input = AgentState(extra_data="추가 데이터", messages=[("user", question)])

config = RunnableConfig(
    recursion_limit=10,
    configurable={"thread_id": "10"},
    tags=["my-tag"]
)


for event in graph.stream(
    input=input,
    config=config,
    output_keys=["extra_data"]
):
  for key, value in event.items():
    print(f"\n[{key}]\n")
    if value:
      print(value.keys())
      if "extra_data" in value:
        print(value["extra_data"])

# [chatbot]
# 
# dict_keys(['extra_data'])
# chatbot dummay data 메시지
# 
# [tools]
# 
# 
# [chatbot]
# 
# dict_keys(['extra_data'])
# chatbot dummay data 메시지
```

아래는 `messages` 키를 지정해 스트리밍하는 예시이다. 

```python
from langchain_core.runnables import RunnableConfig

question = "현재 실시간 비트코인 가격을 알려줘"

input = AgentState(extra_data="추가 데이터", messages=[("user", question)])

config = RunnableConfig(
    recursion_limit=10,
    configurable={"thread_id": "10"},
    tags=["my-tag"]
)


for event in graph.stream(
    input=input,
    config=config,
    output_keys=["messages"]
):
  for key, value in event.items():
    if value and "messages" in value:
      print(f"\n[{key}]\n")
      print(value["messages"][-1].content)


# [chatbot]
# 
# 
# 
# [tools]
# 
# Bitcoin (BTC) price, live charts, news and more. Bitcoin to USD price is updated in real time. Learn about Bitcoin, receive market updates and more. Get the latest Bitcoin price in USD, currently at 102,435.00, live chart, 24h stats, market cap, trading volume, and real-time updates. ... Together, these five entities hold nearly 1.2 million BTC valued at about $36 billion at the current price. This is more than 5.6% of Bitcoin's total supply. Among the corporate holders, Grayscale has the ... The current price of Bitcoin in United States is $104,707.98 per (BTC / USD) Get the latest price, news, live charts, and market trends about Bitcoin. Get up to $200 for getting started The Kitco Bitcoin price index provides the latest Bitcoin price in US Dollars using an average from the world's leading exchanges. The live price of Bitcoin today is $104.223K, with a current market cap of $2.066T. The 24-hour trading volume is 30,112M. The price of BTC to USD is updated in real time.
# 
# [chatbot]
# 
# 현재 비트코인 가격은 약 102,435.00 달러에서 104,707.98 달러 사이입니다. 이는 실시간으로 업데이트되는 가격이며, 변동될 수 있습니다.
```  
