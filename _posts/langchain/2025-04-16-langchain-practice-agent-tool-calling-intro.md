--- 
layout: single
classes: wide
title: "[LangChain] LangChain Introduction"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
toc: true
use_math: true
---  

## LangChain Agent
`LangChain Agent` 는 `LLM` 기반 애플리케이션에서 중요한 역할을 하는 컴포넌트로, 
`AI` 시스템이 더욱 자율적이고 목표 지향적으로 작업을 수행할 수 있게 해주는 컴포넌트이다. 
`Agent` 는 주어진 목표를 달성하기 위해 상호작용하며 의사 결정을 내리고 해동을 취하는 지능형 시스템이다.  

`LLM` 이 다양한 `Tool` 을 활용해 동적인 의사결정을 할 수 있도록 구성된 컴포넌트를 의미한다.
사용자의 입력에 따라 적절한 도구를 선택하고 실행하여 최적의 답변을 생성하는 방식으로 동작하게 된다.  

### Agent Type
`Agent` 는 아래와 같은 다양한 유형을 목적에 맞게 사용할 수 있다. 

- `ZERO_SHOT_REACT_DESCRIPTION` : 사전 학습 없이 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 간단한 질문에 대해 빠르게 답변해야 하는 경우 사용하기 좋다. 
- `REACT_DOCSTORE` : 문서 저장소에 접근하여 관련 정보를 검색하고 질문에 답하는 에이전트이다. 대량의 문서나 데이터베이스에서 종보를 검색해야 하는 경우 사용하기 좋다. 
- `SELF_ASK_WITH_SEARCH` : 복잡한 질문을 더 간단한 질문으로 나누고, 검색 도구를 사용해 답을 찾는 에이전트이다. 복잡한 문제를 단계별로 해결해야 하는 경우 사용하기 좋다. 
- `CONVERSATIONAL_REACT_DESCRIPTION` : 대화형으로 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 사용자와의 대화에서 자연스럽게 정보를 제공해야 하는 경우 사용하기 좋다. 
- `CHAT_ZERO_SHOT_REACT_DESCRIPTION` : 대화형 모델에 최적화된 사전 학습 없이 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 대화형 AI 챗봇 구현에 사용하기 좋다. 
- `CHAT_CONVERSATIONAL_REACT_DESCRIPTION` : 대화형으로 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 사용자와의 대화에서 자연스럽게 정보를 제공해야 하는 경우 사용하기 좋다. 
- `STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION` : 여러 입력을 가진 도구를 호출할 수 있는 대화형 모델에 최적화된 에이전트이다. 복잡한 입력을 처리해야 하는 대화형 AI 시스템에 사용하기 좋다. 
- `OPENAI_FUNCTIONS` : `OpenAI` 함수 사용에 최적화된 에이전트이다. `OpenAI` 의 다양한 기능을 화룡ㅇ해야 하는 경우 사용하기 좋다. 
- `OPENAI_MULTI_FUNCTIONS` : 여러 `OpenAI` 함수를 사용하는 에이전트이다. 여러 `OpenAI` 기능을 동시에 활용해야 하는 경우 사용하기 좋다. 


## LangChain Tool calling
`LangChain Tool calling` 은 `LLM` 이 특정 기능을 실행하도록 설게된 도구를 호출하는 방식을 의미한다. 
사용자가 입력한 질문에 따라 `LLM` 이 적절한 `Tool` 을 선택하고 실행해 최적의 결과를 제공한다. 
`LLM` 이 단순히 답변을 생성하는 것이 아니라 `API` 호출, 데이터베이스, 계산기 등 활용 가능한 다양한 `Tool` 을 직접 실행해 결과를 생성하는 것을 의미한다.  

`Tool Calling` 을 활용하면 아래와 같은 장점을 얻을 수 있다. 

- 기능 확장 : `LLM` 의 능력을 크게 확장할 수 있다. 모델이 자체적으로 가지고 있지 않은 정보나 기능에 접근하도록 할 수 있다. 
- 실시간 정보 : 웹 검색 등을 통해 모델이 학습하지 못한 최신 정보를 얻을 수 있어, 답변의 최신성을 제공할 수 있다. 
- 적확성 향상 : 계산기, 전문성 있는 정보를 직접 모델에 제공해 정확한 수치나 정보의 신뢰성을 높일 수 있다. 
- 다양한 작업 : 코드 실행, 파일 조장, API 호출 등 작업의 범위를 확장할 수 있다. 

[Tools](https://python.langchain.com/docs/integrations/tools/)
를 보면 `LangChain` 에서 기본으로 제공하는 `built-in tools/toolkit` 를 확인할 수 있다.  


## Agent Demo
다음은 `OpenWeather API` 를 사용해 날씨를 조회하고, 
그 결과를 바탕으로 날씨를 요약한느 `Agent` 구현 예시이다.
`LLM` 모델로는 `groq` 를 사용해 `llama-3.3-70b-versatile` 모델을 사용한다.  

### 환경 설정
본 예제를 구현 및 실행하기 위해 필요한 파이썬 패키지는 아래와 같다.

```text
# requirements.txt

langchain==0.3.20
langchain-core==0.3.20
langchain-community==0.3.20
langchain-groq
requests
```  

코드 실행에 필요한 전체 `import` 구문은 아래와 같다.  

```python
import os
import getpass
import requests
from langchain_core.tools import tool
from langchain.chat_models import init_chat_model
from langchain.agents import initialize_agent, AgentType
from typing import Annotated, List
from datetime import datetime, timedelta
```  
