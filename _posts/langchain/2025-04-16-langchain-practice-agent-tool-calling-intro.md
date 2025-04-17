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

