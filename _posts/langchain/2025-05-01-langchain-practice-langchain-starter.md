--- 
layout: single
classes: wide
title: "[LangChain] LangChain 환경 구성 및 기본 사용법"
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
    - LangSmith
    - LCEL
toc: true
use_math: true
---  

## Langchain
[Langchain Intro]({{site.baseurl}}{% link _posts/langchain/2025-03-09-langchain-practice-langchain-intro.md %})
에서 알아본 `LangChain` 은 `LLM(Large Language Model)` 을 활용한 애플리케이션 개발을 쉽게 할 수 있도록 도와주는 프레임워크이다. 
주요 2가지 기능 중 문맥을 인식하는 기능은 언어 모델을 다양한 문맥 소스와 연결해 프롬프트 지시사항, 예제, 응답 근거 내용이 포함된다. 
이를 통해 언어 모델은 제공된 정보를 바탕으로 정확도와 관련성 높은 답변을 생성할 수 있다. 

다른 하나는 추론하는 기능으로 주어진 문맥을 바탕으로 어떠한 답변을 제공하거나, 필요한 조치가 무엇인지 스스로 추론할 수 있다. 
언어 모델이 정보를 재생산하는 것을 넘어, 주어진 상황을 분석하고 적절한 해결책을 제시할 수 있다는 의미이다.  

예제에 필요한 필수 패키지는 아래와 같다.  

```text
# requirements.txt
langchain
langchain-core
langchain-groq
```  

기본 사용법에 대해 알아보면서 몇가지 `LahgChain` 의 구성요소의 개념과 사용법을 다루지만, 해당 포스팅에서 깊게 다루지는 않는다. 
이후 각 요소를 더 깊게 다루는 포스팅을 작성할 예정이다.  
