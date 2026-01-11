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

## Summary of LangGraph
`LangGraph` 에서 대화 기록을 지속적으로 관리하는 것은 `AI` 애플리케이션에서 가장 널리 사용되는 지속성 패턴중 하나이다. 
이 접근법은 사용자와 `AI` 간의 연속적인 상호작용을 가능하게 하여 맥락을 유지하면서 자연스러운 대화 흐름을 제공한다.  

그러나 대화 세션이 연장되고 메시지가 계속 출적될수록, 
전체 대화 히스토리가 언어 모델의 `context window` 에서 차지하는 비중이 급격히 증가한다. 
이러한 현상은 열 가지 부정적인 결과를 초래할 수 있다. 

- 비용 증가 : 더 많은 토큰을 처리해야 하므로 `LLM API` 호출 비용 상승
- 응답 지연 : 긴 컨텍스트 처리로 인한 응답 시간 증가
- 성능 저하 : 컨텍스트가 너무 길어지면 모델의 추론 품질이 떨어질 수 있음
- 메모리 제한 : 컨텍스트 윈도우 한계에 도달할 위험성

이러한 문제를 효과적으로 해결하기 위한 전략 중 하나는 대화 요약 및 압축 기법을 활용하는 것이다. 
이 방법은 누적된 대화 내용을 간결한 요약문으로 변환하고, 이를 최근의 핵심 메시지들과 결합하여 컨텍스트를 최적화하는 접근법이다. 
이를 위해 아래와 같은 단계를 가진다. 

- 임계점 감지 : 대화 길이가 미리 정의된 기준(메시지 개수, 총 토큰 수, 문자 수)을 초과했는지 모니터링
- 지능형 요약 생성 : 대화의 핵심 정보와 맥락을 보존하면서 간결한 요약문을 생성하기 위한 전용 프롬프트 엔지니어링
- 선택적 메시지 정리 : 요약된 내용을 제회하고 가장 최근의 `N`개 메시지만 유지하도록 이전 메시지들을 체계적으로 제거 

이 과저에서 주요한 내용은 `RemoveMessage` 기능을 적절히 활용해 메모리에서 불필요한 대화 기록을 효율적으로 제거하는 것이다. 
이를 통해 시스템 리소스를 최적화하고 대화의 품질을 지속적으로 유지할 수 있다.  

`ask_llm` 은 `messages` 를 사용해 `LLM` 에 요청을 보내는 함수이다. 
다른 점은 대화 요약본이 존재한다면 이를 시스템 메시지로 추가해 대화에 포함시킨다.  

```python
from typing import Literal, Annotated
from langchain_core.messages import SystemMessage, RemoveMessage, HumanMessage
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import MessagesState, StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_google_genai import ChatGoogleGenerativeAI
import os


# 상태 메모리 저장소
memory = MemorySaver()

class SummaryState(MessagesState):
    messages: Annotated[list, add_messages]
    summary: str


# 모델 초기화
os.environ["GOOGLE_API_KEY"] = "AIzaSyBBWl6ZPwjUUxuYallZnZ1cEPqnQGqsVzs"
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

# ask_llm 노드 동작 정의
# 이전 요약본이 존재하면 추가해서 사용하고, 없으면 이전 메시지를 사용한다.

def ask_llm(state: SummaryState):
  # 요약 내용
  summary = state.get("summary", "")

  if summary:
    system_message = f"Summary of conversation earlier: {summary}"
    messages = [SystemMessage(content=system_message)] + state["messages"]
  else:
    messages = state["messages"]

  response = llm.invoke(messages)


  return {"messages" : [response]}
```  

그리고 `should_continue` 함수는 임계점을 감지하고, 
초과했다면 요약이 수행될 수 있도록 한다. 
넘지 않았다면 종료한다. 
예제에서 사용하는 임계점의 기준은 메시지의 수가 6개를 초과하는 것이다.  

```python
# 요약 결정 노드 동작 정의
# 요약이 불필요하다면 END 노드로 이동

def should_continue(state: SummaryState) -> Literal["summarize_conversation", END]:
  # 지금까지 메시지
  messages = state["messages"]

  if len(messages) > 6:
    return "summarize_conversation"
  else:
    return END
```  

`summarize_conversation` 함수는 노드의 대화를 요약하고, 
오래된 메시지는 삭제한다.  

```python
# 요약 생성 및 메시지 삭제 노드 동작 정의
def summarize_conversation(state: SummaryState):
  # 이전 요약 내용
  summary = state.get("summary", "")

  if summary:
    summary_message = (
        f"Here is a summary of the conversation so far: {summary}\n\n"
        "Please update and expand this summary in Korean, taking into account the new messages above:"
    )
  else:
    summary_message = "Based on the messages above, write a summary in Korean."

  # 요약 프롬프트와 메시지 결합
  messages = state["messages"] + [HumanMessage(content=summary_message)]
  # 요약 생성
  response = llm.invoke(messages)
  # 이전 메시지 삭제
  delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]

  return {"summary" : response.content, "messages": delete_messages}
```  

이제 정의한 함수를 노드로 등록해 그래프를 구성한다.  

```python
# 그래프 정의
from IPython.display import Image, display

graph_builder = StateGraph(SummaryState)

graph_builder.add_node("conversation", ask_llm)
graph_builder.add_node(summarize_conversation)

graph_builder.add_edge(START, "conversation")

graph_builder.add_conditional_edges(
    "conversation",
    should_continue,
)

graph_builder.add_edge("summarize_conversation", END)

graph = graph_builder.compile(checkpointer=memory)


# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/summary-subgraph-1.png)
