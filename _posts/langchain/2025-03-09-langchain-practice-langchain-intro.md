--- 
layout: single
classes: wide
title: "[LangChain] LangChain Introduction"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: 'LLM 을 활용한 애플리케이션 개발을 쉽게 할 수 있도록 도와주는 프레임워크인 Langchain 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
toc: true
use_math: true
---  

## LangChain
`LangChain` 은 `LLM(Large Language Model)` 을 활용한 애플리케이션 개발을 쉽게 할 수 있도록 도와주는 프레임워크이다. 
다양한 `LLM` 사용 및 연결해 `Chain` 형태로 조합할 수 있도록 지원한다. 
이를 통해 대화형 AI, 자동화된 데이터 처리, 검색 및 요약 기능을 포함한 다양한 자연어 처리 애플리케이션을 개발할 수 있다.  

### 주요 기능 및 구성 요소
`LangChain` 은 여러 가지 핵심 구성 요소를 통해 강력한 AI 애플리케이션 개발을 지원한다.  

#### LLM Wrappers
`LangChain` 은 다양한 `LLM` 을 사용할 수 있도록 `Wrapper` 를 제공한다. 
이를 통해 쉽게 연겨할 수 있도록 API 래퍼를 제공해 사용자는 간단한 설정으로 원하는 모델을 활용할 수 있다.  

#### Prompt Templates
프롬프트는 `LLM` 과 상호작용에서 중요한 요소이다. 
`LangChain` 은 유연한 프롬프트 관리 기능을 제공하여 다양한 입력을 조합하거나 최적화된 프롬프트를 설계하는 데 도움을 준다.  


#### Chains
체인은 여러 개의 `LLM` 호출 또는 다른 연산을 연걸하는 개념이다. 
예를 들어 사용자의 입력을 분석한 뒤 데이터를 검색하고, 결과를 요약하여 반환하는 일련의 과정이 체인으로 구성될 수 있다.  

- `Sequential Chain` : 단계별로 `LLM` 을 호출하여 연속적인 작업을 수행한다. 
- `Router Chain` : 사용자의 입력을 기반으로 적절한 체인을 선택하여 실행한다. 

#### Memory
`LangChain` 은 대화형 애플리케이션에서 문맥을 유지할 수 있도록 메모리 기능을 제공한다. 
이를 통해 이전 대화 내용을 기억하고, 연속적인 대화를 보다 자연스럽게 처리할 수 있다.  

- `ConversationBufferMemory` : 전체 대화 내용 저장
- `ConversationSummaryMemory` : 요약된 대화 내용을 저장
- `VectorDBMemory` : 벡터 데이터베이스를 활용한 문맥 유지

#### Agents
에이전트는 `LLM` 이 다양한 도구와 상호작용할 수 있도록 하는 기능이다. 
예를 들어 외부 API를 호출하여 정보를 가져오거나 데이터베이스에서 필요한 데이터를 검색하는 등의 작업을 수행할 수 있다.  

#### Tools
`LangChain` 은 계산기, 검색 엔진, API 호출 등의 다양한 외부 도구와 휩게 연동할 수 있도록 설계되어 있다. 
이를 활용해 단순한 텍스트 응답을 넘어 더욱 복잡한 작업을 수행할 수 있다.  


### Architecture
`LangChain` 은 아래와 같은 여러 패키지로 구성된 프레임워크이다. 

![그림 1]({{site.baseurl}}/img/langchain/langchain-intro-1.svg)


#### langchain-core
다양한 구성 요소에 대한 기본 추상화와 함께 구성하는 방법을 포함한다. 
채팅 모델, 벡터 저장소, 도구 등의 핵심 구성 요소에 대한 인터페이스가 정의되어 있다. 

#### langchain
주요 `LangChain` 패키지는 애플리케이션의 인지 아키텍터를 구성하는 `Chain` 과 `Retrieval` 전략을 포함한다. 

### Integration Packages
`langchain-openai`, `langchain-anthropics` 등 인기 있는 통합 기능들은 별도 패키지로 제공된다. 

- 전체 페키지 목록 : https://python.langchain.com/docs/integrations/providers/
- API Reference : https://python.langchain.com/api_reference/

#### langchain-community
`LangChain` 커뮤니티에서 유지 미 관리하는 서드파티 통합 기능을 포함하고 있다. 
중요한 패키지는 `Integration Packages` 에 분라돼 포함되어 있다. 
채팅 모델, 벡터 저장소, 도구 등 다양한 구성 요소에 대한 통합 기능이 포함된다.  


#### langgraph
`LangChain` 을 확장하여, `Multi-Actor` 기반의 상태를 유지하는 애플리케이션을 구축하기 위한 패키지이다. 
각 단계를 `Graph` 의 `Node` 와 `Edge` 로 모델링하는 방식으로 설계돼 있다. 
일반적인 에이전트 구현을 위한 고수준 인터페이스와 사용자 맞춤형 흐름을 구성할 수 있는 저수준 인터페이스를 제공한다. 

- langgraph overview : https://langchain-ai.github.io/langgraph/concepts/high_level/#core-principles

#### langserve
`LangChain` 체인을 `REST API` 로 배포할 수 있도록 도와주는 패키지이다. 
이를 통해 프로덕션 환경에서 실행 가능한 `API` 를 손쉽게 설정할 수 있다.  

주로 간단한 `Runnable` 을 배포하고 `langchain-core` 의 기본 요소들과 함께 사용하도록 설계돼있다. 
그러므로 `LangGraph` 를 배포하는 경우 `LangGraph Platform` 을 사용하는 것이 좋다.  

- langserve overview : https://python.langchain.com/docs/langserve/


#### LangSmith
개발자가 `LLM` 애플리케이션을 디버깅, 테스트, 평가, 모니터링할 수 있도록 지원하는 개발자 플랫폼이다.  

- LangSmith overview : https://docs.smith.langchain.com/


### Demo
데모는 `LangChain` 의 몇가지 기본 기능을 사용하는 간단한 예시이다. 
`LLM` 모델은 `Groq` 의 `llama3-8b-8192` 을 사용해 진행했다.  

우선 `LangChain` 을 사용하기 위해 아래 패키지를 설치해 준다.  

```bash
$ pip install langchain

$ pip install -qU "langchain[groq]"
```

```python
# 기본적인 LLM 호출

import os
import getpass
from langchain.chat_models import init_chat_model
from langchain_core.messages import HumanMessage, SystemMessage

os.environ['GROQ_API_KEY'] = getpass.getpass('인증키 입력')

model = init_chat_model("llama3-8b-8192", model_provider="groq")


messages = [
    SystemMessage("숫자의 제곱근을 구한다."),
    HumanMessage("4")
]

response = model.invoke(messages)

print(response.content)
# The square root of 4 is 2.
```  

```python
# 템플릿 및 체인 사용
import os
import getpass
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate


os.environ['GROQ_API_KEY'] = getpass.getpass('인증키 입력')

model = init_chat_model("llama3-8b-8192", model_provider="groq")

prompt = ChatPromptTemplate.from_template("{num} 의 제곱근은 ?")

chain = prompt | model | StrOutputParser()

chain.invoke({"num": 4})
# 4의 제곱근은 2입니다.
```  

```python
# 메모리 사용
import os
import getpass
from langchain.chat_models import init_chat_model
from langchain_core.messages import HumanMessage
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import START, MessagesState, StateGraph

os.environ['GROQ_API_KEY'] = getpass.getpass('인증키 입력')
llm = init_chat_model("llama3-8b-8192", model_provider="groq")

workflow = StateGraph(state_schema=MessagesState)

def call_model(state: MessagesState):
    response = llm.invoke(state["messages"])

    return {"messages": response}


workflow.add_edge(START, "model")
workflow.add_node("model", call_model)

memory = MemorySaver()
app = workflow.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "abc123"}}

query = "내 나이는 20이고 키는 180이야"

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print() 

query = "내 이름과 키를 곱해줘"

input_messages = [HumanMessage(query)]
output = app.invoke({"messages": input_messages}, config)
output["messages"][-1].pretty_print()
# ================================== Ai Message ==================================
# 
# So you're 20 years old and 180 cm tall! That's a great age and height! 😊 How's life treating you so far? 😊
# ================================== Ai Message ==================================
# 
# Your name is not provided, so I'll assume you want me to multiply your age and height. 😊
# 
# Your age is 20 and your height is 180, so let's multiply them:
# 
# 20 × 180 = 3600
# 
# The result is 3600! 😊
```  





---  
## Reference
[Introduction](https://python.langchain.com/docs/introduction/)  
[Architecture](https://python.langchain.com/docs/concepts/architecture/)  


