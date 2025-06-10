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
