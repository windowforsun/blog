--- 
layout: single
classes: wide
title: "[LangChain] LangChain Prompt"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain 에서 Prompt 를 사용해 언어 모델에 대한 입력을 구조화하는 방법에 대해 알아보자'
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

## LangChain Expression Language
`LCEL` 은 `LangChain` 프레임워크 내에서 체인, 프롬프트, 모델의 연결과 조작을 더 쉽고 강력하게 만들어주는 도구적 언어이다. 
`LCEL` 에서 제공하는 표현식을 이용해 다양한 `LLM` 워크플로우를 선언적이고 직관적으로 구성할 수 있게 한다. 

주요 특징은 아래와 같다. 

- 간결함 : 복잡한 체인 구조를 간단한 문법으로 표현
- 표현력 : 다양한 체인 조합, 브랜칭, 반복, 동시 실행 등 쉽게 구현
- 유지보수성 : 파이프라인을 선언적으로 작성하여 코드를 더 읽기 쉽고 관리하기 쉽게 만듦
- 직관적 연결 : 다양한 컴포넌트(프롬프트, 모델, 출력 파서 등)를 함수처럼 연결(`|`)하여 구성

`LCEL` 의 기본적인 구성 요소는 다음과 같다. 

- `Runnable` : `LCEL` 의 모든 구성 요소의 기반이 되는 인터페이스이다. 
- `Chain` : 여러 `Runnable` 을 순차적으로 연결한 실행 파이프라인의 상위 개념이다. 
- `RunnableMap` : 여러 `Runnable` 을 병렬로 실행하여 여러 개의 결과를 동시에 반환한다. 
- `RunnableSequence` : 여러 `Runnable` 을 순차적으로 연결하는 객체이다. 
- `RunnableLambda` : 파이프라인 내에서 `Python` 함수 등 임의의 함수를 래핑하여 사용할 수 있도록 한다. 

`LCEL` 의 추가적인 구현체로는 다음과 같은 것들이 있다. 

- `RunnableBranch` : 입력 조건에 따라 여러 `Runnable` 중 하나를 선택하여 실행한다. 
- `RunnablePassthrough` : 입력을 그대로 다음 단계로 넘기는 역할을 한다. (특별한 처리 없이 전달)
- `RunnableParallel` : `RunaableMap` 과 유사하게 여러 `Runnable` 을 병렬로 실행하여 결과를 반환한다.
- `RunnableEach` : 리스트 형태의 입력에 대해 각 요소마다 동일한 `Runnable` 을 실행한다. 
- `RunnableRetry` : 실패 시 재시도 로직을 포함한 래퍼이다. 
- `RunnableWithFallbacks` : 실패 시 대체 `Runnable` 을 차례로 시도한다.  

사용 가능한 `runnables` 의 전체 목록은 [여기](https://python.langchain.com/api_reference/core/runnables.html)
에서 확인 가능하다.  

`LCEL` 을 사용하면 `프롬프트 -> LLM -> 파서` 와 같이 다양한 컴포넌트를 직렬로 간편하게 연결해 구성할 수 있다. 
모든 체인을 연결하는 방식을 사용하기 때문에 복잡한 처리도 `LCEL` 을 통해 간단하게 구현할 수 있다. 
간단한 사용 예시는 다음과 같다. 


```python
from langchain.chat_models import init_chat_model
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser

os.environ["GROQ_API_KEY"] = "api key"
model = model = init_chat_model("llama-3.3-70b-versatile", model_provider="groq")
prompt = ChatPromptTemplate.from_template("다음 질의 혹은 주제에 대해 설명해, 답변은 최소 30자 최대100자로: {query}")
output_parser = StrOutputParser()

chain = prompt | model | output_parser

result = chain.invoke({"query": "LangChain"})
# LangChain은 AI와 언어 모델을 위한 오픈 소스 플랫폼으로, 개발자들이 자신의 언어 모델을 쉽게 구축하고 배포할 수 있도록 지원합니다.
```  

`LCEL` 의 장점은 다음과 같다. 

- 가독성 : 복잡한 체인도 파이프 연산자로 직관적으로 표현
- 재사용성 : 각 컴포넌트(프롬프트, 모델 등)는 독립적으로 재사용 가능
- 유연성 : 조건 분기, 반복, 병렬 등 다양한 워크플로우 유연하게 구현
- 디버깅 용이 : 각 단계별로 결과를 쉽게 확인할 수 있음

### RunnableMap
`RunnableMap` 은 여러 개의 `Runnable` 을 병렬로 실행하여 각기 다른 결과를 딕셔너리 형태로 반환하는 역할을 한다. 
하나의 입력을 받아 여러 개 처리(분석, 생성 등)를 동시에 실행하고, 
각 다른 결과를 `key-value` 쌍으로 모아 최종적으로는 딕셔너리 형태로 반환한다.  

`RunnableMap` 은 아래와 같은 경우 사용할 수 있다.

- 동시에 여러 작업을 수행하고 싶을 때
- 중복 실행을 피하면서, 다양한 분석/생성을 병렬 처리하고 싶을 때
- 여러 `LLM` 체인 결과를 한 번에 모아서 관리하고 싶을 때 

아래는 자용자의 질의를 요약하는 결과와 감정을 분석하는 처리를 모두 수행하는 `RunnableMap` 의 예시이다. 

```python
from langchain_core.runnables import RunnableMap
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

summary_template = """당신은 요약 전문가 입니다. 사용자 질의를 10자 이내로 요약하세요. 

# 질의
{question}
"""
summary_prompt = ChatPromptTemplate.from_template(summary_template)

summary_chain = summary_prompt | model | StrOutputParser()

setiment_template = """당신은 감정분석 전문가 입니다. 사용자 질의를 긍정, 부정으로 분류하세요.

# 질의
{question}

# 답변은 반드시 긍정 혹은 부정으로만 답해야 합니다.
"""
sentiment_prompt = ChatPromptTemplate.from_template(setiment_template)

sentiment_chain = sentiment_prompt | model | StrOutputParser()

multi_chain = RunnableMap({
    "요약" : summary_chain,
    "감정분석" : sentiment_chain
})

result = multi_chain.invoke({"question" : "오늘 아침에 일어나 사과를 먹었는데 맛이 너무 좋아 친구들에게 알려줬어요."})
# {'요약': '사과 먹고 친구한테 말함', '감정분석': '긍정'}
```  


### RunnableSequence
`RunnableSequence` 는 여러 개의 `Runnable` 을 순차적으로 연결하여 실행하는 체인 역할을 한다. 
입력을 받아 첫 번째 `Runnable` 에 전달한 뒤 그 결과를 두 번째 `Runnable` 에 전달하고 마지막 `Runnable` 을 통해 최종 결과를 만드는 
파이프라인 또는 직렬 체인구조를 구현할 수 있다. 

`RunnableSequence` 는 아래와 같은 경우 사용할 수 있다.

- 여러 단계를 거쳐 데이터를 처리하고 싶을 때
- 단계별로 서로 다른 처리가 필요한 복합 워크플로우를 설계할 때
- 체인의 각 단계를 재사용하고 싶을 때 

아래는 사용자의 질의를 요약하고 그 결과를 영어로 번역하는 `RunnableSequence` 의 예제이다. 

```python
from langchain_core.runnables import RunnableSequence
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

summary_template = """당신은 요약 전문가 입니다. 사용자 질의를 10자 이내로 요약하세요.

# 질의
{question}
"""
summary_prompt = ChatPromptTemplate.from_template(summary_template)

summary_chain = summary_prompt | model | StrOutputParser()

translate_template = """한글을 영어로 번역하는 전문가입니다. 사용자의 질의를 영어로 번역하세요.

# 질의
{question}

"""
translate_prompt = ChatPromptTemplate.from_template(translate_template)

translate_chain = translate_prompt | model | StrOutputParser()

sequence_chain = RunnableSequence(
    first=summary_chain,
    last=translate_chain
)
# or equal
# sequence_chain = summary_chain | translate_chain

result = sequence_chain.invoke({"question" : "오늘 아침에 일어나 사과를 먹었는데 맛이 너무 좋아 친구들에게 알려줬어요."})
# I told my friend that the apple was delicious.
```


### RunnableSequence
`RunnableSequence` 는 여러 개의 `Runnable` 을 순차적으로 연결하여 실행하는 체인 역할을 한다.
입력을 받아 첫 번째 `Runnable` 에 전달한 뒤 그 결과를 두 번째 `Runnable` 에 전달하고 마지막 `Runnable` 을 통해 최종 결과를 만드는
파이프라인 또는 직렬 체인구조를 구현할 수 있다.

`RunnableSequence` 는 아래와 같은 경우 사용할 수 있다.

- 여러 단계를 거쳐 데이터를 처리하고 싶을 때
- 단계별로 서로 다른 처리가 필요한 복합 워크플로우를 설계할 때
- 체인의 각 단계를 재사용하고 싶을 때

아래는 사용자의 질의를 요약하고 그 결과를 영어로 번역하는 `RunnableSequence` 의 예제이다.

```python
from langchain_core.runnables import RunnableSequence
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

summary_template = """당신은 요약 전문가 입니다. 사용자 질의를 10자 이내로 요약하세요.

# 질의
{question}
"""
summary_prompt = ChatPromptTemplate.from_template(summary_template)

summary_chain = summary_prompt | model | StrOutputParser()

translate_template = """한글을 영어로 번역하는 전문가입니다. 사용자의 질의를 영어로 번역하세요.

# 질의
{question}

"""
translate_prompt = ChatPromptTemplate.from_template(translate_template)

translate_chain = translate_prompt | model | StrOutputParser()

sequence_chain = RunnableSequence(
    first=summary_chain,
    last=translate_chain
)
# or equal
# sequence_chain = summary_chain | translate_chain

result = sequence_chain.invoke({"question" : "오늘 아침에 일어나 사과를 먹었는데 맛이 너무 좋아 친구들에게 알려줬어요."})
# I told my friend that the apple was delicious.
```

