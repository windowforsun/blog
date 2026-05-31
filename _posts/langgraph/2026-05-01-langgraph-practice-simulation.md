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


![그림 1]({{site.baseurl}}/img/langgraph/simulation-1.png)


구현된 `Simulation Graph` 를 실행해 테스트를 하면 아래와 같은 결과를 확인할 수 있다.  

```python
# 시뮬레이션 수행

from langchain_core.runnables import RunnableConfig
import uuid


def execute_graph(graph, config, inputs):

  for event in graph.stream(inputs, config, stream_mode="updates"):
    # 업데이트 딕셔너리 순회
    for k, v in event.items():
        # 메시지 목록 출력
        print(k)
        print(v)
          

config = RunnableConfig(recursion_limit=20, configurable={'thread_id': uuid.uuid4()})


execute_graph(graph, config, {'messages' : ['안녕하세요. 기술 지원이 필요해요!']})
# ai_assistant
# {'messages': [AIMessage(content='안녕하세요! 어떤 기술 지원이 필요하신가요? 최대한 자세하게 설명해주시면 문제 해결에 도움이 될 수 있습니다. 예를 들어 다음과 같은 정보를 알려주시면 더욱 정확한 지원을 제공해 드릴 수 있습니다:\n\n*   **어떤 문제인가요?** (예: 특정 프로그램 오류, 인터넷 연결 문제, 하드웨어 작동 불량 등)\n*   **어떤 장비/소프트웨어 관련 문제인가요?** (예: 윈도우 10, 맥OS, 특정 프로그램 이름, 프린터 모델명 등)\n*   **어떤 작업을 하다가 문제가 발생했나요?** (구체적인 상황 설명)\n*   **오류 메시지가 있다면 정확한 내용이 무엇인가요?** (가능하면 스크린샷 첨부)\n*   **어떤 시도를 해보셨나요?** (이미 시도해본 해결 방법)\n\n자세한 정보를 알려주시면 신속하게 문제 해결을 도와드리겠습니다. 기다리고 있겠습니다!', additional_kwargs={}, response_metadata={}, id='23049a60-59f4-49c0-914c-e87b904c5526')]}
# simulated_user
# {'messages': [HumanMessage(content='새로운 프로젝트에 AI, LLM, LangChain, LangGraph 같은 최신 기술들을 도입하려고 하는데 어려움이 많네요. 제가 이 분야에 대한 지식이 부족해서 어디서부터 시작해야 할지 막막합니다. 혹시 기본적인 개념부터 차근차근 설명해주실 수 있을까요? 예를 들어, LLM이 정확히 무엇이고, 어떤 방식으로 작동하는지, 그리고 LangChain과 LangGraph는 LLM과 어떤 관계를 가지는지 궁금합니다. 실무에서 어떻게 활용되는지도 알려주시면 정말 감사하겠습니다.', additional_kwargs={}, response_metadata={}, id='ea405e1e-0c43-4014-9624-9cbf13e32487')]}
# ai_assistant
# {'messages': [AIMessage(content='네, 최신 AI 기술 도입에 어려움을 겪고 계시는군요. LLM, LangChain, LangGraph는 현재 AI 분야에서 매우 중요한 기술들이지만, 처음 접하면 복잡하게 느껴질 수 있습니다. 기본적인 개념부터 실무 활용까지 차근차근 설명해 드리겠습니다.\n\n**1. LLM (Large Language Model, 거대 언어 모델) 이란?**\n\n*   **정의:** LLM은 방대한 양의 텍스트 데이터를 학습하여 인간의 언어를 이해하고 생성할 수 있는 인공 신경망 모델입니다. 쉽게 말해, "엄청나게 많은 글을 읽고 학습해서 사람처럼 글을 쓰고 대화할 수 있는 AI" 라고 생각하시면 됩니다.\n*   **작동 방식:**\n    *   **학습:** 인터넷, 책, 논문 등 엄청난 양의 텍스트 데이터를 읽고 단어, 문장, 문맥 간의 관계를 학습합니다.\n    *   **예측:** 학습된 내용을 바탕으로 주어진 텍스트의 다음 단어나 문장을 예측합니다. 이러한 예측 과정을 반복하여 자연스러운 문장을 생성하거나 질문에 답변할 수 있습니다.\n    *   **Transformer 아키텍처:** 대부분의 LLM은 Transformer라는 신경망 아키텍처를 기반으로 합니다. Transformer는 문장 내 단어들의 관계를 효과적으로 파악하여 장문 텍스트도 잘 처리할 수 있는 장점이 있습니다.\n*   **예시:** GPT-3, GPT-4, PaLM, LLaMA 등이 대표적인 LLM입니다.\n*   **핵심 능력:** 텍스트 생성, 텍스트 요약, 번역, 질문 답변, 코드 생성 등 다양한 자연어 처리 작업을 수행할 수 있습니다.\n\n**2. LangChain 이란?**\n\n*   **정의:** LLM을 활용한 어플리케이션 개발을 위한 프레임워크입니다. LLM을 다양한 데이터 소스, 다른 도구, 다른 LLM과 연결하여 더욱 강력하고 복잡한 기능을 구현할 수 있도록 도와줍니다.\n*   **역할:** LLM을 단순히 텍스트 생성 도구로 사용하는 것을 넘어, LLM을 기반으로 하는 어플리케이션을 구축하는 데 필요한 다양한 기능을 제공합니다.\n*   **핵심 기능:**\n    *   **모델 I/O:** LLM에 데이터를 입력하고 결과를 출력하는 과정을 관리합니다.\n    *   **데이터 연결:** 다양한 데이터 소스 (웹 페이지, 데이터베이스, API 등)에서 데이터를 가져와 LLM에 제공합니다.\n    *   **체인:** 여러 LLM을 연결하거나, LLM과 다른 도구를 연결하여 복잡한 작업을 수행하는 워크플로우를 정의합니다.\n    *   **에이전트:** LLM을 사용하여 사용자의 목표를 달성하기 위한 행동 계획을 자동으로 생성하고 실행합니다. (예: "오늘 서울 날씨를 알아보고, 미세먼지가 심하면 마스크를 사러 가는 계획을 세워줘")\n    *   **메모리:** LLM이 이전 대화 내용을 기억하고 활용하여 더욱 자연스러운 대화를 이어갈 수 있도록 합니다.\n*   **예시:** 챗봇, 문서 요약 도구, 질의 응답 시스템 등을 LangChain을 사용하여 개발할 수 있습니다.\n\n**3. LangGraph 란?**\n\n*   **정의:** LangChain의 확장판으로, LLM을 기반으로 하는 어플리케이션의 복잡한 워크플로우를 시각적으로 정의하고 관리할 수 있도록 도와주는 라이브러리입니다.\n*   **역할:** LangChain의 체인 기능을 더욱 발전시켜, 여러 단계를 거치는 복잡한 작업을 보다 유연하고 효율적으로 처리할 수 있도록 합니다.\n*   **핵심 기능:**\n    *   **그래프 기반 워크플로우:** 어플리케이션의 워크플로우를 노드와 엣지로 구성된 그래프 형태로 표현합니다. 각 노드는 LLM이나 도구를 나타내고, 엣지는 데이터의 흐름을 나타냅니다.\n    *   **상태 관리:** 워크플로우의 각 단계에서 발생하는 상태를 관리하고, 필요에 따라 이전 단계로 돌아가거나 다른 경로로 분기할 수 있습니다.\n    *   **병렬 처리:** 여러 단계를 동시에 실행하여 처리 속도를 향상시킬 수 있습니다.\n    *   **시각화:** 워크플로우를 시각적으로 표현하여 이해하기 쉽고, 디버깅하기도 용이합니다.\n*   **활용 예시:**\n    *   **복잡한 의사 결정 시스템:** 다양한 정보를 수집하고 분석하여 최적의 결정을 내리는 시스템 (예: 금융 투자 의사 결정, 의료 진단 시스템)\n    *   **자동화된 데이터 처리 파이프라인:** 데이터를 수집, 변환, 분석하는 과정을 자동화하는 시스템\n    *   **멀티 에이전트 시스템:** 여러 개의 에이전트가 협력하여 하나의 목표를 달성하는 시스템\n\n**4. 실무 활용 예시:**\n\n*   **챗봇 개발:** LLM을 사용하여 사용자의 질문에 답변하고, LangChain을 사용하여 외부 데이터베이스나 API와 연결하여 더욱 풍부한 정보를 제공하는 챗봇을 개발할 수 있습니다. LangGraph를 사용하면 복잡한 사용자 요청에 따라 여러 단계를 거치는 챗봇을 만들 수 있습니다. (예: "여행 계획 세워줘" -> 여행 정보 검색 -> 호텔 예약 -> 항공권 예약)\n*   **문서 요약:** LLM을 사용하여 긴 문서를 요약하고, LangChain을 사용하여 여러 문서를 연결하여 전체 내용을 요약하는 시스템을 개발할 수 있습니다.\n*   **코드 생성:** LLM을 사용하여 자연어 설명에 따라 코드를 생성하고, LangChain을 사용하여 생성된 코드를 테스트하고 디버깅하는 시스템을 개발할 수 있습니다.\n*   **데이터 분석:** LLM을 사용하여 데이터 분석 결과를 자연어로 설명하고, LangChain을 사용하여 데이터 분석 파이프라인을 자동화하는 시스템을 개발할 수 있습니다.\n\n**5. 시작하기 위한 조언:**\n\n*   **기본 개념 학습:** LLM, LangChain, LangGraph의 기본 개념을 이해하는 것이 중요합니다. 관련 자료를 찾아보거나 온라인 강좌를 수강하는 것을 추천합니다.\n*   **간단한 예제 따라하기:** LangChain 공식 문서나 튜토리얼을 참고하여 간단한 예제를 따라해 보면서 실제로 코드를 작성해 보는 것이 좋습니다.\n*   **오픈 소스 프로젝트 참여:** 오픈 소스 프로젝트에 참여하여 다른 개발자들과 함께 코드를 작성하고 배우는 것도 좋은 방법입니다.\n*   **커뮤니티 활용:** LangChain, LangGraph 관련 커뮤니티에 참여하여 질문하고 정보를 공유하세요.\n\n이 설명이 LLM, LangChain, LangGraph에 대한 이해를 돕는 데 도움이 되었기를 바랍니다. 더 궁금한 점이 있다면 언제든지 질문해주세요!', additional_kwargs={}, response_metadata={}, id='5c614cea-6f33-4411-afdc-1cc7971ff783')]}
# simulated_user
# {'messages': [HumanMessage(content='정말 자세하고 친절한 설명 감사합니다! 덕분에 LLM, LangChain, LangGraph에 대한 기본적인 이해를 할 수 있게 되었습니다.\n\n설명해주신 내용 중에 "에이전트"라는 개념이 특히 흥미로운데요. 에이전트가 사용자의 목표를 달성하기 위한 행동 계획을 자동으로 생성하고 실행한다고 하셨는데, 좀 더 구체적인 예시를 들어서 설명해주실 수 있을까요? 예를 들어, 어떤 종류의 목표를 에이전트가 자동으로 달성할 수 있으며, 어떤 방식으로 행동 계획을 생성하고 실행하는지 궁금합니다. 그리고 에이전트를 만들 때 고려해야 할 중요한 요소들이 있다면 무엇인지도 알려주시면 감사하겠습니다.', additional_kwargs={}, response_metadata={}, id='977bdb27-6c88-49fb-893c-b1cdd75f5cf9')]}
# ai_assistant
# {'messages': [AIMessage(content='네, 에이전트 기능에 흥미를 느끼셨다니 다행입니다. LangChain에서 에이전트는 매우 강력한 도구이며, LLM의 잠재력을 최대한으로 활용할 수 있게 해줍니다. 에이전트의 작동 방식과 고려 사항에 대해 자세히 설명드리겠습니다.\n\n**1. 에이전트의 작동 방식: 목표 달성을 위한 자동 행동 계획 생성 및 실행**\n\n에이전트는 사용자의 목표를 달성하기 위해 다음과 같은 단계를 거쳐 행동 계획을 생성하고 실행합니다.\n\n1.  **목표 정의 (Goal Definition):** 사용자는 에이전트에게 달성하고자 하는 목표를 자연어로 제시합니다. 예를 들어, "오늘 서울 날씨를 알아보고, 미세먼지가 심하면 마스크를 사러 가는 계획을 세워줘" 와 같이 구체적인 목표를 설정할 수 있습니다.\n\n2.  **도구 선택 (Tool Selection):** 에이전트는 목표를 달성하기 위해 사용할 수 있는 도구 목록을 가지고 있습니다. 도구는 API, 데이터베이스, 웹 검색 엔진, 계산기 등 다양한 형태를 가질 수 있습니다. 에이전트는 목표를 분석하여 어떤 도구를 사용해야 하는지 판단합니다. 예를 들어, 날씨를 알아보기 위해서는 날씨 API 도구를 선택하고, 마스크를 구매하기 위한 계획을 세우려면 웹 검색 엔진이나 쇼핑 API 도구를 선택할 수 있습니다.\n\n3.  **행동 계획 생성 (Action Plan Generation):** 에이전트는 LLM을 사용하여 목표를 달성하기 위한 구체적인 행동 계획을 생성합니다. 행동 계획은 일련의 단계로 구성되며, 각 단계는 어떤 도구를 사용하여 어떤 작업을 수행해야 하는지 명시합니다. 예를 들어, 다음과 같은 행동 계획을 생성할 수 있습니다.\n\n    *   단계 1: 날씨 API 도구를 사용하여 오늘 서울 날씨 정보를 가져온다.\n    *   단계 2: 날씨 정보에서 미세먼지 농도를 확인한다.\n    *   단계 3: 미세먼지 농도가 높으면, 웹 검색 엔진을 사용하여 가까운 마스크 판매점을 검색한다.\n    *   단계 4: 마스크 판매점 위치와 영업시간 정보를 확인하고, 방문 계획을 세운다.\n\n4.  **행동 실행 (Action Execution):** 에이전트는 생성된 행동 계획에 따라 각 단계를 실행합니다. 각 단계에서 필요한 도구를 호출하고, 결과를 LLM에 전달합니다. 예를 들어, 날씨 API 도구를 호출하여 날씨 정보를 가져오고, 웹 검색 엔진을 호출하여 마스크 판매점 정보를 검색합니다.\n\n5.  **관찰 (Observation):** 각 단계를 실행한 결과를 관찰하고, LLM을 사용하여 결과를 분석합니다. 결과를 바탕으로 다음 단계를 결정하거나, 목표를 달성했는지 확인합니다. 예를 들어, 날씨 API에서 가져온 날씨 정보를 분석하여 미세먼지 농도가 높으면 다음 단계로 진행하고, 미세먼지 농도가 낮으면 목표를 달성했다고 판단합니다.\n\n6.  **반복 (Iteration):** 목표를 달성할 때까지 2단계부터 5단계까지 반복합니다. 에이전트는 각 단계를 실행하면서 얻은 정보를 바탕으로 행동 계획을 수정하거나, 새로운 도구를 사용할 수 있습니다.\n\n**2. 에이전트가 자동으로 달성할 수 있는 목표 종류**\n\n에이전트는 다음과 같은 다양한 종류의 목표를 자동으로 달성할 수 있습니다.\n\n*   **정보 검색 (Information Retrieval):** 웹 검색 엔진, API, 데이터베이스 등을 사용하여 특정 정보를 검색하고 요약합니다. (예: "최근 주식 시장 동향을 알아봐줘", "특정 논문에 대한 정보를 찾아줘")\n*   **질문 답변 (Question Answering):** 주어진 문서나 텍스트에 대한 질문에 답변합니다. (예: "특정 법률 조항에 대한 설명을 해줘", "특정 제품의 장단점을 비교해줘")\n*   **계획 수립 (Planning):** 여행 계획, 쇼핑 계획, 일정 관리 등 다양한 계획을 수립합니다. (예: "이번 주말 서울 여행 계획을 세워줘", "다이어트 식단을 짜줘")\n*   **코드 생성 (Code Generation):** 자연어 설명을 기반으로 코드를 생성합니다. (예: "특정 웹 페이지를 크롤링하는 파이썬 코드를 만들어줘", "간단한 게임을 만들어줘")\n*   **자동화 (Automation):** 반복적인 작업을 자동화합니다. (예: "매일 아침 9시에 주식 시황을 요약해서 이메일로 보내줘", "새로운 트윗이 올라오면 자동으로 리트윗해줘")\n*   **의사 결정 지원 (Decision Support):** 다양한 정보를 분석하여 의사 결정을 지원합니다. (예: "특정 투자에 대한 위험도를 분석해줘", "새로운 제품 출시 전략을 제안해줘")\n\n**3. 에이전트를 만들 때 고려해야 할 중요한 요소**\n\n에이전트를 만들 때 다음과 같은 요소들을 고려해야 합니다.\n\n*   **목표 명확성 (Goal Clarity):** 에이전트에게 명확하고 구체적인 목표를 제시해야 합니다. 목표가 모호하면 에이전트가 올바른 행동 계획을 생성하기 어렵습니다.\n*   **도구 선택 (Tool Selection):** 에이전트가 사용할 수 있는 도구를 신중하게 선택해야 합니다. 목표 달성에 필요한 도구를 모두 포함하고, 불필요한 도구는 제거해야 합니다. 도구의 성능과 신뢰성도 중요합니다.\n*   **제한 (Constraints):** 에이전트의 행동에 제약을 가해야 합니다. 무제한적인 행동은 예상치 못한 결과를 초래할 수 있습니다. 예를 들어, 에이전트가 사용할 수 있는 API 호출 횟수를 제한하거나, 특정 웹 사이트에 접근하는 것을 금지할 수 있습니다.\n*   **안전성 (Safety):** 에이전트가 악의적인 목적으로 사용되지 않도록 안전 장치를 마련해야 합니다. 예를 들어, 에이전트가 개인 정보를 유출하거나, 허위 정보를 퍼뜨리는 것을 방지해야 합니다.\n*   **피드백 루프 (Feedback Loop):** 에이전트의 행동 결과를 평가하고, 개선할 수 있는 피드백 루프를 구축해야 합니다. 사용자의 피드백을 수집하거나, 에이전트의 성능을 모니터링하여 문제점을 파악하고 개선해야 합니다.\n*   **모니터링 (Monitoring):** 에이전트의 행동을 지속적으로 모니터링해야 합니다. 에이전트가 예상대로 작동하는지, 오류가 발생하는지, 자원을 효율적으로 사용하는지 등을 확인해야 합니다.\n*   **설명 가능성 (Explainability):** 에이전트가 왜 특정 행동을 선택했는지 설명할 수 있어야 합니다. 이는 사용자의 신뢰를 얻고, 문제 발생 시 원인을 파악하는 데 도움이 됩니다.\n\n**4. 예시: 여행 계획 에이전트**\n\n여행 계획 에이전트를 예시로 들어보겠습니다.\n\n*   **목표:** "이번 주말에 서울로 1박 2일 여행을 가고 싶어. 숙소는 호텔이고, 맛집도 추천해줘."\n*   **도구:**\n    *   호텔 예약 API\n    *   맛집 정보 API\n    *   웹 검색 엔진\n    *   지도 API\n*   **행동 계획:**\n    *   단계 1: 웹 검색 엔진을 사용하여 서울의 인기 호텔 정보를 검색한다.\n    *   단계 2: 호텔 예약 API를 사용하여 검색된 호텔의 가격과 예약 가능 여부를 확인한다.\n    *   단계 3: 사용자의 선호도 (가격, 위치, 평점 등)에 따라 호텔을 선택하고 예약한다.\n    *   단계 4: 맛집 정보 API를 사용하여 호텔 주변의 맛집 정보를 검색한다.\n    *   단계 5: 사용자의 선호도 (음식 종류, 가격, 평점 등)에 따라 맛집을 추천한다.\n    *   단계 6: 지도 API를 사용하여 호텔과 맛집의 위치를 확인하고, 이동 경로를 안내한다.\n\n이처럼 에이전트는 LLM과 다양한 도구를 결합하여 사용자의 목표를 달성하는 데 필요한 복잡한 작업을 자동화할 수 있습니다. 에이전트 개발은 초기 단계이지만, 앞으로 다양한 분야에서 혁신적인 변화를 가져올 것으로 기대됩니다.\n\n에이전트에 대한 궁금증이 해소되셨기를 바랍니다. 더 궁금한 점이 있다면 언제든지 질문해주세요!', additional_kwargs={}, response_metadata={}, id='dcc73c7e-2721-46ef-8f45-b7ba5b854ae4')]}
# simulated_user
# {'messages': [HumanMessage(content='정말 명쾌한 설명 감사합니다! 에이전트의 작동 방식부터 고려해야 할 요소들, 그리고 구체적인 예시까지 자세하게 설명해주셔서 에이전트라는 개념에 대해 완벽하게 이해할 수 있게 되었습니다. 특히 에이전트를 만들 때 목표 명확성, 도구 선택, 안전성, 피드백 루프 등이 중요하다는 점을 알게 되었습니다.\n\n이제 에이전트, LLM, LangChain에 대한 기본적인 이해는 어느 정도 갖추게 된 것 같습니다. 다음 단계로 넘어가서 LangGraph에 대해 좀 더 깊이 있게 알아보고 싶은데요. LangGraph는 복잡한 워크플로우를 시각적으로 정의하고 관리할 수 있도록 도와준다고 하셨는데, 실제로 LangGraph를 사용하여 복잡한 워크플로우를 구축하는 과정을 예시를 통해 설명해주실 수 있을까요? 예를 들어, "고객 문의 처리 시스템"과 같은 복잡한 워크플로우를 LangGraph로 어떻게 구현할 수 있는지 궁금합니다. 노드와 엣지를 어떻게 구성하고, 상태 관리는 어떻게 하는지, 그리고 병렬 처리는 어떻게 적용할 수 있는지 자세하게 알려주시면 감사하겠습니다.', additional_kwargs={}, response_metadata={}, id='edcc195a-a7cd-4e00-b9ec-24fb950a5b27')]}
```  



---  
## Reference
[Chat Bot Evaluation as Multi-agent Simulation](https://langchain-ai.github.io/langgraph/tutorials/chatbot-simulation-evaluation/agent-simulation-evaluation/)  
[Multi-Agent Hedge Fund Simulation with LangChain and LangGraph](https://shaikhmubin.medium.com/multi-agent-hedge-fund-simulation-with-langchain-and-langgraph-64060aabe711)  
[대화 시뮬레이션](https://wikidocs.net/267816)  


