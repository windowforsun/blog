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
