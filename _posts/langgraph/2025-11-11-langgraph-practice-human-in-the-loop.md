--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Agent Memory and Stream"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: 'LangGraph 에이전트에서 Human-in-the-Loop(HITL) 구현과 상태 수동 업데이트 방법을 알아보자'
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

## LangGraph Human-in-the-Loop
`LangGraph` 에서 `Human-in-the-Loop`(`HITL`) 는 `LLM` 기반 워크플로우 또는 에이전트 플로우를 설계할 떄, 
자동화된 흐름 중간에 `사람의 개입` 이 필요할 때 이를 쉽게 삽입할 수 있도록 만든 기능이다. 
순전히 `LLM` 이나 자동화된 노드들만으로 해결하기 어려운 경우, 특정 지점에서 사람의 판단, 입력, 승인 등을 기다리고 그 결과를 받아 
흐름을 이어갈 수 있게 해준다.  

주요 개념 및 특징은 다음과 같다. 

- 중간 개입
  - `LLM` 이 답변을 만들어냈지만 민감한 이슈나 정확성이 중요한 단계에서는 사람이 검토/수정하도록 할 수 있다. 
  - 그래프의 노드로 `Human Node` 를 넣으면 해당 시점에 사람이 개입해서 입력을 주거나, 승인을 할 때까지 워크플로우가 일시 정지된다. 
- 비동기 처리
  - 실제로는 사람이 답변을 입력할 떄까지 시스템이 기다릴 수 있어야 하므로, `LangGraph` 에서는 이러한 노드를 비동기적으로 처리할 수 있도록 지원한다. 
  - 슬랙, 이메일, 웹 UI, `CLI` 등 다양한 방식으로 사람에게 요청을 보내고, 답변이 돌아오면 워크플로우가 이어진다. 

    
### 에이전트 수행 지속 여부 개입하기
`Human-in-the-Loop` 를 구현하는 방법 중 하나는 `interrupt` 를 사용해 사용자가 자동화된 노드의 수행을 계속할지 결정할 수 있도록 하는 것이다.  
예제 진행을 위해 아래와 같은 질문에 대해 웹검색을 통해 답변을 제공하는 `LangGraph` 에이전트를 구현해 사용한다.  

```python
from typing import Annotated, List, Dict
from typing_extensions import TypedDict
from langchain_core.tools import tool
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_community.tools import DuckDuckGoSearchRun
from langchain_google_genai import ChatGoogleGenerativeAI
import os

# LLM 초기화
os.environ["GOOGLE_API_KEY"] = "api key"
model = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

# 상태 정의
class AgentState(TypedDict):
  messages: Annotated[list, add_messages]

# 도구 정의 및 바인딩
tool = DuckDuckGoSearchRun()
tools = [tool]
model_with_tools = model.bind_tools(tools)

# 챗봇 노드추가
def chatbot(state: AgentState):
  return {"messages" : [model_with_tools.invoke(state["messages"])]}

graph_builder = StateGraph(AgentState)
graph_builder.add_node("chatbot", chatbot)

# 도구 노드 추가
tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)


# 조건부 엣지 추가
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

# 메모리 저장소 초기화
memory = MemorySaver()

# 메모리 저장소를 사용해 그래프 컴파일
graph = graph_builder.compile(checkpointer=memory)
```  

구현한 그래프를 시각화 하면 다음과 같다.  

```python
from IPython.display import Image, display


# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

.. 그림 ..
human-in-the-loop-1.png

구현해 볼 것은 사용자 질문에 대해 `LLM` 이 웹 검색 `Tool` 에 사용할 검색어를 확인 후 지속여부를 결정하는 흐름이다. 
이를 위해 `interrupt_before=tools` 로 지정해 질문을 처리하도록 구성한다.  

```python
from langchain_core.runnables import RunnableConfig

query = "AI 관련 최신 뉴스를 알려주세요."

input = AgentState(messages=["user", query])

config = RunnableConfig(
    recursion_limi=10,
    configurable={'thread_id' : "1"},
    tags=["my-tag"]
)

for event in graph.stream(
    input=input,
    config=config,
    stream_mode="values",
    interrupt_before=["tools"],
):
  for key, value in event.items():
      # key 는 노드 이름
      print(f"\n[{key}]\n")
      # value 는 노드의 출력값
      print(value[-1].pretty_print())

      # value 에는 state 가 dict 형태로 저장(values 의 key 값)
      if "messages" in value:
          print(f"메시지 개수: {len(value['messages'])}")
  print("---" * 10, " 노드완료 ", "---" * 10)
  
# [messages]
# 
# ================================ Human Message =================================
# 
# AI 관련 최신 뉴스를 알려주세요.
# None
# ------------------------------  노드완료  ------------------------------
# 
# [messages]
# 
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (d3a6bbee-31e6-468d-8446-97c9d293925f)
# Call ID: d3a6bbee-31e6-468d-8446-97c9d293925f
# Args:
# query: AI 최신 뉴스
# None
# ------------------------------  노드완료  ------------------------------
```  

그래프가 중단된 상태에서 현 상태를 출력해 확인해 본다.

```python
# interrupt 된 상태 확인
snapshot = graph.get_state(config)

print(snapshot.next)
# ('tools',)

snapshot.values["messages"][-1].pretty_print()
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (09af21e4-a61d-4085-9fb3-20aeca3af5ed)
# Call ID: 09af21e4-a61d-4085-9fb3-20aeca3af5ed
# Args:
# query: AI 최신 뉴스
```  

다음 실행할 노드는 `tools` 노드이고, 웹 검색 도구 노드 수행시 사용할 쿼리는 `AI 최신 뉴스`이다. 
검색 쿼리를 확인했으니, 중단된 그래프를 이어서 시작한다. 
`LangGraph` 에서 중단된 그래프를 이어서 시작하는 방법은 입력에 `None` 을 전달하면 된다.  

```python
# interrupt 된 지점 이후 부터 이어서 그래프 진행
events = graph.stream(None, config, stream_mode="values")

for event in events:
  if "messages" in event:
    event["messages"][-1].pretty_print()

# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (d3a6bbee-31e6-468d-8446-97c9d293925f)
# Call ID: d3a6bbee-31e6-468d-8446-97c9d293925f
# Args:
# query: AI 최신 뉴스
# ================================= Tool Message =================================
# Name: duckduckgo_search
# 
# 인공지능 최신 뉴스 지금 뜨거운 AI 이슈 총정리 빠르게 변화하는 인공지능 (AI) 산업은 매일 새로운 소식을 전하고 있습니다. 최신 기술 동향부터 글로벌 기업의 전략, 그리고 혁신적인 연구 결과까지, 지금 가장 주목해야 할 AI 뉴스를 한눈에 정리했습니다! 1. AI가 우리 삶 곳곳에 활용되면서 AI 반도체에 대한 수요도 빠르게 늘고 있습니다. 관련 시장은 오는 2028년 현재의 두 배 가량으로 확대될 전망인데요. 이 시장에서 미국과 중국, 대만이 선두를 차지하기 위해 싸우고 있는 가운데 ... 타이완 "AI 붐 덕분에 올해 수출 715조 원 넘어 사상 최대 전망" 타이완 북부 지룽 항구세계적인 인공지능 붐으로 올해 타이완의 수출액이 5천177억 ... AI 트렌드와 최신 뉴스 총정리: 지금 주목해야 할 이슈들2025년 상반기, AI는 어디까지 왔을까요? 지금 이 순간, AI가 바꾸고 있는 세계를 살펴보세요.안녕하세요 여러분! 2025년 4월 첫째 주 AI 핵심 뉴스 요약 (총 25건)1. 메타, Llama 4 모델 2종 출시 - Scout와 Maverick 모델 공개2. 자율주행차, 현실적 상용화 전망 - 10~15년 공존 예상3. 마이크로소프트 Copilot 대규모 업데이트 - 개인화 및 시각 인식 강화4. 알리바바, Qwen 3 AI 모델 곧 공개5. 아마존, 최대 규모 AI 데이터센터 계획 ...
# ================================== Ai Message ==================================
# 
# AI 관련 최신 뉴스를 검색한 결과, AI 반도체 수요 증가, 타이완의 AI 붐으로 인한 수출 증가, 그리고 메타, 마이크로소프트, 알리바바, 아마존 등 주요 기업들의 AI 모델 출시 및 업데이트, 데이터센터 계획 등의 소식이 있습니다. 좀 더 자세한 정보를 원하시면, 특정 뉴스에 대해 질문해주세요.
```  

추가로 현재 `LangGraph` 에이전트는 `checkpoint` 를 구현에 추가한 상태이기 때문에 그래프는 무기한 중단되고 언제든지 원하는 시점에 시작할 수 있다. 
`get_state_history()` 메서드를 통해 그래프의 히스트리를 확인해 보고 그 중 특정 시점을 `to_replay` 에 저장한다.  

```python
# checkpointer 를 추가했기 때문에, 그래프는 무기한 일시 중지되고 언제든지 다시 시작할 수 있다.
# get_state_history 메서드를 사용해 상태 기록을 가져오고 그중 특정 시점을 저장한다.

to_replay = None

for state in graph.get_state_history(config):
    print("메시지 수: ", len(state.values["messages"]), "다음 노드: ", state.next)

    if len(state.values["messages"]) == 3:
      to_replay = state

# 저장된 checkpoint 의 다음 실행 노드
print(to_replay.next)
# ('tools',)

# 저장된 checkpoint 의 설정 정보
print(to_replay.config)
# {'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f05991e-93ea-6004-8001-27d8117f3b84'}}
```  

`to_replay.config` 출력값을 보면 `checkpoint_id` 가 포함돼 있는데, 
해당 값을 제공하면 `LangGraph` 에서 해당 체크포인트 상태를 로드해 해당 지점부터 다시 시작할 수 있다. 
이 경우도 입력값은 `None` 으로 전달해야 한다.  

```python
for event in graph.stream(None, to_replay.config, stream_mode="values"):
  if "messages" in event:
    event["messages"][-1].pretty_print()

# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (d3a6bbee-31e6-468d-8446-97c9d293925f)
# Call ID: d3a6bbee-31e6-468d-8446-97c9d293925f
# Args:
# query: AI 최신 뉴스
# ================================= Tool Message =================================
# Name: duckduckgo_search
# 
# 타이완 "AI 붐 덕분에 올해 수출 715조 원 넘어 사상 최대 전망" 타이완 북부 지룽 항구세계적인 인공지능 붐으로 올해 타이완의 수출액이 5천177억 ... 인공지능 최신 뉴스 지금 뜨거운 AI 이슈 총정리 빠르게 변화하는 인공지능 (AI) 산업은 매일 새로운 소식을 전하고 있습니다. 최신 기술 동향부터 글로벌 기업의 전략, 그리고 혁신적인 연구 결과까지, 지금 가장 주목해야 할 AI 뉴스를 한눈에 정리했습니다! 1. AI 트렌드와 최신 뉴스 총정리: 지금 주목해야 할 이슈들2025년 상반기, AI는 어디까지 왔을까요? 지금 이 순간, AI가 바꾸고 있는 세계를 살펴보세요.안녕하세요 여러분! "AI를 눈으로 볼 수 있게"…빅테크들 앞다퉈 '올인'한 기술은?, 생성형 AI 발전에 피지컬 AI 부상 '휴머노이드 로봇' 등 핵심 기술 글로벌 빅테크, 자체 ... AI 최신 소식 오늘의 AI 핵심 뉴스 TOP 5 - 정부부터 마이크로소프트까지, 오늘의 결정적 변화 smartupgrade 2025. 5. 20. 20:00 AI는 이제 하루 단위로 산업 지형을 바꾸고 있습니다.
# ================================== Ai Message ==================================
# 
# 최신 AI 뉴스에 따르면, 타이완은 AI 붐으로 인해 올해 수출액이 사상 최대치를 기록할 것으로 예상됩니다. 또한, 생성형 AI 발전에 따라 피지컬 AI, 특히 휴머노이드 로봇이 부상하고 있다는 소식도 있습니다. AI 기술은 현재 매우 빠른 속도로 발전하고 있으며, 빅테크 기업들은 AI 기술에 많은 투자를 하고 있습니다.
```  

### 에이전트 상태 수동 업데이트
`LangGraph` 에서는 중간단계 개입 되돌림을 통해 상태 수정과 `Replay` 를 구현할 수 있다. 
이는 `LLM` 기반 자동화 워크플로우에서 신뢰성과 유연성을 높이기 위해 도입한 중요한 개념이다. 
워크플로우를 실행하다가 특정 단계(노드)에서 사람이 개입해야 하거나, 잘못된 상태가 감지된 경우, 해당 시점으로 되돌아가서 상태를 수정하고 다시 진행할 수 있다.  

- 실행 중단/개입 : `LLM` 답변이 부적절하거나, 도구 호출 결과가 잘못된 경우, 해당 시점에서 자동화를 멈추고 사람이 개입하여 상태를 직접 수정할 수 있다. 
- 상태 수정 : 사람이 직접 `state` 를 수정하여 잘못된 정보, 누락된 정보 등을 보완할 수 있다. 
- 되돌림(`Rollback`) : 각 단계 `state` 를 저장하고 있기 때문에, 원하는 이전 단계로 되돌아가(`Rollback`) 그 시점의 `state` 부터 `Replay` 가 가능하다.  

`LangGraph` 에이전트는 앞선 예제와 동일한 것을 사용한다.  
이번 예제에서는 에이전트가 웹 검색 도구에 사용하는 상태를 수정해 적합한 최종 답변을 제공할 수 있도록 구현해 본다.  

```python
from langchain_core.runnables import RunnableConfig

query = "LangGraph 가 무엇인지 조사해 알려줘"

input = AgentState(messages=[("user", query)])

config = RunnableConfig(
    configurable={"thread_id": "2"}
)

# interrupt_before, interrupt_after 를 적용할 수 있는 목록
print(graph.nodes.keys())
# dict_keys(['__start__', 'chatbot', 'tools'])

events = graph.stream(
    input=input, config=config, interrupt_before=["tools"], stream_mode="values"
)

for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()

# ================================ Human Message =================================
# 
# LangGraph 가 무엇인지 조사해 알려줘
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (f245c634-891f-4a2f-968d-8942e79b4e3e)
# Call ID: f245c634-891f-4a2f-968d-8942e79b4e3e
# Args:
# query: LangGraph
```  

그래프에서 중단된 지점의 스냅샷을 가져와 가장 최근 메시지를 확인하면, 
에이전트가 웹 검색 도구에 사용할 검색 쿼리를 확인할 수 있다.  

```python
snapshot = graph.get_state(config)

last_message = snapshot.values["messages"][-1]

last_message.pretty_print()
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (f245c634-891f-4a2f-968d-8942e79b4e3e)
# Call ID: f245c634-891f-4a2f-968d-8942e79b4e3e
# Args:
# query: LangGraph
```  

#### 노드의 결과 수정하기
현재 중단된 지점은 웹 검색 도구 노드이다. 
이때 관리자가 보다 적합한 검색 결과를 직접 개입하는 경우에 대해 살펴본다.  

웹 검색 도구의 결과로 관리자가 입력한 아래와 같은 가상 검색 결과를 제공한다고 가정한다.  

```python
modified_search_result = """[수정된 웹 검색 결과]
문의 내용의 답변은 다음과 같습니다.
직접 알아보세요!
"""
```  

도구 노드의 결과를 직접 수정하기 위해서는 동일한 `tool_call_id` 를 사용해야 한다. 
아래와 같은 방법으로 현재 도구 노드의 `tool_call_id` 를 가져올 수 있다.  

```python
# 메시지를 수정하려면 수정하고자 하는 Message 와 일치하는 tool_call_id 를 지정해야 한다.
tool_call_id = last_message.tool_calls[0]["id"]
# f245c634-891f-4a2f-968d-8942e79b4e3e
```  

이제 해당 `tool_call_id` 를 사용해 상태 업데이트에 사용할 `ToolMessage` 를 생성한다.  

```python
from langchain_core.messages import AIMessage, ToolMessage

new_messages = [
    ToolMessage(
        content=modified_search_result,
        tool_call_id=tool_call_id
    )
]

new_messages[-1].pretty_print()
# ================================= Tool Message =================================
# 
# [수정된 웹 검색 결과]
# 문의 내용의 답변은 다음과 같습니다.
# 직접 알아보세요!
```  

만들어진 `TollMessage` 로 그래프의 상태를 업데이트해 주면된다. 
이때 `StateGraph` 의 [update_state](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.CompiledStateGraph.update_state)
메서드를 사용한다.  

`update_state` 메서드를 간략하게 설명하면 다음과 같다. 

메개 변수는 아래와 같다. 

- `config` : `RunnableConfig` 객체로, 그래프 실행에 필요한 설정 정보를 포함한다.
- `values` : `State` 의 값을 포함하는 딕셔너리로, 그래프의 현재 상태를 나타낸다.
- `as_node` : `string` 타입으로 값의 출처로 간주할 노드 이름을 지정한다. 

주요 특징은 아래와 같다. 

- 체크포인터를 통해 이전 상태를 로드하고 새로운 상태를 저장한다. 
- 서브그래프에 대한 상태 업데이트를 처리한다. 
- `as_node` 가 지정되지 않은 경우, 마지막으로 상태를 업데이트한 노드를 찾는다. 
- 그래프의 상태를 수동으로 업데이트할 때 사용한다. 
- 상태 업데이트 중 `SharedValues` 에 쓰기 작업은 허용되지 않는다.  

```python
# update_state 를 사용해 상태 업데이트

graph.update_state(
    config,
    {"messages" : new_messages},
    as_node="tools"
)

print(graph.get_state(config).values["messages"][-1])
# ToolMessage(content='[수정된 웹 검색 결과] \n문의 내용의 답변은 다음과 같습니다. \n직접 알아보세요!\n', id='902b0354-4be8-491c-8e80-bd8c388d29fa', tool_call_id='f245c634-891f-4a2f-968d-8942e79b4e3e')

# 상태 수동 업데이트 후 현재 그래프 상태 확인
snapshot = graph.get_state(config)
print(snapshot.next)
# ('chatbot',)
```  

상태를 수동 업데이트하고 그래프에서 실행할 다음 노드를 보면 `chatbot` 으로 다음 노드로 이동한 것을 확인할 수 있다. 
이는 `update_state` 는 특정 시점의 체크포인트 `state` 를 불러와 수정 하면 해당 노드의 결과는 업데이트된 상태의 것으로 간주하고, 
`Replay` 를 수행하면 수정된 시점 이후의 노드부터 실행된다.  

이제 중단된 그래프를 다시 실행하면 아래와 같이, 사용자가 수정한 검색 결과를 기반으로 최종 답변이 나오는 것을 확인할 수 있다.  

```python
# 수정된 메시지를 바탕으로 이어서 수행

events = graph.stream(None, config, stream_mode="values")

for event in events:
  if "messages" in event:
    event["messages"][-1].pretty_print()
# ================================= Tool Message =================================
# 
# [수정된 웹 검색 결과]
# 문의 내용의 답변은 다음과 같습니다.
# 직접 알아보세요!
# 
# ================================== Ai Message ==================================
# 
# LangGraph에 대한 정보를 직접 검색해 보시는 것이 좋겠습니다.

# 만약 최종 메시지까지 수정을 원하다면 아래와 같이 할 수 있다. 
# graph.update_state(
#     config,
#     {
#         "messages" : [
#             AIMessage(content="Google 에서 아래 제시된 키워드로 검색해 보시는 걸 추천 드립니다. ...")
#         ]
#     }
# )
```  

마지막으로 상태 수동 업데이트가 `checkpoint` 에 정상적으로 반영됐는지 확인하면 아래와 같다.  

```python
# 그래프 상태 스냅샷 생성
snapshot = graph.get_state(config)

# 최근 세 개의 메시지 출력
for message in snapshot.values["messages"]:
    message.pretty_print()
# ================================ Human Message =================================
# 
# LangGraph 가 무엇인지 조사해 알려줘
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (f245c634-891f-4a2f-968d-8942e79b4e3e)
# Call ID: f245c634-891f-4a2f-968d-8942e79b4e3e
# Args:
# query: LangGraph
# ================================= Tool Message =================================
# 
# [수정된 웹 검색 결과]
# 문의 내용의 답변은 다음과 같습니다.
# 직접 알아보세요!
# 
# ================================== Ai Message ==================================
# 
# LangGraph에 대한 정보를 직접 검색해 보시는 것이 좋겠습니다.

# 다음 실행 노드, 모두 실행된 상태로 상태가 업데이트 됨
print(snapshot.next)
# ()
```  

#### 노드의 입력값 수정하기
이번에는 웹 검색 도구의 쿼리를 수정해 그래프가 실행되도록 해본다. 
랜던 값을 사용해 `thread_id` 를 지정해서 체크포인터가 구분될 수 있도록 하고, 
동일하게 `tools` 노드 이전에 중단되도록 해 그래프를 실행한다.  

```python
import random

thread_id = random.randrange(1, 99999999999)

query = "LangGraph 가 무엇인지 조사해 알려줘"

input = AgentState(messages=[("user", query)])

config = {"configurable": {"thread_id" : thread_id}}

events = graph.stream(
    input=input,
    config=config,
    interrupt_before=["tools"],
    stream_mode="values"
)

for event in events:
  if "messages" in event:
    event["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# LangGraph 가 무엇인지 조사해 알려줘
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (b81a82bd-01a8-4c1b-9b7b-964192b397b2)
# Call ID: b81a82bd-01a8-4c1b-9b7b-964192b397b2
# Args:
# query: LangGraph
```  

상태의 마지막 메시지를 확인하고, 웹 검색 도구의 쿼리 매개변수를 수정한다. 
그리고 이를 `AIMessage` 로 만들어 상태를 업데이트 한다. 
이때 중요한 점은 기존 메시지 `id` 를 그대로 사용해야 한다는 점이다.  

```python
from langchain_core.messages import AIMessage

snapshot = graph.get_state(config)

# messages 마지막 메시지
existing_message = snapshot.values["messages"][-1]

# 마지막 메시지 id
print(existing_message.id)
# run--1a65c077-5c72-457b-bbf9-47e7aeeabc2f-0

# 마지막 메시지의 첫번째 도구
print(existing_message.tool_calls[0])
# {'name': 'duckduckgo_search', 'args': {'query': 'LangGraph'}, 'id': 'b81a82bd-01a8-4c1b-9b7b-964192b397b2', 'type': 'tool_call'}

# 위 마지막 메시지의 첫번째 도구의 query 를 수정해 본다.

# tool_calls 를 복사해 새로운 도구 호출 생성
new_tool_call = existing_message.tool_calls[0].copy()

# 쿼리 메개변수 수정
new_tool_call["args"] = {"query" : "LangGraph checkpointing"}
print(new_tool_call)
# {'name': 'duckduckgo_search',
#  'args': {'query': 'LangGraph checkpointing'},
#  'id': 'b81a82bd-01a8-4c1b-9b7b-964192b397b2',
#  'type': 'tool_call'}

new_message = AIMessage(
    content=existing_message.content,
    tool_calls=[new_tool_call],
    id=existing_message.id
)

# 웹 검색 도구에서 사용할 업데이트된 메시지
new_message.pretty_print()
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (b81a82bd-01a8-4c1b-9b7b-964192b397b2)
# Call ID: b81a82bd-01a8-4c1b-9b7b-964192b397b2
# Args:
# query: LangGraph checkpointing


# 업데이트 된 메시지로 상태 업데이트
graph.update_state(config, {"messages":[new_message]})
# {'configurable': {'thread_id': '3145555862',
#                   'checkpoint_ns': '',
#                   'checkpoint_id': '1f054d8d-099a-607d-8002-656231f0c8cb'}}
```  

최종적으로 그래프 상태에서 마지막 메시지의 `tool_calls` 를 확인해 상태 수정이 잘 됐는지 확인한다.  

```python
# 업데이트 후 그래프의 상태 중 마지막 메시지의 도구
graph.get_state(config).values["messages"][-1].tool_calls
# [{'name': 'duckduckgo_search',
#   'args': {'query': 'LangGraph checkpointing'},
#   'id': 'b81a82bd-01a8-4c1b-9b7b-964192b397b2',
#   'type': 'tool_call'}]
```  

이제 업데이트된 상태로 다시 그래프를 이어서 진행한다. 
그러면 앞서 수정한 검색 쿼리로 그래프가 이어 수행되는 것을 확인할 수 있다. 

```python
# 그래프 스트림 이어서 수행
events = graph.stream(None, config, stream_mode="values")

for event in events:
  if "messages" in event:
    event["messages"][-1].pretty_print()
# ================================== Ai Message ==================================
# Tool Calls:
# duckduckgo_search (b81a82bd-01a8-4c1b-9b7b-964192b397b2)
# Call ID: b81a82bd-01a8-4c1b-9b7b-964192b397b2
# Args:
# query: LangGraph checkpointing
# ================================= Tool Message =================================
# Name: duckduckgo_search
# 
# Thread Threads enable the checkpointing of multiple different runs, making them essential for multi-tenant chat applications and other scenarios where maintaining separate states is necessary. A thread is a unique ID assigned to a series of checkpoints saved by a checkpointer. Hello everyone, I am a beginner in databases. If I use Postgres as a checkpointer in a production environment, how should I correctly use Langgraph in my API when using a web framework? (I am using FastAPI) Create a single global graph object and use only one checkpointer. Create a new graph object for each API request but use the same checkpointer. Create a new graph object for each API ... The checkpoint mechanism and human-computer interaction features of LangGraph provide powerful tools for building complex and reliable AI systems. By using these features wisely, we can create more intelligent, flexible, and controllable applications. LangGraph is an orchestration framework for complex agentic systems and is more low-level and controllable than LangChain agents. On the other hand, LangChain provides a standard interface to ... We've released LangGraph v0.2 for increased customization with new checkpointer libraries. LangGraph v0.2 allows you to tailor stateful LangGraph apps to...
# ================================== Ai Message ==================================
# 
# LangGraph는 복잡한 에이전트 시스템을 위한 오케스트레이션 프레임워크입니다. LangChain 에이전트보다 더 낮은 수준에서 제어가 가능합니다. LangGraph는 여러 실행의 체크포인트를 가능하게 하여, 멀티 테넌트 채팅 애플리케이션과 같이 별도의 상태를 유지해야 하는 시나리오에 필수적입니다. 또한, LangGraph v0.2는 새로운 체크포인터 라이브러리를 통해 사용자 정의를 향상시켰습니다.
```  

체크포인터에 업데이트된 상태가 잘 반영 됐는지 확인하기 위해 동일한 `thread_id` 로 추가 질문을 이어서 수행해 본다. 
그러면 요청한 출처 정보에 수정한 검색 쿼리가 나오는 것을 볼 수 있고, 이를 통해 잘 상태가 수정되고 체크포인터에도 반영됐음을 확인할 수 있다.  

```python
events = graph.stream(
    {
        "messages" : (
            "user",
            "지금까지 질문을 모두 하나로 정리해줘 그리고 출처도 포함해줘"
        )
    },
    config,
    stream_mode="values"
)

for event in events:
  if "messages" in event:
    event["messages"][-1].pretty_print()
# ================================ Human Message =================================
# 
# 지금까지 질문을 모두 하나로 정리해줘 그리고 출처도 포함해줘
# ================================== Ai Message ==================================
# 
# LangGraph는 복잡한 에이전트 시스템을 위한 오케스트레이션 프레임워크이며, LangChain 에이전트보다 더 낮은 수준에서 제어가 가능합니다. LangGraph는 여러 실행의 체크포인트를 가능하게 하여, 멀티 테넌트 채팅 애플리케이션과 같이 별도의 상태를 유지해야 하는 시나리오에 필수적입니다. 또한, LangGraph v0.2는 새로운 체크포인터 라이브러리를 통해 사용자 정의를 향상시켰습니다. (출처: DuckDuckGo 검색 "LangGraph checkpointing")
```  
