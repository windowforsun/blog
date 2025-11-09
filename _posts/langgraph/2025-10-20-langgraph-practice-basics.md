--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Basics"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LangGraph 사용에 앞서 필요한 기본 지식을 설명하고, 이를 바탕으로 간단한 Chatbot 을 구현해 본다.'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - Multi-Agent
    - LLM
    - AI
    - Graph
    - AI Framework
    - TypedDict
    - Annotated
    - add_messages
    - StateGraph
    - Node
    - Edge
toc: true
use_math: true
---  


## LangGraph Basics
`LangGraph` 사용에 앞서 필요한 기본 지식을 설명하고, 
이를 바탕으로 간단한 `Chatbot` 을 구현해 본다.  

### TypedDict
`TypedDict` 는 `LangGraph` 에서 사용하는 `State` 의 타입을 명확하게 생성하는 
`Python` 의 타입 힌트 도구이다. 
이는 `Python` 의 `typing` 모듈에서 제공하는 기능으로, 
딕셔너리의 키와 값을 명확하게 지정하는 데 사용된다.
이렇게 상태의 구조와 타입을 명확하게 정의하여, 각 `Node` 에서 주고받는 데이터의 일관성을 보장한다. 

`TypedDict` 는 `dict` 와 유사하지만 다음과 같은 차이점이 있다. 

- 타입 검사 : 정적 타입 검사를 제공한다. 
- 키와 값 타입 : 각 키에 대해 구체적인 타입을 지정할 수 있다. 
- 유연성 : 정의된 구조를 따라야 한다. 정의되지 않는 키는 오류가 발생한다. 

아래는 `TypedDict` 의 예시이다. 

```python
from typing import Dict, TypedDict

class ExampleTypedDict(TypedDict):
    name : str
    age : int


example_typed_dict : ExampleTypedDict = {
    "name" : "Jack",
    "age" : 30,
}


```  

`TypedDict` 을 그냥 사용하면 일반적인 `Dict` 와 큰 차이가 없다. 
그 이유는 `TypedDict` 는 정적 타입 힌트를 제공하는 도구일 뿐,
런타임에서 타입을 실제로 검사하거나 강제하지는 않기 때문이다. 
정적 타입검사를 사용하고 싶은 경우 `mypy` 혹은 `pyright` 와 같은 도구를 사용하면 타입 오류를 탐지해 낼 수 있다.  


### Annotated
`Annotated` 는 `Python` 의 `typing` 모듈에서 제공하는 기능으로, 
타입에 추가적인 설명(`Metadata`) 을 덧붙일 수 있다. 
`LangGraph` 에서는 상태 필드에 설명, 예시, 목적 등을 추가하여 코드의 가독성과 문서화를 돕는다. 
상태 필드에 대한 설명과 예시를 명확히 해 유지보수에 용이하게 한다. 
그리고 자동 문서화, 입력 검증 등에도 활용할 수 있다.  

또한 타입 힌트에 추가적인 정보를 추가해 추가적인 라이브러리(`Pydantic` 등)와 함께 사용해 데이터 유효성 검사를 수행할 수 있다.  


아래는 `Annotated` 의 예시이다. 

```python
from typing import Annotated, List
from pydantic import Field, BaseModel, ValidationError

# 기본 사용
name: Annotated[str, "사용자의 이름"]
age: Annotated[int, "사용자의 나이"]
skills: Annotated[List[str], "사용자의 기술"]



# Pydandic 과 사용
class Person(BaseModel):
  name: Annotated[str, Field(..., description="사용자의 이름")]
  age: Annotated[int, Field(gt=0, lt=150, description="사용자의 나이")]
  skills: Annotated[List[str], Field(min_items=1, max_items=3, description="사용자의 기술")]


valid_example = Person(name="jack", age=30, skills=["Java", "C", "LangChain"])
# Person(name='jack', age=30, skills=['Java', 'C', 'LangChain'])



try:
    invalid_example = Person(name="jack", age=300, skills=[])
except Error as e:
    print(e)
# ValidationError: 2 validation errors for Person
# age
# Input should be less than 150 [type=less_than, input_value=300, input_type=int]
# For further information visit https://errors.pydantic.dev/2.11/v/less_than
# skills
# List should have at least 1 item after validation, not 0 [type=too_short, input_value=[], input_type=list]
# For further information visit https://errors.pydantic.dev/2.11/v/too_short
# 
# During handling of the above exception, another exception occurred:
```  


### add_messages
`add_messages` 는 `LangGraph` 에서 대화형 워크플로우를 만들 때, 상태의 `messages` 리스트에 새로운 메시지를 추가하는 함수이다. 
`LLM` 응답이나 사용자 입력을 메시지 기록에 누적할 때 사용한다. 
대화 흐름을 관리하여, 다음 `LLM` 호출 시 `context` 로 활용할 수 있도록 한다. 
그리고 각 노드에서 메시지를 추가해 전체 대화 흐름을 기록하는데 사용할 수 있다.  

`add_messages` 함수는 2개의 인자를 받아 좌, 우 메시지를 병학하는 방식으로 동작한다. 
기본적으로 `append-only` 상태를 유지하고, 동일한 `ID` 를 가진 메시지가 있는 경우, 새 메시지로 기존 메시지를 대체한다.  

아래는 `add_messages` 의 예시이다. 

```python
from langchain_core.messages import AIMessage, HumanMessage
from langgraph.graph import add_messages

msg_1 = [HumanMessage(content="1+1 은?", id=1)]
msg_2 = [AIMessage(content="2", id=2)]

result_1 = add_messages(msg_1, msg_2)
# [HumanMessage(content='1+1 은?', additional_kwargs={}, response_metadata={}, id='1'),
#  AIMessage(content='2', additional_kwargs={}, response_metadata={}, id='2')]


# id 가 동일한 경우, 새 메시지로 기존 메시지를 대체한다.
msg_1 = [HumanMessage(content="1+1 은?", id=1)]
msg_2 = [AIMessage(content="2", id=1)]

result_2 = add_messages(msg_1, msg_2)
# [AIMessage(content='2', additional_kwargs={}, response_metadata={}, id='1')]
```  

`TypedDict`, `Annotated` 와 함께 `add_messages` 를 사용하면 아래와 같다.  

```python
from langchain_core.messages import AIMessage, HumanMessage
from langgraph.graph import add_messages
from typing import Annotated, TypedDict

class MyMessages(TypedDict):
  messages: Annotated[list, add_messages]


my_msg = MyMessages(messages=[HumanMessage(content="1+1 은?", id=1)])

my_msg["messages"].append(AIMessage(content="2", id=2))
# {'messages': [HumanMessage(content='1+1 은?', additional_kwargs={}, response_metadata={}, id='1'),
#               AIMessage(content='2', additional_kwargs={}, response_metadata={}, id='2')]}
```  


### LangGraph Chatbot Example
앞서 알아본 기본 개념들을 바탕으로 간단한 `Chatbot` 을 구현해 본다. 

#### StateGraph
`StateGraph` 객체는 챗봇의 구조를 `State Machine` 으로 정의한다. 
이를 통해 `nodes` 를 추가하여 챗봇이 호출할 수 있는 `LLM` 과 함수들을 나타내고, 
`edges` 를 추가해 봇이 이러한 함수들 간에 어떻게 전환해야 하는지 정의한다.  

먼저 아래와 같이 상태값으로 사용할 `ChatBotState` 를 정의하고 `StateGraph` 를 생성한다.  

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START
from langgraph.graph.message import add_messages

# 상태 정의
class ChatBotState(TypedDict):
    messages: Annotated[list, add_messages]

# 그래프 생성
graph_builder = StateGraph(State)
```  

### Nodes
다음으로 `Node` 를 추가한다. 
`Nodes` 는 작업 단위를 나타내며 일반적으로 `Python` 함수를 사용할 수 있다. 
챗봇에서 필요한 작업은 `LLM` 모델에 정의한 상태를 전달해 그 결과를 받는 것이므로 아래와 같이 정의한다.  

```python
from langchain_google_genai import ChatGoogleGenerativeAI

import os

# llm 정의
os.environ["GOOGLE_API_KEY"] = "api key"
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")


# 챗봇 함수 정의
def chatbot(state: ChatBotState):
    return {"messages": [llm.invoke(state["messages"])]}


# 함수 or callable 을 사용해 챗봇 노드 추가
graph_builder.add_node("chatbot", chatbot)
```  

### Edges
`Edges` 는 노드 간의 연결을 나타내며, 
각 노드가 어떤 조건에서 다음 노드로 전환되는지를 정의한다. 
`START` 는 그래프가 실행될 때마다 직업을 시작할 위치이다. 
`END` 는 그래프 흐름으 종료 지점을 나타낸다.  

```python
# 그래프 엣지 추가
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)
```  


### Graph Compile
`StateGraph` 정의 및 `Node`, `Edge` 추가가 완료되면 
`compile` 메서드를 호출해 그래프를 컴파일한다. 
이를 통해 상태를 바탕으로 호출할 수 있는 `CompiledGraph` 객체가 생성된다.  

```python
# 그래프 컴파일
graph = graph_builder.compile()
```   


### Graph Visualization
다음과 같이 컴파일한 `Graph` 객체를 사용해서 그래프를 시각화 할 수 있다.  

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/basics-1.png)


### Run Graph(Chatbot)
최종적으로 아래와 같이 구성한 그래프를 실행해 챗봇을 구현할 수 있다.  

```python
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)


while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break
        stream_graph_updates(user_input)
    except:
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
        
# User: 대한민국 광역시도 기준 인구 TOP 10을 지역명과 인구로 나열해줘
# Assistant: ## 대한민국 광역시도별 인구 TOP 10 (2024년 5월 기준)
# 
# 아래는 2024년 5월 행정안전부 주민등록 인구통계 자료에 기반한 광역시도별 인구 순위입니다.
# 
# | 순위 | 지역명    | 인구수 (명) |
# | --- | -------- | -------- |
# | 1   | 경기도    | 13,647,547  |
# | 2   | 서울특별시 | 9,407,540   |
# | 3   | 부산광역시 | 3,283,684   |
# | 4   | 경상남도   | 3,254,383   |
# | 5   | 인천광역시 | 2,999,323   |
# | 6   | 경상북도   | 2,607,932   |
# | 7   | 대구광역시 | 2,360,582   |
# | 8   | 충청남도   | 2,134,722   |
# | 9   | 전라북도   | 1,759,587   |
# | 10  | 전라남도   | 1,691,576   |
# 
# **참고:**
# 
# *   인구수는 주민등록 인구 기준으로, 실제 거주 인구와는 차이가 있을 수 있습니다.
# *   최신 자료는 행정안전부 주민등록 인구통계 웹사이트에서 확인하실 수 있습니다.
# User: langgraph 에 대해 짧게 소개해줘
# Assistant: LangGraph는 LangChain에서 제공하는 도구로, **LLM (Large Language Model)을 활용한 복잡한 대화형 애플리케이션을 구축하기 위한 프레임워크**입니다. 간단히 말해, LLM을 마치 레고 블록처럼 연결하여 **상태 관리, 순환 흐름, 조건부 분기** 등을 구현할 수 있게 해줍니다.
# 
# **핵심 특징:**
# 
# *   **그래프 기반 구조:** LLM, 함수, 프롬프트 등을 노드로 연결하고, 노드 간의 흐름을 정의하여 복잡한 대화 흐름을 시각적으로 표현하고 관리할 수 있습니다.
# *   **상태 관리:** 대화의 상태를 추적하고 업데이트하여 이전 대화 내용을 기억하고 활용할 수 있습니다.
# *   **순환 흐름:** 특정 조건을 만족할 때까지 노드를 반복적으로 실행하는 루프를 구현할 수 있습니다.
# *   **조건부 분기:** 대화의 흐름을 조건에 따라 분기하여 다양한 시나리오에 대응할 수 있습니다.
# *   **유연성 및 확장성:** 다양한 종류의 LLM, 도구, 프롬프트를 통합하여 사용자 정의 에이전트를 구축할 수 있습니다.
# 
# **LangGraph를 사용하면 다음과 같은 유형의 애플리케이션을 만들 수 있습니다:**
# 
# *   **자율 에이전트:** 목표를 달성하기 위해 스스로 계획을 세우고 실행하는 에이전트
# *   **챗봇:** 복잡한 질문에 답변하고 다양한 작업을 수행하는 챗봇
# *   **데이터 분석 도구:** LLM을 사용하여 데이터를 분석하고 시각화하는 도구
# *   **게임 에이전트:** 게임 환경에서 전략적으로 행동하는 에이전트
# 
# **LangGraph는 복잡한 대화형 애플리케이션을 구축하는 데 강력한 도구이지만, 초기 학습 곡선이 있을 수 있습니다. LangChain에 대한 기본적인 이해가 필요하며, 그래프 기반 프로그래밍에 대한 경험이 있으면 더욱 도움이 됩니다.**
# 
# 더 자세한 내용은 LangChain 공식 문서 ([https://python.langchain.com/docs/langgraph](https://python.langchain.com/docs/langgraph))를 참고하시기 바랍니다.
# User: q
# Goodbye!
```  


### Full code

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_google_genai import ChatGoogleGenerativeAI
from IPython.display import Image, display
import os


# 상태 정의 
class ChatBotState(TypedDict):
  messages: Annotated[list, add_messages]


# llm 정의
os.environ["GOOGLE_API_KEY"] = "api key"
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")


# 챗봇 함수 정의
def chatbot(state: ChatBotState):
  return {"messages": [llm.invoke(state["messages"])]}


# 그래프 생성
graph_builder = StateGraph(ChatBotState)


# 함수 or callable 을 사용해 챗봇 노드 추가
graph_builder.add_node("chatbot", chatbot)

# 그래프 엣지 추가
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)


# 그래프 컴파일
graph = graph_builder.compile()

# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass


# 그래프 실행
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)


while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break
        stream_graph_updates(user_input)
    except:
        # fallback if input() is not available
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```





---  
## Reference
[Build a basic chatbot](https://langchain-ai.github.io/langgraph/tutorials/get-started/1-build-basic-chatbot/)  
[Python typing](https://docs.python.org/3/library/typing.html)  
[LangGraph 에 자주 등장하는 Python 문법이해](https://wikidocs.net/264613)  
[Graph API concepts](https://langchain-ai.github.io/langgraph/concepts/low_level/)  
[LangGraph Reference](https://langchain-ai.github.io/langgraph/reference/)  
[LangGraph Tutorial](https://github.com/langchain-ai/langgraph/tree/main/docs/docs/tutorials)  
[LangGraph Docs](https://github.com/langchain-ai/langgraph/tree/main/docs/docs)  


