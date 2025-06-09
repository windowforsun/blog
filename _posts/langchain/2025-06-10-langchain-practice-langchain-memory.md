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
