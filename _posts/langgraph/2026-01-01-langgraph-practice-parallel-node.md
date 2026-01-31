--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Parallel Node"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LangGraph 에서 노드를 동시에 실행할 수 있는 Parallel Node 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - Parallel Node
    - LLM
    - Python
toc: true
use_math: true
---  

## LangGraph Parallel Node
그래프 상에서 두 개 이상의 노드를 동시에(병렬로) 실행하여, 
전체 처리 속도를 높이거나 다양한 경로로 데이터를 처리할 수 있도록 할 수 있다. 
일반적으로 워크플로우는 직렬(순차적)로 노드가 실행되지만, 
서로 의존성이 없는 작업은 병렬 실행이 효율적일 수 있다. 
`LangGraph` 에서는 한 노드에 대한 분기(`edge`)를 여러 노드로 연결하고, 
여러 노드로 분기된 `edge` 가 다시 하나의 노드로 합쳐지는 구조로 병렬 처리를 구현할 수 있다.  

`LangGraph` 에서는 병렬 처리를 위해 `fan-out` 과 `fan-in` 이라는 구조와 개념을 사용한다. 
이는 복잡한 작업을 효율적으로 처리할 때 사용하는 전형적인 병렬 처리 패턴이다.  

- `fan-out`(확장)
  - 큰 작업을 여러 개의 더 작은, 독립적인 작업으로 나누어 동시에 처리하는 과정이다. 
  - 이는 마치 요리를 만들 때 필요한 각각의 제료 손질을 병렬로 진행하는 것과 비슷하다.  
- `fan-in`(수집)
  - 여러 경로로 나눠졌던 작업(또는 데이터)이 모두 끝나면, 이 결과를 다시 하나로 모으는 과정이다.  
  - 각각 준비한 재료를 한데 모아 요리를 완성하는 것과 같다.

병렬 처리된 결과를 다시 합칠 때 단순히 값을 덮어쓰는 것이 아니라,
여러 결과를 누적하거나 결합해야 하는 경우가 있다. 
이런 경우 그래프 상태에 `reducer` 의 `add` 연산자인 `add_message` 를 사용해, 
여러 노드의 결과가 리스트라면, 각 리스트를 이어붙이는 식으로 집계할 수 있다.  

`LangGraph` 에서는 타입 자체를 바꾸지 않고, 
타입에 `reducer` 함수를 `주석`처럼 첨부할 수 있도록 `Annotated` 를 사용한다. 
리스트 타입에 `add reducer`(`add_message`) 를 주석으로 달면, 해당 카에 값을 추가할 때마다 리스트를 자동으로 이어붙이게 동작한다. 
이렇게 하면 병렬 브랜치에서 온 여러 결과를 자연스럽게 하나로 모을 수 있다.  

예제로 진행할 예시 구조는 다음과 같다. 

- `fan-out` : `begin` 노드에서 `parallel-1`, `parallel-2` 노드로 분기
- `fan-in` : `parallel-1`, `parallel-2` 노드에서 `agg` 노드로 결합

```python
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from IPython.display import Image, display

class State(TypedDict):
  aggregate: Annotated[list, add_messages]

class SimpleNode:
  def __init__(self, node_secret: str):
    self._value = node_secret

  # 호출시 상태 업데이트
  def __call__(self, state: State) -> Any:
    print(f"Adding {self._value} to {state['aggregate']}")
    return {"aggregate": [self._value]}


graph_builder = StateGraph(State)

# 노드 begin 부터 parallel_1, parallel_2, agg 노드 생성 및 할당
graph_builder.add_node("begin", SimpleNode("I am the begin node"))
graph_builder.add_node("parallel_1", SimpleNode("I am the parallel_1 node"))
graph_builder.add_node("parallel_2", SimpleNode("I am the parallel_2 node"))
graph_builder.add_node("agg", SimpleNode("I am the agg node"))

# 노드 연결
graph_builder.add_edge(START, "begin")
graph_builder.add_edge("begin", "parallel_1")
graph_builder.add_edge("begin", "parallel_2")
graph_builder.add_edge("parallel_1", "agg")
graph_builder.add_edge("parallel_2", "agg")
graph_builder.add_edge("agg", END)

# 그래프 컴파일
agent = graph_builder.compile()

# 그래프 시각화
try:
    display(Image(agent.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/parallel-node-1.png)


그래프를 실행하면 `reducer` 를 통해 각 노드에 추가된 값들이 차례대로 누적되는 것을 확인할 수 있다.  

```python
# 그래프 실행
agent.invoke({"aggregate": []}, {"configurable" : {"thread_id": "1"}})
# Adding I am the begin node to []
# Adding I am the parallel_1 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='abc8b55f-0682-46b8-af9f-19fdefe326bf')]
# Adding I am the parallel_2 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='abc8b55f-0682-46b8-af9f-19fdefe326bf')]
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='abc8b55f-0682-46b8-af9f-19fdefe326bf'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a118a146-2d64-405c-aa15-ace135c13f3d'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='9020c811-e1e2-4cf5-a4ff-c0f6187846f7')]
# {'aggregate': [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='abc8b55f-0682-46b8-af9f-19fdefe326bf'),
#                HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a118a146-2d64-405c-aa15-ace135c13f3d'),
#                HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='9020c811-e1e2-4cf5-a4ff-c0f6187846f7'),
#                HumanMessage(content='I am the agg node', additional_kwargs={}, response_metadata={}, id='5ad5f7eb-ee1e-4f2d-8179-102747fe27bf')]}
```  

### Error when parallel
`LangGraph` 는 여러 노드가 동시에 실행되는 `super-step` 구조를 사용한다. 
`super-step` 이란 병렬로 분기된 여러 노드가 한 번에 처리되는 하나의 프로세스 단계를 의미한다. 
이는 `LangGraph` 의 실행 모델에서 사용되는 용어로, 그래프 내 여러 노드가 동시에 병렬로 실행되는 하나의 논리적 치리 단계를 의미한다. 
한 번에 여러 노드가 병렬로 실행될 수 있는데, 이들 병렬 실행의 `한 묶음`이 바로 하나의 `super-step` 인 것이다. 
트랜잭션적 처리로 `super-step` 내에서 실행되는 모든 노드의 작업이 모두 성공해야 해당 단계의 상태 업데이트가 이뤄진다. 
병렬 분기 중 하나라도 오류가 발생하면 전체 `super-step` 이 롤백 되어, 이전 상태로 유지된다.  

`API` 호출이나 `LLM`, 데이터베이스 등 외부 리소스와의 상호작용이 포함된 노드에서 오류가 발생할 수 있다. 
`LangGrpah` 는 이런 오류를 처리하기 위해 아래 2가지 방식으로 예외 처리를 지원한다. 

1. 노드 내부에서 직접 예외 처리(`Python` 코드 사용)
  - 각 노드 함수 내에서 `try-except` 블록을 사용해 예외를 처리할 수 있다. 
  - 필요하다면 오류 메시지나 기본값을 반환하여 워크플로우가 계속 진행되도록 할 수 있다. 
2. [retry_policy](https://langchain-ai.github.io/langgraph/reference/types/#langgraph.types.RetryPolicy) 사용
  - 노드 추가 시, `retry_policy` 를 지정해 특정 예외 발생 시 해당 노드만 자동으로 재시도할 수 있다. 
  - 이때 실패한 분기만 재실행되머, 이미 성공한 다른 분기는 불필요하게 다시 실행되지 않는다.  

예제에서는 위 2가지 방법 중 `retry_policy` 를 사용해 예외를 처리하는 방법에 대해 알아본다.  

```python
# 병렬처리 중 예외 발생시 대응

from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.types import RetryPolicy
import random

thread_id = random.randrange(1, 99999999999)
class State(TypedDict):
  aggregate: Annotated[list, add_messages]

class SimpleNode:
  def __init__(self, node_secret: str):
    self._value = node_secret

  # 호출시 상태 업데이트
  def __call__(self, state: State) -> Any:
    print(f"Adding {self._value} to {state['aggregate']}")

    if random.randrange(1, 10) % 2 == 0:
      print(f"Error {self._value}")
      raise Exception("예외 발생")

    return {"aggregate": [self._value]}


graph_builder = StateGraph(State)

# 노드 begin 부터 parallel_1, parallel_2, agg 노드 생성 및 할당
graph_builder.add_node("begin", SimpleNode("I am the begin node"))
graph_builder.add_node("parallel_1", SimpleNode("I am the parallel_1 node"), retry_policy=RetryPolicy(max_attempts=5))
graph_builder.add_node("parallel_2", SimpleNode("I am the parallel_2 node"), retry_policy=RetryPolicy(max_attempts=5))
graph_builder.add_node("agg", SimpleNode("I am the agg node"), retry_policy=RetryPolicy(max_attempts=5))

# 노드 연결
graph_builder.add_edge(START, "begin")
graph_builder.add_edge("begin", "parallel_1")
graph_builder.add_edge("begin", "parallel_2")
graph_builder.add_edge("parallel_1", "agg")
graph_builder.add_edge("parallel_2", "agg")
graph_builder.add_edge("agg", END)

# 그래프 컴파일
agent = graph_builder.compile()
```  

특정 확률로 예외가 발생하도록 하고 그래프를 실행하면 예외가 발생한 노드에서만 재시도가 이뤄지는 것을 확인할 수 있다.  

```python
# 예외 처리 그래프 실행
agent.invoke({"aggregate": []}, {"configurable" : {"thread_id": "1"}})
# Adding I am the begin node to []
# Adding I am the parallel_1 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068')]
# Adding I am the parallel_2 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068')]
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a781af5a-d01f-438a-9d67-20053ee2efd1'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='bcc5ed9d-6364-419d-b822-a1ec327fed44')]
# Error I am the agg node
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a781af5a-d01f-438a-9d67-20053ee2efd1'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='bcc5ed9d-6364-419d-b822-a1ec327fed44')]
# Error I am the agg node
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a781af5a-d01f-438a-9d67-20053ee2efd1'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='bcc5ed9d-6364-419d-b822-a1ec327fed44')]
# Error I am the agg node
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a781af5a-d01f-438a-9d67-20053ee2efd1'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='bcc5ed9d-6364-419d-b822-a1ec327fed44')]
# Error I am the agg node
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a781af5a-d01f-438a-9d67-20053ee2efd1'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='bcc5ed9d-6364-419d-b822-a1ec327fed44')]
# {'aggregate': [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='cc44fa8a-fb8e-4211-a2b9-76310d814068'),
#                HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a781af5a-d01f-438a-9d67-20053ee2efd1'),
#                HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='bcc5ed9d-6364-419d-b822-a1ec327fed44'),
#                HumanMessage(content='I am the agg node', additional_kwargs={}, response_metadata={}, id='19884d28-b3e4-4157-994f-aa7b624ae896')]}
```  

### Parallel Fan-out/Fan-in with Intermediate Steps
`fan-out`, `fan-in` 구조를 사용해 병렬 처리를 구현할 때, 
각 중간에 여러 단계가 있는 경우 구조와 구현에 예시에 대해 알아본다.  

```python
# 추가 단계가 있는 병렬 노드의 fan-out 과 fan-in

from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.types import RetryPolicy
import random
from IPython.display import Image, display

thread_id = random.randrange(1, 99999999999)
class State(TypedDict):
  aggregate: Annotated[list, add_messages]

class SimpleNode:
  def __init__(self, node_secret: str):
    self._value = node_secret

  # 호출시 상태 업데이트
  def __call__(self, state: State) -> Any:
    print(f"Adding {self._value} to {state['aggregate']}")

    return {"aggregate": [self._value]}


graph_builder = StateGraph(State)

# 노드 begin 부터 parallel_1, parallel_2, agg 노드 생성 및 할당
graph_builder.add_node("begin", SimpleNode("I am the begin node"))
graph_builder.add_node("parallel_1", SimpleNode("I am the parallel_1 node"))
graph_builder.add_node("parallel_1_a", SimpleNode("I am the parallel_1_a node"))
graph_builder.add_node("parallel_2", SimpleNode("I am the parallel_2 node"))
graph_builder.add_node("agg", SimpleNode("I am the agg node"))

# 노드 연결
graph_builder.add_edge(START, "begin")
graph_builder.add_edge("begin", "parallel_1")
graph_builder.add_edge("begin", "parallel_2")
graph_builder.add_edge("parallel_1", "parallel_1_a")
# 2개 노드를 묶어서 엣지 추가
graph_builder.add_edge(["parallel_1_a", "parallel_2"], "agg")
graph_builder.add_edge("agg", END)

# 그래프 컴파일
agent = graph_builder.compile()

# 그래프 시각화
try:
    display(Image(agent.get_graph().draw_mermaid_png()))
except Exception:
    pass
```

![그림 1]({{site.baseurl}}/img/langgraph/parallel-node-2.png)


구현된 그래프를 실행하면 아래와 같이 순차적으로 실행된 결과가 나온다. 


```python
# Adding I am the begin node to []
# Adding I am the parallel_1 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='a7aba457-4042-4c36-9cfa-94ccc56c4dfa')]
# Adding I am the parallel_2 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='a7aba457-4042-4c36-9cfa-94ccc56c4dfa')]
# Adding I am the parallel_1_a node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='a7aba457-4042-4c36-9cfa-94ccc56c4dfa'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='574623bf-55da-4e66-880f-06dbc8b68f04'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='f6b0aec9-9426-4ca1-858e-651272704500')]
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='a7aba457-4042-4c36-9cfa-94ccc56c4dfa'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='574623bf-55da-4e66-880f-06dbc8b68f04'), HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='f6b0aec9-9426-4ca1-858e-651272704500'), HumanMessage(content='I am the parallel_1_a node', additional_kwargs={}, response_metadata={}, id='c3747f8a-e0e8-448f-b7f3-d8a3b315927f')]
# {'aggregate': [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='a7aba457-4042-4c36-9cfa-94ccc56c4dfa'),
#   HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='574623bf-55da-4e66-880f-06dbc8b68f04'),
#   HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='f6b0aec9-9426-4ca1-858e-651272704500'),
#   HumanMessage(content='I am the parallel_1_a node', additional_kwargs={}, response_metadata={}, id='c3747f8a-e0e8-448f-b7f3-d8a3b315927f'),
#   HumanMessage(content='I am the agg node', additional_kwargs={}, response_metadata={}, id='6ca503e9-f7b5-46a2-966d-df74d574d4f6')]}
```  

### Parallel with Conditional Branching
병렬로 여러 노드를 실행할 때, 
매번 모든 노드를 병렬로 실행하는 것이 아니라 조건에 따라 노드 조합을 병렬로 실행하도록 구성할 수 있다.  

이는 `add_conditional_edges` 를 사용해 분기 로직 등 자세한 내용을 직접 구현해 추가하는 방식이다. 
예제에서는 상태에 `which` 라는 필드를 추가하고 그래프 실행시 `which` 에 `parallel_1,parallel_2` 와 같이 `,` 로 구분하여 병렬로 실행할 노드 이름을 전달한다.  

```python
# 조건부 분기(conditional branching)

from typing import Annotated, Any, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from IPython.display import Image, display

class State(TypedDict):
  which: str
  aggregate: Annotated[list, add_messages]

class SimpleNode:
  def __init__(self, node_secret: str):
    self._value = node_secret

  # 호출시 상태 업데이트
  def __call__(self, state: State) -> Any:
    print(f"Adding {self._value} to {state['aggregate']}")
    return {"aggregate": [self._value]}


graph_builder = StateGraph(State)

# 노드 begin 부터 parallel_1, parallel_2, agg 노드 생성 및 할당
graph_builder.add_node("begin", SimpleNode("I am the begin node"))
graph_builder.add_node("parallel_1", SimpleNode("I am the parallel_1 node"))
graph_builder.add_node("parallel_2", SimpleNode("I am the parallel_2 node"))
graph_builder.add_node("parallel_3", SimpleNode("I am the parallel_3 node"))
graph_builder.add_node("agg", SimpleNode("I am the agg node"))

# 상태의 which` 값에 따른 조건부 라우팅 경로 결정 함수
def route(state: State) -> Sequence[str]:
  return [item.strip() for item in state["which"].split(',')]


# 노드 연결
graph_builder.add_edge(START, "begin")
parallel_nodes = ["parallel_1", "parallel_2", "parallel_3"]

graph_builder.add_conditional_edges(
    "begin",
    route,
    parallel_nodes
)

for node in parallel_nodes:
  graph_builder.add_edge(node, "agg")

graph_builder.add_edge("agg", END)

# 그래프 컴파일
agent = graph_builder.compile()

# 그래프 시각화
try:
    display(Image(agent.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/parallel-node-3.png)


구성된 그래프를 실행하면 3개의 병렬 노드 중 `which` 에 전달한 노드들만 병렬로 수행되는 것을 확인할 수 있다.  

```python
# 그래프 실행
agent.invoke({"aggregate": [], "which" : "parallel_1,parallel_3"}, {"configurable" : {"thread_id": "1"}})
# Adding I am the begin node to []
# Adding I am the parallel_1 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='f8e56bd4-75b2-4c8e-a899-6e26b5e6b7cf')]
# Adding I am the parallel_3 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='f8e56bd4-75b2-4c8e-a899-6e26b5e6b7cf')]
# Adding I am the agg node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='f8e56bd4-75b2-4c8e-a899-6e26b5e6b7cf'), HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a24a58b7-da2c-44dd-a846-f3674a29f0c3'), HumanMessage(content='I am the parallel_3 node', additional_kwargs={}, response_metadata={}, id='b8139be4-e875-4c76-b218-d125942b2e0f')]
# {'which': 'parallel_1,parallel_3',
#  'aggregate': [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='f8e56bd4-75b2-4c8e-a899-6e26b5e6b7cf'),
#                HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='a24a58b7-da2c-44dd-a846-f3674a29f0c3'),
#                HumanMessage(content='I am the parallel_3 node', additional_kwargs={}, response_metadata={}, id='b8139be4-e875-4c76-b218-d125942b2e0f'),
#                HumanMessage(content='I am the agg node', additional_kwargs={}, response_metadata={}, id='8d179217-790e-4ed5-99cc-eb85471f6528')]}
```  

### Confidence-based Fan-out Ranking
앞서 설명한 것과 같이, 병롤 노드들은 하나의 `super-step` 으로 실행하고, 완료된 다음에 순차적으로 상태에 적용된다.
만약 병렬 `super-step` 에서 특정 기준으로 업데이트 순서가 필요한 경우, 
별도의 집계 로직을 만들어 `edge` 로 등록해 사용할 수 있다.  

아래 예제는 `reliability` 라는 필드를 우선순위 값으로 사용해 병렬 노드의 결과를 병합하는 예시이다.  

```python
# fan-out 값의 신뢰도에 따른 정렬

from typing import Annotated, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages
from IPython.display import Image, display

# 팬아웃 값들의 병합 로직 구현, 빈 리스트 처리 및 리스트 연결 수행
def reduce_fanouts(left, right):
  if left is None:
    left = []

  if not right:
    return []

  return left + right

# 상태 관리를 위한 타입 정의, 집계 및 팬아웃 값 저장 구조 설정
class State(TypedDict):
  aggregate: Annotated[list, add_messages]
  fanout_values: Annotated[list, reduce_fanouts]
  which: str


# 상태의 which` 값에 따른 조건부 라우팅 경로 결정 함수
def route(state: State) -> Sequence[str]:
  return [item.strip() for item in state["which"].split(',')]


class SimpleNode:
  def __init__(self, node_secret: str):
    self._value = node_secret

  # 호출시 상태 업데이트
  def __call__(self, state: State) -> Any:
    print(f"Adding {self._value} to {state['aggregate']}")
    return {"aggregate": [self._value]}

# reliability 값을 가진 병렬 노드 클래스 정의
class ParallelNode:
  def __init__(
      self,
      node_secret: str,
      reliability: float
  ):
    self._value = node_secret
    self._reliability = reliability

  def __call__(self, state: State) -> Any:
    print(f"Adding {self._value} to {state['aggregate']} in parallel.")

    return {
        "fanout_values": [
            {
                "value" : [self._value],
                "reliability" : self._reliability
            }
        ]
    }


# 팬아웃 값들을 신뢰도 기준으로 정렬 및 집계
def agg_fanout_values(state: State) -> Any:
  ranked_values = sorted(
      state["fanout_values"], key=lambda x: x["reliability"], reverse=True
  )

  print(f"Ranked values: {ranked_values}")

  return {
      "aggregate" : [x["value"][0] for x in ranked_values] + ["I am the agg node"],
      "fanout_values": []
  }

graph_builder = StateGraph(State)
graph_builder.add_node("begin", SimpleNode("I am the begin node"))
graph_builder.add_node("parallel_1", ParallelNode("I am the parallel_1 node", reliability=0.5))
graph_builder.add_node("parallel_2", ParallelNode("I am the parallel_2 node", reliability=0.9))
graph_builder.add_node("parallel_3", ParallelNode("I am the parallel_3 node", reliability=0.7))

# 집계 노드 추가
graph_builder.add_node("agg", agg_fanout_values)

# 병렬 노드 조건부 분기 추가
parallel_nodes = ["parallel_1", "parallel_2", "parallel_3"]
graph_builder.add_conditional_edges(
    "begin",
    route,
    parallel_nodes
)


# 엣지 추가
graph_builder.add_edge(START, "begin")
for node in parallel_nodes:
  graph_builder.add_edge(node, "agg")
graph_builder.add_edge("agg", END)

# 그래프 컴파일
agent = graph_builder.compile()

# 그래프 시각화
try:
    display(Image(agent.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/parallel-node-4.png)


각 병렬 노드의 `reliability` 우선순위는 아래와 같다.  

- `parallel_1` : 0.5
- `parallel_2` : 0.9
- `parallel_3` : 0.7

`parallel_1` 과 `parallel_3` 를 실행하면 아래와 같이 우선순위가 높은 `parallel_3` 의 결과가 먼저 위치한다.  

```python
# 그래프 실행
agent.invoke({"aggregate": [], "which" : "parallel_1,parallel_3", "fanout_values": []}, {"configurable" : {"thread_id": "1"}})
# Adding I am the begin node to []
# Adding I am the parallel_1 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='5fbfcee3-0547-4dd5-8bea-fef2481938e8')] in parallel.
# Adding I am the parallel_3 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='5fbfcee3-0547-4dd5-8bea-fef2481938e8')] in parallel.
# Ranked values: [{'value': ['I am the parallel_3 node'], 'reliability': 0.7}, {'value': ['I am the parallel_1 node'], 'reliability': 0.5}]
# {'aggregate': [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='5fbfcee3-0547-4dd5-8bea-fef2481938e8'),
#                HumanMessage(content='I am the parallel_3 node', additional_kwargs={}, response_metadata={}, id='a79781c6-fc2d-4d33-b9ac-1012953644d4'),
#                HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='495161a2-04a0-4f83-af37-a6c913bd7aed'),
#                HumanMessage(content='I am the agg node', additional_kwargs={}, response_metadata={}, id='8ff22a7e-993c-4892-af73-2d204be1ff65')],
#  'fanout_values': [],
#  'which': 'parallel_1,parallel_3'}
```  

모든 병렬 노드를 수행하면 `parallel_2`, `parallel_3`, `parallel_1` 순으로 최종 결과가 집계된 것을 확인할 수 있다.  

```python
# 그래프 실행
agent.invoke({"aggregate": [], "which" : "parallel_3,parallel_2,parallel_1", "fanout_values": []}, {"configurable" : {"thread_id": "2"}})
# Adding I am the begin node to []
# Adding I am the parallel_1 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='28f5555c-a789-497d-9d58-bbc7d0e32e3c')] in parallel.
# Adding I am the parallel_2 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='28f5555c-a789-497d-9d58-bbc7d0e32e3c')] in parallel.
# Adding I am the parallel_3 node to [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='5fbfcee3-0547-4dd5-8bea-fef2481938e8')] in parallel.
# Ranked values: [{'value': ['I am the parallel_2 node'], 'reliability': 0.9}, {'value': ['I am the parallel_3 node'], 'reliability': 0.7}, {'value': ['I am the parallel_1 node'], 'reliability': 0.5}]
# {'aggregate': [HumanMessage(content='I am the begin node', additional_kwargs={}, response_metadata={}, id='28f5555c-a789-497d-9d58-bbc7d0e32e3c'),
#                HumanMessage(content='I am the parallel_2 node', additional_kwargs={}, response_metadata={}, id='1a00a136-336b-4976-8caa-f32283800599'),
#                HumanMessage(content='I am the parallel_3 node', additional_kwargs={}, response_metadata={}, id='a79781c6-fc2d-4d33-b9ac-1012953644d4'),
#                HumanMessage(content='I am the parallel_1 node', additional_kwargs={}, response_metadata={}, id='49c4b278-6821-4be4-b7b8-aac2a822d9a5'),
#                HumanMessage(content='I am the agg node', additional_kwargs={}, response_metadata={}, id='599692d6-03e0-4526-912f-f6c8313a56d8')],
#  'fanout_values': [],
#  'which': 'parallel_3,parallel_2,parallel_1'}
```  



---  
## Reference
[How to create branches for parallel node execution](https://langchain-ai.github.io/langgraphjs/how-tos/branching/)  
[super-step](https://langchain-ai.github.io/langgraphjs/concepts/low_level/#graphs)  
[Add retry policies](https://langchain-ai.github.io/langgraph/how-tos/graph-api/?h=retry_policy#add-runtime-configuration)  
[Parallel Node](https://wikidocs.net/265766)  

