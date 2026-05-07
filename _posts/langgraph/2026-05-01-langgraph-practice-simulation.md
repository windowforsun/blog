--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Simulation"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LangGraph 를 바탕으로 Agent/Assistant 를 만들고 이를 반복 테스트하고 개선점을 파악하는 시뮬레이션에 대해 알아보자.'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - Simulation
toc: true
use_math: true
---  


## LangGraph Agent Simulation
구현된 `Agent/Assistant` 가 정상적으로 동작하는지, 적절한 답변을 제공하는지 품질에 대한 경가를 할 때, 
실제 사용자화의 대화를 일일이 수동으로 테스트하면 시가니 많이 소요되고 평가가 어렵다. 
용이한 테스트를 위해서는 `재현 가능성` 을 통해 동일한 시나리오와 조건에서 반복적으로 테스트할 수 있어야 개선점을 명확히 파악할 수 있기 때문이다. 
그리고 `자동화` 를 통해 대화 시뮬레이션을 바탕으로 다양한 응답 케이스를 자동으로 생성/평가/수집할 수 있어야 한다.  

`LangGraph` 를 활용하면 이러한 자동화된 평가를 위한 시뮬레이션을 비교적 쉽게 구현할 수 있다. 
각 노드는 이에전트, 사용자 도구, 등 특정 역할을 담당하고, 그래프 형태의 설계로 흐름에 따라 대화가 진행 되도록 구성할 수 있기 때문이다.  

이러한 평가를 위한 시뮬레이션은 다음과 같은 활용을 할 수 있다.  

- 테스트 자동화 : 다양한 시나이로를 미리 정의해 자동으로 테스트하고 코드 변경 시마다 대화 시뮬레이션을 바탕으로 품질 확보와 반복적인 개선이 가능하다. 
- 성능 평가 : 응답의 정확도, 친절함, 문제 해결 능력 등을 자동으로 점수화 가능하다. 
- 반복 개선 : 테스트 결과를 바탕으로 에이전트의 답변 패턴, 행동 등을 지속적으로 개선할 수 있다. 
- 실 서비스 적용 전 검증 : 실제 서비스에 적용하기 전에 다양한 시나리오를 통해 에이전트의 동작을 사전 검증할 수 있다.  
- A/B 테스트 : 서로 다른 에이전트 버전을 시뮬레이션해 비교하고 최적의 버전을 선택할 수 있다. 
- 실시간 모니터링 : 실제 서비스 적용 이후에도 지소적으로 수집된 시나리오를 바탕으로 시뮬레이션을 돌려 품질 유지 및 이상 탐지를 할 수 있다. 

이러한 평가 시뮬레이션을 위해 `LangGraph` 를 활용해 간단한 `AI Assistant(상담사)` 를 구현하고, 
이를 가상 사용자를 통해 시뮬레이션하는 예제를 살펴본다.  


### Graph State
가장 먼저 구현할 그래프에서 사용할 상태를 정의한다.  

```python
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict


class AgentState(TypedDict):
  messages: Annotated[list, add_messages]
```  


### AI Assistant Chain
시뮬레이션 대상이 되는 `AI Assistant` 를 구현한다. 
`AI Assistant` 의 역할은 상담사로 소프트웨어 관련 질의에 전문적인 답변을 제공하는 기업 전문 상담사로 설정한다. 

```python
# 상담사 챗봇 노드 함수 정의

from typing import List
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder, SystemMessagePromptTemplate, HumanMessagePromptTemplate
from langchain_core.messages import BaseMessage, AIMessage, HumanMessage
from langchain_core.output_parsers import StrOutputParser


def call_chatbot(messages: List[BaseMessage]) -> dict:
  prompt = ChatPromptTemplate.from_messages(
      [
          SystemMessagePromptTemplate.from_template(template="""
          You are a software expert and a professional inquiry response worker dispatched to a company. 
          Your answer must be in Korean.
          """),
          MessagesPlaceholder(variable_name="messages")
      ]
  )

  llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")
  chain = prompt | llm | StrOutputParser()

  return chain.invoke({'messages' : messages})
```  

### Simulation User Chain
가상 사용자는 지시사항에 따라 상담사에게 질문하는 대화를 시뮬레이션 한다. 
지시사항은 외부에서 주입할 수 있도록 구성해 다양한 시나리오를 시뮬레이션 할 수 있도록 구성한다.  

```python
# 고객 역할 시뮬레이션 노드 함수 정의

from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder, SystemMessagePromptTemplate, HumanMessagePromptTemplate

def create_scenario(name: str, instructions: str):
  prompt = ChatPromptTemplate.from_messages(
      [
          SystemMessagePromptTemplate.from_template(template="""
          You are a developer of software Company.
          I'm interacting with a user who is a technical support representative.

          Your name is {name}.

          #Instruction:
          {instruction}

          [Important]
          - When the conversation is over, respond with one word: 'Completed'.
          - You must use Korean for your answer.
          """),
          MessagesPlaceholder(variable_name="messages")
      ]
  )

  prompt = prompt.partial(name=name, instruction=instructions)

  return prompt
```  

아래와 같이 지시사항을 설정해 사용할 수 있다.  

```python
# 지시사항을 추가하고 사용자 시뮬레이션 노드 생성

instructions = """
You are currently trying to introduce new technologies such as AI, LLM, LangChain, LangGraph into your new project.
However, there are many difficulties with the new concepts and technologies.
Start with the basics, ask more in-depth questions, and achieve breadth and depth of knowledge to a practical level.
"""

name = 'windowforsun'

create_scenario(name, instructions).pretty_print()
```  

구현된 가장 사용자와 지시사항을 바탕으로 테스트 시뮬레이션을 수행해 보면 아래와 같다.  

```python
from langchain_core.messages import HumanMessage

llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

simulated_user = create_scenario(name, instructions) | llm | StrOutputParser()

messages = [HumanMessage(content='안녕하세요? 무엇을 도와드릴까요?')]

simulated_user.invoke(messages)
# 안녕하세요. 저는 windowforsun입니다. 최근에 새로운 프로젝트에 AI, LLM, LangChain, LangGraph 같은 기술들을 도입하려고 하는데, 개념이 너무 생소해서 어려움을 겪고 있습니다. 혹시 기본적인 내용부터 차근차근 설명해주실 수 있을까요? 어떤 부분부터 시작하는 게 좋을지 조언을 구하고 싶습니다.
```  

### Simulation Node
시뮬레이션에 필요한 노드를 정의한다.  

가장 먼저 `AI Assistant` 노드를 정의한다. 
앞서 구현한 `AI Assistant Chain` 에 대화를 입력으로 넣으면 최종 답변을 `AIMessage` 로 응답하는 노드이다.  

```python
from langchain_core.messages import AIMessage

def ai_assistant_node(state):
  messages = state['messages']
  ai_response = call_chatbot(messages)

  return {'messages' : [AIMessage(content=ai_response)]}


ai_assistant_node(
    {
        'messages' : [('user', '안녕하세요?'),
                      ('assistant', '안녕하세요? 무엇을 도와드릴까요?'),
                      ('user', '신규 프로젝트에 AI, LLM 등 을 도입하려 하고 있어요.')]
    }
)
# {'messages': [AIMessage(content='아, 신규 프로젝트에 AI와 LLM(Large Language Model)을 도입하시려는군요. 아주 좋은 방향입니다. AI와 LLM은 현재 다양한 분야에서 혁신을 가져오고 있으며, 프로젝트의 효율성과 성능을 크게 향상시킬 수 있습니다.\n\n도입을 고려하시는 프로젝트의 종류, 목표, 그리고 현재 가지고 계신 리소스에 대해 좀 더 자세히 알려주시면, 더 구체적인 조언을 드릴 수 있습니다. 예를 들어 다음과 같은 질문에 답해주시면 좋습니다.\n\n*   **프로젝트의 목표는 무엇인가요?** (예: 고객 서비스 자동화, 콘텐츠 생성, 데이터 분석, 의사 결정 지원 등)\n*   **어떤 종류의 데이터를 다루게 되나요?** (예: 텍스트, 이미지, 음성, 숫자 데이터 등)\n*   **현재 어떤 기술 스택을 사용하고 계신가요?** (예: Python, TensorFlow, PyTorch, AWS, Azure, GCP 등)\n*   **AI 및 LLM 관련 경험이 있는 팀원이 있나요?**\n*   **예산은 어느 정도 예상하시나요?**\n\n이러한 정보를 바탕으로, 다음과 같은 측면에서 도움을 드릴 수 있습니다.\n\n*   **적합한 AI/LLM 기술 및 모델 추천:** 프로젝트 목표와 데이터에 맞는 최적의 기술과 모델을 선택할 수 있도록 도와드립니다. (예: GPT-3, BERT, Transformer, CNN, RNN 등)\n*   **데이터 전처리 및 모델 학습 전략:** AI/LLM 모델의 성능을 극대화하기 위한 데이터 전처리 방법과 학습 전략을 제시합니다.\n*   **클라우드 환경 구축 및 관리:** AI/LLM 모델을 효율적으로 운영하기 위한 클라우드 환경 구축 및 관리 방안을 안내합니다. (AWS, Azure, GCP 등)\n*   **API 연동 및 시스템 통합:** 기존 시스템과 AI/LLM 모델을 원활하게 연동하기 위한 API 설계 및 통합 전략을 제공합니다.\n*   **보안 및 윤리적 고려 사항:** AI/LLM 모델 사용 시 발생할 수 있는 보안 문제와 윤리적 문제에 대한 해결 방안을 제시합니다. (개인 정보 보호, 데이터 편향성 등)\n*   **팀 교육 및 기술 지원:** 팀원들의 AI/LLM 관련 역량 강화를 위한 교육 프로그램을 추천하고, 기술적인 문제 발생 시 지원을 제공합니다.\n\n어떤 부분부터 시작하고 싶으신가요? 아니면 특정적으로 궁금한 점이 있으신가요? 편하게 말씀해주세요.', additional_kwargs={}, response_metadata={})]}
```  

다음으로 `Simulation User` 노드를 정의한다.
가상 사용자 노드를 정의하기 전에 한가지 고려해야 할 점은 메시지의 형식이다. 
`AI Assistant`, `Simulation User` 모두 `LLM` 이기 때문에 모두 `AIMessage` 형식으로 응답한다. 
하지만 시뮬레이션에서 기대하는 것은 `HumanMessage` 와 `AIMessage` 가 번갈아 가며 오고가는 모양세일 것이다. 
그리고 `AI Assistant` 에게는 `Simulation User` 의 응답이 `HumanMessage` 이여야 하고,
`Simulation User` 에게는 `AI Assistant` 의 응답이 `AIMessage` 여야 한다. 
그러므로 이러한 변환 처리를 해주어야 정상적인 시뮬레이션을 수행할 수 있다.  

```python
def _swap_roles(messages):
  """
  메시지의 역할을 교환하는 함수
  시뮬레이션 사용자 단계에서 메시지 타입을 아래와 같이 교환한다.
  AI -> Human
  Human -> AI
  """

  new_messages = []
  for m in messages:
    if isinstance(m, AIMessage):
      new_messages.append(HumanMessage(content=m.content))
    else:
      new_messages.append(AIMessage(content=m.content))

  return new_messages


def simulated_user_node(state: AgentState):
  """
  시뮬레이션 사용자 노드 함수
  """
  new_messages = _swap_roles(state['messages'])

  response = simulated_user.invoke({'messages' : new_messages})

  return {'messages' : [HumanMessage(content=response)]}
```  

### Simulation Graph
그래프 정의에 앞서 그래프 순환의 종료를 판단하는 조건부 엣지를 정의한다. 
그렇지 않으면 시뮬레이션은 무한하게 진행될 수 있기 때문에 최대 6회의 대화 혹은 `FINISHED` 메시지가 오면 종료하도록 한다.  

```python
def should_continue(state: AgentState):
  """
  시뮬레이션의 종료 시점 또는 조건을 설정한다. (설정하지 않으면 시뮬레이션은 무한하게 진행될 수 있다.)
  여기서는 상태의 메시지 개수로 판단해 종료처리한다.
  그리고 FINISHED 라고 답한 경우에도 종료한다.
  """

  if len(state['messages']) > 6:
    return 'end'
  elif state['messages'][-1].content == 'FINISHED':
    return 'end'
  else:
    return 'continue'
```  

이제 정의된 내용들을 바탕으로 `Simulation Graph` 를 정의한다.  

```python
from langgraph.graph import START, END, StateGraph
from IPython.display import Image, display

graph_builder = StateGraph(AgentState)

graph_builder.add_node('simulated_user', simulated_user_node)
graph_builder.add_node('ai_assistant', ai_assistant_node)

graph_builder.add_edge('ai_assistant', 'simulated_user')

graph_builder.add_conditional_edges(
    'simulated_user',
    should_continue,
    {
        'end' : END,
        'continue' : 'ai_assistant'
    }
)

graph_builder.set_entry_point('ai_assistant')

graph = graph_builder.compile()


# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

