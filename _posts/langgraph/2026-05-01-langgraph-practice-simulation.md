--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Simulation"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: 'LangGraph 를 바탕으로 Agent/Assistant 를 만들고 이를 반복 테스트하고 개선점을 파악하는 시뮬레이션에 대해 알아보자.'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - Simulation
toc: true
use_math: true
---  


## LangGraph Agent Simulation
구현된 `Agent/Assistant` 가 정상적으로 동작하는지, 적절한 답변을 제공하는지 품질에 대한 경가를 할 때, 
실제 사용자화의 대화를 일일이 수동으로 테스트하면 시가니 많이 소요되고 평가가 어렵다. 
용이한 테스트를 위해서는 `재현 가능성` 을 통해 동일한 시나리오와 조건에서 반복적으로 테스트할 수 있어야 개선점을 명확히 파악할 수 있기 때문이다. 
그리고 `자동화` 를 통해 대화 시뮬레이션을 바탕으로 다양한 응답 케이스를 자동으로 생성/평가/수집할 수 있어야 한다.  

`LangGraph` 를 활용하면 이러한 자동화된 평가를 위한 시뮬레이션을 비교적 쉽게 구현할 수 있다. 
각 노드는 이에전트, 사용자 도구, 등 특정 역할을 담당하고, 그래프 형태의 설계로 흐름에 따라 대화가 진행 되도록 구성할 수 있기 때문이다.  

이러한 평가를 위한 시뮬레이션은 다음과 같은 활용을 할 수 있다.  

- 테스트 자동화 : 다양한 시나이로를 미리 정의해 자동으로 테스트하고 코드 변경 시마다 대화 시뮬레이션을 바탕으로 품질 확보와 반복적인 개선이 가능하다. 
- 성능 평가 : 응답의 정확도, 친절함, 문제 해결 능력 등을 자동으로 점수화 가능하다. 
- 반복 개선 : 테스트 결과를 바탕으로 에이전트의 답변 패턴, 행동 등을 지속적으로 개선할 수 있다. 
- 실 서비스 적용 전 검증 : 실제 서비스에 적용하기 전에 다양한 시나리오를 통해 에이전트의 동작을 사전 검증할 수 있다.  
- A/B 테스트 : 서로 다른 에이전트 버전을 시뮬레이션해 비교하고 최적의 버전을 선택할 수 있다. 
- 실시간 모니터링 : 실제 서비스 적용 이후에도 지소적으로 수집된 시나리오를 바탕으로 시뮬레이션을 돌려 품질 유지 및 이상 탐지를 할 수 있다. 

이러한 평가 시뮬레이션을 위해 `LangGraph` 를 활용해 간단한 `AI Assistant(상담사)` 를 구현하고, 
이를 가상 사용자를 통해 시뮬레이션하는 예제를 살펴본다.  


### Graph State
가장 먼저 구현할 그래프에서 사용할 상태를 정의한다.  

```python
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict


class AgentState(TypedDict):
  messages: Annotated[list, add_messages]
```  
