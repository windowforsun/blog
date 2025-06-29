--- 
layout: single
classes: wide
title: "[LangChain] LangChain Prompt"
header:
  overlay_image: /img/langchain-bg-2.jpg
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


## Text Splitter
`Text Splitter` 는 큰 텍스트 문서를 언어 모델이 처리할 수 있는 작은 단위(`chunk`)로 나누는 도구이다. 
대부분 `LLM` 은 입력 토큰 수 제한이 있어 큰 문서를 그대로 처리할 수 없기 때문에, 
세분화함으로써 질문에 연관성이 있는 정보만 가져오는 데 도움이 된다. 
각각의 단위는 특정 주제나 내용에 초점을 맞추므로, 관련성이 높은 정보를 제공할 수 있게 된다. 
그리고 전체 문서를 `LLM` 으로 입력하면 그 만큼 비용이 발생하고, 효율적인 답변을 얻기 어려울 수 있다. 
이러한 문제는 할루시네이션으로 이어질 수 있기 때문에 정보량을 줄이면서 질문에 필요한 정보만 발췌해 이를 개선할 수 있다.  

`Test Splitter` 는 다양한 형식으로 작성된 문서에서 구조를 파악한다. 
이는 문서의 헤더, 본문, 푸터, 페이지 번호, 섹션 제목 등이 될 수 있다. 
그리고 문서를 어떤 단위로 나눌지 결정한다. 
이는 문서의 내용과 목적에 따라 페이지, 섹션, 문단 등 다양하게 결정될 수 있다. 
하나의 문서를 몇 개의 토큰 단위로 나눌지 결정해야 하고, 
각 분할마다 맥락이 이어질 수 있도록 얼만큼을 겹치도록 할지도 결정해야 한다.  

공통적으로 `Text Splitter` 는 아래와 같은 주요 매개변수가 있다.  

- `chunk_size` : 각 청크의 최대 크기(문자 또는 토큰 수)
- `chunk_overlap` : 연속돤 청크 간 중복되는 부분의 크기
- `sparators` : 텍스트 분할에 사용할 구분자 목록
- `length_function` : 청크 크기를 측정하는 함수(문자 수 또는 토큰 수)

`Text Splitter` 의 종류는 [여기](https://python.langchain.com/api_reference/text_splitters/index.html)
에서 확인할 수 있다. 

필요에 따라 아래 라이브러리 설치가 필요할 수 있다. 

```bash
$ pip install -qU langchain-text-splitters
```
