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
