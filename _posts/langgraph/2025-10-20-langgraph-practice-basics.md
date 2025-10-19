--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Basics"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - Multi-Agent
    - LLM
    - AI
    - Graph
    - AI Framework
    - TypedDict
    - Annotated
    - add_messages
    - StateGraph
    - Node
    - Edge
toc: true
use_math: true
---  


## LangGraph Basics
`LangGraph` 사용에 앞서 필요한 기본 지식을 설명하고, 
이를 바탕으로 간단한 `Chatbot` 을 구현해 본다.  

### TypedDict
`TypedDict` 는 `LangGraph` 에서 사용하는 `State` 의 타입을 명확하게 생성하는 
`Python` 의 타입 힌트 도구이다. 
이는 `Python` 의 `typing` 모듈에서 제공하는 기능으로, 
딕셔너리의 키와 값을 명확하게 지정하는 데 사용된다.
이렇게 상태의 구조와 타입을 명확하게 정의하여, 각 `Node` 에서 주고받는 데이터의 일관성을 보장한다. 

`TypedDict` 는 `dict` 와 유사하지만 다음과 같은 차이점이 있다. 

- 타입 검사 : 정적 타입 검사를 제공한다. 
- 키와 값 타입 : 각 키에 대해 구체적인 타입을 지정할 수 있다. 
- 유연성 : 정의된 구조를 따라야 한다. 정의되지 않는 키는 오류가 발생한다. 

아래는 `TypedDict` 의 예시이다. 

```python
from typing import Dict, TypedDict

class ExampleTypedDict(TypedDict):
    name : str
    age : int


example_typed_dict : ExampleTypedDict = {
    "name" : "Jack",
    "age" : 30,
}


```  

`TypedDict` 을 그냥 사용하면 일반적인 `Dict` 와 큰 차이가 없다. 
그 이유는 `TypedDict` 는 정적 타입 힌트를 제공하는 도구일 뿐,
런타임에서 타입을 실제로 검사하거나 강제하지는 않기 때문이다. 
정적 타입검사를 사용하고 싶은 경우 `mypy` 혹은 `pyright` 와 같은 도구를 사용하면 타입 오류를 탐지해 낼 수 있다.  
