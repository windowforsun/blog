--- 
layout: single
classes: wide
title: "[LangChain] LangChain Memory"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: 'LangChain 에서 Memory 를 사용해 이전 대화를 기억하고 맥락을 유지하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
    - Memory
    - ConversationChain
    - LLMChain
    - MessagePlaceholder
    - ConversationBufferMemory
    - ConversationBufferWindowMemory
    - ConversationTokenBufferMemory
    - ConversationSummaryMemory
    - ConversationSummaryBufferMemory
toc: true
use_math: true
---  

## Memory
`LangChain` 에서 `Memory` 는 `AI` 애플리케이션이 이전 대화를 기억하고 맥락을 유지할 수 있게 해주는 핵심 기능이다. 
`Memory` 없이는 언어 모델과의 각 상화작용이 독립적이고 상태가 없는 상태로 처리된다.
이렇게 언어 모델 자체는 이전 상호작용에 대한 내장 메모리가 없어, 각 요청은 독립적으로 처리된다. 
`Memory` 를 사용하면 아래와 같은 방식을 통해 이전 대화를 기억하고 맥락을 유지할 수 있다.

- 이전 대화 저장
- 관련 정보 검색
- 새로운 프롬프트에 컨텍스트 통합

`Memory` 를 활요해 `AI` 애플리케이션을 구현한다면,
`ConversationChain`, `LLMChain`, `MessagePlaceholder` 를 사용해 이전 대화를 기억하고 맥락을 유지할 수 있다.  

먼저 `ConversationChain` 의 경우 간단하고 빠르게 대화형 `AI` 를 구현할 수 있지만, 복잡한 시나리오에는 적합하지 않다. 

- 장점 
  - 간편함 : 대화형 `AI` 애플리케이션을 쉽게 만들 수 있도록 설계돼있다. 
  - 자동화된 메모리 관리 : 대화의 문맥을 자동으로 관리하여 사용자가 별도로 신경 쓸 필요가 없다. 
  - 빠른 구현 : 간단한 대화형 애플리케이션을 빠르게 구현할 수 있다. 
- 단점 
  - 제한된 유연성 : 복잡한 시나리오나 맞춤형 프롬프트를 필요로 하는 경우 유연성이 떨어질 수 있다. 
  - 확장성 부족 : 고급 기능이나 복잡한 대화 흐름을 구현하기에는 한계가 있다. 

```python
from langchain.chains import ConversationChain

memory = ConversationBufferMemory()
conversation = ConversationChain(
    llm=model,
    memory=memory
)

conversation.predict(input="제품 A/S를 받고 싶습니다.")
```  

다음으로 `LLMChain` 는 유연하고 확장성이 뛰어나지만, 설정과 사용성이 더 복잡하다. 

- 장점
  - 유연성 : 사용자 정의 프롬프트와 다양한 입력을 처리할 수 있어 복잡한 시나리오에 적합하다.
  - 고급 기능 : 고급 프롬프트 엔지니어링과 다양한 구성 요소와의 통합이 가능하다. 
  - 확장성 : 복잡한 대화 흐름과 고급 기능을 구현할 수 있다. 
- 단점
 - 복잡성 : 설정과 사용성이 `ConversationChain` 보다 복잡할 수 있다. 
 - 추가 작업 필요 : 메모리 관리와 프롬프트 설정에 더 많은 작업이 필요할 수 있다. 


```python
from langchain.chains import LLMChain
from langchain.prompts import ChatPromptTemplate

template = """
당신은 유능한 AI 어시스트턴트입니다. 사용자 질의에 적절하 답변을 제공하세요.
{chat_history}
Human: {input}
AI:
"""

prompt = ChatPromptTemplate.from_template(template)
memory = ConversationBufferMemory(memory_key="chat_history")
chain = LLMChain(llm=model, prompt=prompt, memory=memory)

chain.predict(input="안녕하세요, 제품 A/S를 받고 싶습니다.")
```  

마지막으로 `MessagePlaceholder` 는 세밀한 제어가 가능하지만, 
설정과 사용성이 더 복잡하고 추가 작업이 필요하다. 

- 장점
  - 세밀한 제어 : 대화의 특정 부분을 동적으로 변경하거나 업데이트할 수 있어 세밀한 제어가 가능하다.
  - 유연성 : 다양한 시나리오에서 유연하게 사용할 수 있다. 
- 단점
  - 복잡성 : 설정과 사용성이 더 복잡할 수 있으며, 잘못 사용하면 오류가 발생하 수 있다.
  - 추가 작업 필요 : 다른 구성 요소와의 통합 및 관리에 더 많은 작업이 필요할 수 있다. 

```python
from langchain_core.prompts import MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser

memory = ConversationBufferMemory(return_messages=True, message_key="chat_history")

prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "당신은 유능한 AI 어시스턴트입니다. 사용자 질의에 적절한 답변을 제공하세요."),
        MessagesPlaceholder(variable_name="chat_history"),
        ("human", "{input}")
    ]
)

chain = prompt | model | StrOutputParser()

chain.invoke({
    "input" : "제품 A/S를 받고 싶습니다.",
    "chat_history" : memory.chat_memory.messages
})
```  



### ConversationBufferMemory
`ConversationBufferMemory` 는 가장 기본적이면서 직관적인 메모리 유형이다. 
대화 내용을 버퍼처럼 저장하는 메모리 컴포넌트이다. 
이는 모든 대화의 턴을 순서대로 저장한다. 
간단히 설명하면 모든 대화 내용을 그대로 기억하는 메모리인 것이다. 

- 작동방식 : 사용자 입력과 `AI` 응답 쌍을 순새대로 저장한다. 
- 접근방식 : 새로운 대화 차례마다 이전의 모든 대화 내용을 불러온다. 
- 형식 : 기본적으로 문자열 형태로 대화 내용을 관리하며, `return_messages=True` 옵션을 통해 메시지 객체 형태로도 반환할 수 있다.  

장점으로는 아래와 같은 것들이 있다. 

- 가장 직관적인 메모리 방식이기 때문에 구현이 간단하다. 
- 대화의 모든 부분을 기억하기 때문에 컨텍스트 손실이 없어 완전한 컨텍스트를 유지할 수 있다.
- 대화 흐름을 쉽게 추적할 수 있어 디버깅에 용이하다. 

단점으로는 아래와 같은 것들이 있다. 

- 대화가 길어질수록 컨텍스트 크기가 계속 커져 토큰 소비가 증가한다. 
- 언어 모델의 컨텍스트 창 크기를 초과하면 오류가 발생할 수 있다. 
- 오래된 대화도 모두 포함되므로 현재 대화와 관련 없는 내용까지 포함된다. 


```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory()
memory.save_context(
    inputs = {
        "human" : "안녕하세요, 제품 A/S를 받고 싶습니다."
    },
    outputs = {
        "ai" : "안녕하세요! 어떤 문제가 발생했나요?"
    }
)

memory.load_memory_variables({})
# <ipython-input-4-64aee8a38692>:3: LangChainDeprecationWarning: Please see the migration guide at: https://python.langchain.com/docs/versions/migrating_memory/
# memory = ConversationBufferMemory()
# {'history': 'Human: 안녕하세요, 제품 A/S를 받고 싶습니다.\nAI: 안녕하세요! 어떤 문제가 발생했나요?'}

memory.save_context(
    inputs = {
        "human" : "제품이 작동하지 않습니다."
    },
    outputs = {
        "ai" : "어떤 제품인가요?"
    }
)

memory.save_context(
    inputs = {
        "human" : "스마트폰입니다."
    },
    outputs = {
        "ai" : "스마트폰 모델명을 알려주시겠어요?"
    }
)

memory.save_context(
    inputs={"human": "모델명은 XYZ123입니다."},
    outputs={
        "ai": "언제 구매하셨나요?"
    },
)

memory.load_memory_variables({})['history']
# Human: 안녕하세요, 제품 A/S를 받고 싶습니다.\nAI: 안녕하세요! 어떤 문제가 발생했나요?\nHuman: 제품이 작동하지 않습니다.\nAI: 어떤 제품인가요?\nHuman: 스마트폰입니다.\nAI: 스마트폰 모델명을 알려주시겠어요?\nHuman: 모델명은 XYZ123입니다.\nAI: 언제 구매하셨나요?

memory.save_context(
    inputs={"human": "6개월 전에 구매했습니다."},
    outputs={
        "ai": "보증 기간 내에 있으므로 무상 수리가 가능합니다. 가까운 서비스 센터를 방문해 주세요."
    },
)
memory.save_context(
    inputs={"human": "서비스 센터 위치를 알려주세요."},
    outputs={
        "ai": "고객님의 위치를 알려주시면 가장 가까운 서비스 센터를 안내해 드리겠습니다."
    },
)

memory.load_memory_variables({})['history']
# Human: 안녕하세요, 제품 A/S를 받고 싶습니다.\nAI: 안녕하세요! 어떤 문제가 발생했나요?\nHuman: 제품이 작동하지 않습니다.\nAI: 어떤 제품인가요?\nHuman: 스마트폰입니다.\nAI: 스마트폰 모델명을 알려주시겠어요?\nHuman: 모델명은 XYZ123입니다.\nAI: 언제 구매하셨나요?\nHuman: 6개월 전에 구매했습니다.\nAI: 보증 기간 내에 있으므로 무상 수리가 가능합니다. 가까운 서비스 센터를 방문해 주세요.\nHuman: 서비스 센터 위치를 알려주세요.\nAI: 고객님의 위치를 알려주시면 가장 가까운 서비스 센터를 안내해 드리겠습니다.



# or memory = ConversationBufferMemory(return_messages=True)
memory.return_messages = True

memory.load_memory_variables({})['history']
# [HumanMessage(content='안녕하세요, 제품 A/S를 받고 싶습니다.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='안녕하세요! 어떤 문제가 발생했나요?', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='제품이 작동하지 않습니다.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='어떤 제품인가요?', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='스마트폰입니다.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='스마트폰 모델명을 알려주시겠어요?', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='모델명은 XYZ123입니다.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='언제 구매하셨나요?', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='6개월 전에 구매했습니다.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='보증 기간 내에 있으므로 무상 수리가 가능합니다. 가까운 서비스 센터를 방문해 주세요.', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='서비스 센터 위치를 알려주세요.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='고객님의 위치를 알려주시면 가장 가까운 서비스 센터를 안내해 드리겠습니다.', additional_kwargs={}, response_metadata={})] 
```

### ConversationBufferWindowMemory
`ConversationBufferWindowMemory` 는 슬라이딩 윈도우처럼 작동해 모든 대화를 저장하는 것이 아니라, 가장 최근의 일정 수(`K`)의 대화 턴만 유지한다. 
그리고 오래된 대화는 자동으로 메모리에서 제거된다. 

- 윈도우 크기 설정 : `K` 매개변수를 통해 기억할 대화 턴 수를 지정한다. 
- 저장 방식 : 입력과 `AI` 응답을 쌍으로 저장한다. 
- 슬라이딩 윈도우 : 새로운 대화가 추가되면 가장 오래된 대화가 메모리에서 제거된다. 
- 메모리 유지 : 항상 최근 `K` 개의 대화 턴만 유지한다. 

장점으로는 아래와 같은 것들이 있다. 

- 이전 대화의 일부만 유지하므로 토큰 소비가 줄어든다. 
- 가장 관련성 높은 최근 대화만 포함하여 현재 맥락에 집중한다. 
- 모델의 컨텍스트 창 크기 제한 내에서 작동하기 쉽다.
- 사용자가 수동으로 컨텍스트를 관리할 필요가 없다. 

단점으로는 아래와 같은 것들이 있다. 

- 윈도우 크기를 넘어서는 이전 대화는 완전히 손실된다.
- 윈도우 외부의 중요한 정보가 필요한 경우 문제가 발생할 수 있다. 
- 최적의 윈도우 크기를 결정하는 것이 어려울 수 있다. 


```python
from langchain.memory import ConversationBufferWindowMemory

memory = ConversationBufferWindowMemory(k=2, return_message=True)

memory.save_context(
    inputs = {
        "human" : "안녕하세요, 제품 A/S를 받고 싶습니다."
    },
    outputs = {
        "ai" : "안녕하세요! 어떤 문제가 발생했나요?"
    }
)
memory.save_context(
    inputs = {
        "human" : "제품이 작동하지 않습니다."
    },
    outputs = {
        "ai" : "어떤 제품인가요?"
    }
)

memory.save_context(
    inputs = {
        "human" : "스마트폰입니다."
    },
    outputs = {
        "ai" : "스마트폰 모델명을 알려주시겠어요?"
    }
)

memory.save_context(
    inputs={"human": "모델명은 XYZ123입니다."},
    outputs={
        "ai": "언제 구매하셨나요?"
    },
)

memory.save_context(
    inputs={"human": "6개월 전에 구매했습니다."},
    outputs={
        "ai": "보증 기간 내에 있으므로 무상 수리가 가능합니다. 가까운 서비스 센터를 방문해 주세요."
    },
)
memory.save_context(
    inputs={"human": "서비스 센터 위치를 알려주세요."},
    outputs={
        "ai": "고객님의 위치를 알려주시면 가장 가까운 서비스 센터를 안내해 드리겠습니다."
    },
)

memory.load_memory_variables({})['history']
# Human: 6개월 전에 구매했습니다.\nAI: 보증 기간 내에 있으므로 무상 수리가 가능합니다. 가까운 서비스 센터를 방문해 주세요.\nHuman: 서비스 센터 위치를 알려주세요.\nAI: 고객님의 위치를 알려주시면 가장 가까운 서비스 센터를 안내해 드리겠습니다.
```  


### ConversationTokenBufferMemory
`ConversationTokenBufferMemory` 는 대화 기록을 토큰 수를 기준으로 관리하는 메모리이다. 
`ConversationBufferMemory` 의 확장 버전으로 대화 턴 수가 아닌 토큰 수를 기준으로 대화를 제한한다. 
이는 마치 토큰 기반 윈도우처럼 직동한다. 
대화 기록의 토큰 수가 설정한 최대 토큰 수를 초과하면, 
가장 오래된 대화부터 제거하여 토큰 수를 제한 이내로 유지할 수 있다. 

- 토큰 제한 설정 : `max_token_limit` 매개변수를 통해 기억할 최대 토큰 수를 지정한다. 
- 저장 방식 : 사용자 입력과 `AI` 응답을  쌍으로 저장한다. 
- 토큰 계산 : 저장된 대화의 토큰 수를 계산한다. 
- 토큰 관리 : 대화 토큰 수가 제한을 초과하여 오래된 대화부터 제거한다. 
- 효율적 맥락 유지 : 토큰 기반으로 제한하여 모델의 컨텍스트 창을 효율적으로 사용한다.  

장점으로는 아래와 같은 것들이 있다.

- 토큰 수를 직접 제어하여 모델의 컨텍스트 창 사용을 최적화한다. 
- 대화 턴 수가 아닌 실제 토큰 수를 기준으로 메모리를 관리하여 더 세밀한 제어가 가능하다. 
- 토큰 수를 정확히 제한하여 `API` 비용을 효율적으로 관리할 수 있다. 
- 모델의 컨텍스트 창 크기에 맞게 정확히 메모리를 조절할 수 있다. 

단점으로는 아래와 같은 것들이 있다. 

- 토큰 계산을 위해 `LLM` 객체가 필요하므로 설정이 더 복잡하다. 
- 토큰 제한을 초과하는 과거 대화는 손실된다. 
- 토큰 계산을 위한 추가 처리가 필요하여 리소스 사용이 증가할 수 있다.  

```python
from langchain.memory import ConversationTokenBufferMemory

memory = ConversationTokenBufferMemory(
    llm = model,
    max_token_limit=300,
    return_messages=True
)

memory.save_context(
    inputs = {
        "human" : "안녕하세요, 제품 A/S를 받고 싶습니다."
    },
    outputs = {
        "ai" : "안녕하세요! 어떤 문제가 발생했나요?"
    }
)
memory.save_context(
    inputs = {
        "human" : "제품이 작동하지 않습니다."
    },
    outputs = {
        "ai" : "어떤 제품인가요?"
    }
)

memory.save_context(
    inputs = {
        "human" : "스마트폰입니다."
    },
    outputs = {
        "ai" : "스마트폰 모델명을 알려주시겠어요?"
    }
)

memory.save_context(
    inputs={"human": "모델명은 XYZ123입니다."},
    outputs={
        "ai": "언제 구매하셨나요?"
    },
)

memory.save_context(
    inputs={"human": "6개월 전에 구매했습니다."},
    outputs={
        "ai": "보증 기간 내에 있으므로 무상 수리가 가능합니다. 가까운 서비스 센터를 방문해 주세요."
    },
)
memory.save_context(
    inputs={"human": "서비스 센터 위치를 알려주세요."},
    outputs={
        "ai": "고객님의 위치를 알려주시면 가장 가까운 서비스 센터를 안내해 드리겠습니다."
    },
)

memory.load_memory_variables({})['history']
# [HumanMessage(content='6개월 전에 구매했습니다.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='보증 기간 내에 있으므로 무상 수리가 가능합니다. 가까운 서비스 센터를 방문해 주세요.', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='서비스 센터 위치를 알려주세요.', additional_kwargs={}, response_metadata={}),
# AIMessage(content='고객님의 위치를 알려주시면 가장 가까운 서비스 센터를 안내해 드리겠습니다.', additional_kwargs={}, response_metadata={})]
```  

### ConversationEntityMemory
`ConversationEntityMemory` 는 대화 중 언급된 특정 엔티티들을 추적하고 기억하는 메모리이다. 
일반적인 대화 메모리와 달리, 
대화에서 등장하는 중요한 개체나 주제를 식별하고 각각에 대한 정보를 별도로 저장한다. 
대화에서 나타나는 사람, 장소, 제품 등과 같은 엔티티를 자동으로 추출하고, 각 엔티티에 대한 정보를 지속적으로 업데이트한다. 
이는 마치 대화 중에 언급된 각 주제에 대한 별도의 메모장을 유지하는 것과 같다.  

- 엔티티 추출 : 대화에서 중요한 엔티티를 자동으로 식별한다. 
- 엔티티별 요약 저장 : 각 엔티티에 대한 정보를 별도로 저장하고 업데이트한다. 
- 컨텍스트 제공 : 측정 엔티티가 다시 언급될 때 해당 엔티티에 대한 저장된 정보를 프롬프트에 포함시킨다. 
- 요약 및 업데이트 : 엔티티에 대한 새로운 정보가 추가되면 요약을 업데이트한다.  

장점으로는 아래와 같은 것들이 있다.

- 특정 엔티티에 대한 정보를 누적하여 저장하므로 대화의 맥락을 더 깊게 이해할 수 있다. 
- 모든 대화를 저장하는 대신 중요한 엔티티와 정보만 추출하여 저장한다. 
- 사용자나 주제에 대한 정보를 기억하여 더 개인화된 응답을 제공할 수 있다. 
- 대화 내용을 엔티티별로 구조화하여 저장하므로 필요한 정보를 더 효율적으로 검색할 수 있다.  

단점으로는 아래와 같은 것들이 있다.

- 설정과 사용이 다른 메모리 유형보다 복잡하다.
- 엔티티 추출과 요약에 `LLM` 을 사용하므로 추가적인 `API` 호출이 필요하다. 
- 엔티티 추출 과정에서 오류가 발생할 수 있어 잘못된 정보가 저장될 수 있다. 
- 다수의 엔티티를 추적할 경우 메모리 사용량이 증가할 수 있다.  

```python
from langchain.chains import ConversationChain
from langchain.memory import ConversationEntityMemory
from langchain.memory.prompt import ENTITY_MEMORY_CONVERSATION_TEMPLATE

print(ENTITY_MEMORY_CONVERSATION_TEMPLATE.template)
# You are an assistant to a human, powered by a large language model trained by OpenAI.
# 
# You are designed to be able to assist with a wide range of tasks, from answering simple questions to providing in-depth explanations and discussions on a wide range of topics. As a language model, you are able to generate human-like text based on the input you receive, allowing you to engage in natural-sounding conversations and provide responses that are coherent and relevant to the topic at hand.
# 
# You are constantly learning and improving, and your capabilities are constantly evolving. You are able to process and understand large amounts of text, and can use this knowledge to provide accurate and informative responses to a wide range of questions. You have access to some personalized information provided by the human in the Context section below. Additionally, you are able to generate your own text based on the input you receive, allowing you to engage in discussions and provide explanations and descriptions on a wide range of topics.
# 
# Overall, you are a powerful tool that can help with a wide range of tasks and provide valuable insights and information on a wide range of topics. Whether the human needs help with a specific question or just wants to have a conversation about a particular topic, you are here to assist.
# 
# Context:
# {entities}
# 
# Current conversation:
# {history}
# Last line:
# Human: {input}
# You:


conversation = ConversationChain(
    llm=model,
    prompt=ENTITY_MEMORY_CONVERSATION_TEMPLATE,
    memory=ConversationEntityMemory(llm=model)
)

conversation.predict(input="철수와 영희는 한 회사에서 일하는 동료입니다.")
conversation.predict(input="철수는 부장이고, 영희는 사원입니다.")
conversation.predict(input="철수는 현재 새로운 프로젝트에 대한 고민이 많습니다.")
conversation.predict(input="영희는 오늘 점심 메뉴가 가장 큰 고민입니다.")
conversation.predict(input="철수는 오늘 퇴근 후 운동을 할 계획을 가지고 있습니다.")
conversation.predict(input="영희는 야근으로 모임에는 가지 못할 것 같습니다.")

conversation.memory.entity_store
# InMemoryEntityStore(store={'철수': '철수는 회사에서 부장으로 일하고 있으며, 오늘 퇴근 후 운동을 할 계획을 가지고 있습니다.', '영희': '영희는 사원이며 야근으로 모임에 가지 못할 예정입니다.'})

conversation.predict(input="영희의 고민은 무엇인가요?")
# 영희의 고민은 오늘 점심 메뉴가 가장 큰 고민인 것으로 보입니다. 그러나 영희 사원은 사원이며 야근으로 모임에 가지 못할 예정입니다. 이는 영희 사원이 업무에 대한 부담을 느끼고 있고, 오늘의 점심 메뉴를 결정하는 것 외에도 업무적인 문제들에 대한 고민을 하고 있을 가능성이 있습니다. 영희 사원은 중요한 업무를 처리해야 하기 때문에 야근을 하게 된 모양입니다. 이러한 상황에서 영희 사원의 고민은 업무적인 문제들에 대한 스트레스와 야근으로 인한 피로, 그리고 점심 메뉴를 결정하는 것 등 다양한 요인으로 인해 복잡할 수 있습니다.
```  



### ConversationKGMemory
`ConversationKGMemory` 는 대화 내용을 지식 그래프(`Knowledge Graph`) 형태로 저장하는 메모리이다. 
이 메모리는 대화에서 언급된 개체들 간의 관계를 추출하여 구조화된 형태로 기억한다. 
대화 내용에서 트리플(`Triple`) 형태의 정보를 추출한다. 트래플은 주어, 서술어, 목적어 형태로 구성되며, 
이를 통해 대화에서 언급된 개체들과 그 관계를 명확하게 표현할 수 있다. 

- 트리플 추출 : 대화 내용에서 주어, 서술어, 목적어 형태의 관계를 추출한다. 
- 지식 그래프 추출 : 추출된 트리플을 바탕으로 지식 그래프를 구성한다. 
- 관련 정보 검색 : 새로운 대화에서 특정 개체가 언급되면, 그래프에서 관련된 정보를 검색한다. 
- 맥락 강화 : 검색된 정보를 바탕으로 응답 생성 시 맥락을 강화한다.  


장점으로는 아래와 같은 것들이 있다.

- 단순한 텍스트가 아닌 구조화된 형태로 정보를 저장하여 관계를 명확히 표현한다. 
- 대화에서 언급된 개체들 간의 관계를 중점적으로 기억한다. 
- 특정 개체에 관련된 정보만 빠르게 검색할 수 있다. 
- 관계 기반 지식을 통해 더 풍부한 맥락을 제공한다. 
- 전체 대화를 저장하는 대신 중요한 관계만 추출하여 저장하므로 토큰 사용이 효율적이다. 

단점으로는 아래와 같은 것들이 있다.

- 모든 대화 내용을 저장하는 것이 아니라 관계만 추출하므로 일부 정보가 손실될 수 있다. 
- 트리플 추출 과정에서 오류가 발생할 수 있어 관계가 정확하게 표현되지 않을 수 있다. 
- 다른 메모리 유형에 비해 설정과 관리가 복잡하다. 
- 트리플 추출에 `LLM` 을 사용하므로 추가적인 `API` 호출이 필요하다.  


```python
from langchain.memory import ConversationKGMemory

memory = ConversationKGMemory(llm=model, return_messages=True)
memory.save_context(
    {"input" : "영희님 반가워요. 오늘부터 함께 업무하게 되는 철수 부장이라고 합니다."},
    {"output" : "철수님 반가워요. 잘 부탁드리겠습니다. 고민이 많아 보이시는데 어떤 고민이 있으신가요 ?"}
)
memory.save_context(
    {"input" : "다음 달부터 시작하는 신규 프로젝트에 대해서 고민이 있습니다. 영희님은 어떠신가요 ?"},
    {"output" : "전 오늘 점심이 가장 큰 고민이네요 하하하 맛있는 걸로 한번 골라보겠습니다."}
)
memory.load_memory_variables({"input" : "철수는 누구입니까?"})
# {'history': [SystemMessage(content='On 철수: 철수 는 부장. 철수 는 영희님과 함께 업무하게 됩니다.', additional_kwargs={}, response_metadata={})]}


# ConversationChain 과 함께 활용

from langchain.prompts.prompt import PromptTemplate

template = """The following is a friendly conversation between a human and an AI. 
The AI is talkative and provides lots of specific details from its context. 
If the AI does not know the answer to a question, it truthfully says it does not know. 
The AI ONLY uses information contained in the "Relevant Information" section and does not hallucinate.

Relevant Information:

{history}

Conversation:
Human: {input}
AI:"""

prompt = PromptTemplate(
    input_variables=["history", "input"],
    template=template
)

conversation_kg = ConversationChain(
    llm=model,
    memory=ConversationKGMemory(llm=model)
)
conversation_kg.predict(input="철수와 영희는 한 회사에서 일하는 동료입니다.")
conversation_kg.predict(input="철수는 부장이고, 영희는 사원입니다.")
conversation_kg.predict(input="철수는 현재 새로운 프로젝트에 대한 고민이 많습니다.")
conversation_kg.predict(input="영희는 오늘 점심 메뉴가 가장 큰 고민입니다.")
conversation_kg.predict(input="철수는 오늘 퇴근 후 운동을 할 계획을 가지고 있습니다.")
conversation_kg.predict(input="영희는 야근으로 모임에는 가지 못할 것 같습니다.")

conversation_kg.memory.load_memory_variables({"input" : "철수는 누구입니까?"})
# {'history': 'On 철수: 철수 는 한 회사에서 일하는 동료. 철수 is a 부장.'}

conversation_kg.predict(input="철수는 누구입니까?")
# 철수는 한 회사에서 일하는 동료입니다. 그는 현재 부장으로서 중요한 역할을 맡고 있습니다. 철수는 매우 친절하고 능력 있는 사람으로, 그의 업무에 대한 열정과 전문성을 항상 보여줍니다. 그는 회사 내에서 중요한 프로젝트를 맡고 있으며, 그의 팀과 함께优秀한 성과를 내고 있습니다. 철수와 함께 일하는 것은 매우 즐겁고 배우는 기회가 많은 것 같습니다.
```  

