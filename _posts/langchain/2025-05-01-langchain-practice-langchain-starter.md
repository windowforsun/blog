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


### LangSmith
`LangSmith` 는 `LangChain` 의 핵심 구성 요소 중 하나로, `LLM` 을 활용한 대화형 AI 개발을 지원한다. 
이는 `LLM` 애플리케이션 개발, 모니터링 및 테스트를 위한 플랫폼으로 프로젝트나 `LangChain` 학습에 도움이 된다. 

이후에 더 자세히 `LangSmith` 에 대해 알아보겠지만 주요한 기능중 하나는 추적기능이다. 
이는 `LLM` 애플리케이션의 동작을 이해하기 위한 중요한 기능으로, `LangSmith` 는 `LangChain` 사용 여부와 관계없이 아래와 같은 추적 기능을 제공한다. 

- 예상치 못한 결과
- 에이전트 루핑
- 체인의 성능 문제
- 에이전트 스텝 별 사용한 토큰 수

사용을 위해서는 [LangSmith](https://smith.langchain.com/)
에 접속해 `API Key` 를 발급한다. 
자세한 가이드는 [여기](https://docs.smith.langchain.com/administration/how_to_guides/organization_management/create_account_api_key)
를 참고한다.  

발급 받은 `API Key` 는 `.env` 환경 변수 파일을 사용하거나 직접 코드에서 환경 변수에 등록해 추적을 활성화 할 수 있다.  

```python
import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_ENDPOINT"] = "https://api.smith.langchain.com"
os.environ["LANGCHAIN_PROJECT"] = "lagnchain-starter"
os.environ["LANGCHAIN_API_KEY"] = "{api_key}"
```  

이후 `LangChain` 을 바탕으로 `LLM` 을 활용하면 `LangSmith` 에서 `langchain-starer` 라는 이름의 프로젝트에서 추적 내용을 확인할 수 있다. 


### LCEL
`LCEL(LangChain Execution Layer)` 은 `LangChain` 의 핵심 구성 요소 중 하나로, `LLM` 애플리케이션을 더 효율적으로 구축하기 위한 표현식 레이어이다. 

주요 특징은 아래와 같다. 
- 파이프라인 구성 : 파이프 연산자(`|`) 를 사용해 여러 컴포넌트를 연결하여 복잡한 워크플로우를 간결하게 표현할 수 있다. 
- 스트리밍 지원 : 데이터가 파이프라인을 통과하면서 실시간으로 처리되는 스트리밍 처리를 지원한다. 
- 병렬 처리 : 여러 작업을 동시에 실행하여 성능을 최적화할 수 있다. 
- 재사용성 : 구성된 재안을 쉽게 재사용하고 결합할 수있다.  


#### PromptTemplate
`PromptTemplate` 을 사용하면 사용자의 입력 변수를 템플릿에 삽입해 `LLM` 에 전달할 수 있다. 

```python
from langchain_core.prompts import PromptTemplate

template = "{item}에 대해 한글로 설명해 주세요."

prompt_template = PromptTemplate.from_template(template)
# PromptTemplate(input_variables=['item'], input_types={}, partial_variables={}, template='{item}에 대해 설명해 주세요.')

prompt = prompt_template.format(item="AI")
# AI에 대해 설명해 주세요.
```  

#### Chain
`LCEL` 을 사용해 다양한 구성 요소를 단일 체인으로 결합할 수 있다. 
체인 구성의 기호는 `|` 를 사용해 서로 다른 구성 요소를 연결하고 한 구성 요소의 출력을 다음 구성 요소의 입력으로 전달한다. 

```python
model = init_chat_model("llama3-8b-8192", model_provider="groq")
chain = prompt_template | model

# PromptTemplate(input_variables=['item'], input_types={}, partial_variables={}, template='{item}에 대해 설명해 주세요.')
# | ChatGroq(client=<groq.resources.chat.completions.Completions object at 0x797940ba75d0>, async_client=<groq.resources.chat.completions.AsyncCompletions object at 0x797940bf1b10>, model_name='llama3-8b-8192', model_kwargs={}, groq_api_key=SecretStr('**********'))
```  

체인 실행은 다양한 방법이 있는데 그 중 하나는 `invoke()` 메서드를 사용하는 것이다.
`invoke()` 메서드는 체인을 실행하고 결과를 반환하는데, 딕셔너리 형태로 입력값을 전달할 수 있다. 


```python
input = {"item" : "반도체"}

chain.invoke(input)
# AIMessage(content='한글로 반도체에 대해 설명드리겠습니다.\n\n**반도체(.semiconductor)**\n\n반도체는 전기적 성질이 반도체인 물질을 의미합니다. 일반적으로는 실리콘(Silicon)이나 게르마늄(Germanium) 등과 같은 물질을 사용합니다. 이 물질들은 전기적 성질이 반도체이므로, 전류를 쉽게 теч을 수 있습니다. 그러나 전류를 가질 때는 전자들이 전기장을 따라 움직여 전류를 생성합니다.\n\n**반도체의 특징**\n\n1. **전기적 성질**: 반도체는 전기적 성질이 반도체이므로, 전류를 쉽게 теч을 수 있습니다.\n2. **전자 이동**: 반도체에 전압을 가하면 전자들이 전기장을 따라 움직여 전류를 생성합니다.\n3. **도핑**: 반도체를 도핑(Doping)하면 전자들이 더 많은 것을 얻을 수 있습니다.\n\n**반도체의 종류**\n\n1. **P-형 반도체(P-Type Semiconductor)**: P-형 반도체는 도핑에 의해 전자가 부족한 반도체입니다.\n2. **N-형 반도체(N-Type Semiconductor)**: N-형 반도체는 도핑에 의해 전자가 더 많은 반도체입니다.\n3. **I-형 반도체(I-Type Semiconductor)**: I-형 반도체는 도핑 없이 전자의 수가 균일한 반도체입니다.\n\n**반도체의 사용**\n\n1. **전자 장치**: 반도체는 전자 장치, 특히 트랜지스터(Transistor)와 집적회로(Integrated Circuit)를 생산하는 데 사용됩니다.\n2. **컴퓨터**: 반도체는 컴퓨터의 중앙 처리 장치(CPU), 메모리, 저장 장치 등에서 사용됩니다.\n3. **전자 기기**: 반도체는 스마트폰, TV, 컴퓨터 등 다양한 전자 기기에서 사용됩니다.\n\n이러한 반도체의 특징, 종류, 사용 등을 설명했습니다. 반도체의 세계는 지속적으로 발전하고 있습니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 475, 'prompt_tokens': 23, 'total_tokens': 498, 'completion_time': 0.395833333, 'prompt_time': 0.003354196, 'queue_time': 0.020995134, 'total_time': 0.399187529}, 'model_name': 'llama3-8b-8192', 'system_fingerprint': 'fp_dadc9d6142', 'finish_reason': 'stop', 'logprobs': None}, id='run-a93dd537-cf6a-4e67-9d11-fb41932e9d82-0', usage_metadata={'input_tokens': 23, 'output_tokens': 475, 'total_tokens': 498})
```  

`Output Parser` 를 사용하면 언어 모델에서 생성된 원시 텍스트를 애플리케이션이 쉽게 사용할 수 있는 구조화된 데이터 형식으로 변환할 수 있다. 

- `Python` 객체
- `JSON` 객체
- 리스트
- 사용자 정의 데이터 타입

그 중 `StrOutputParser` 는 `str` 타입으로 변환해주는 파서이다. 

```python
from langchain_core.output_parsers import StrOutputParser

output_parser = StrOutputParser()
chain = prompt_template | model | output_parser
chain.invoke(input)

# 😊
# 
# 반도체(半導體)는 전자공학에서 사용되는 중요한 물질입니다. 반도체는 전기적, 열적, 광학적 성질이 있는 반질체로, 전자 회로의 기본 구성 요소입니다.
# 
# 반도체는 도선(導線)과 반도체 소자(半導體素子)를 포함합니다. 도선은 전기를 전송하기 위해 사용되는 가늘고 긴 물질로, 전기적 신호를 전달하는 데 사용됩니다. 반도체 소자는 특정한 기능을 수행하는 소자로, 예를 들어, 트랜지스터(Transistor)는 전류의 조절을 수행합니다.
# 
# 반도체는 다음과 같은 특징을 가지고 있습니다.
# 
# 1. 전기적 성질: 반도체는 전기적 전류를 통과할 수 있습니다. 그러나, 도선과는 달리 전기적 전류는 제한됩니다.
# 2. 열적 성질: 반도체는 열에 의해 전기적 성질이 변화합니다. 일반적으로, 온도가 올라갈수록 전기적 전류가 증가합니다.
# 3. 광학적 성질: 반도체는 광학적 신호를 통해 전기적 신호를 전달할 수 있습니다.
# 
# 반도체의 예로는 트랜지스터, 다이오드, IC(Integrated Circuit) 등이 있습니다. 이러한 반도체 소자들은 전자 회로의 기본 구성 요소로 사용됩니다.
# 
# 반도체는 다양한 산업에서 사용됩니다. 예를 들어, 전자 제품, 자동차, 의료 기기, 통신 장비 등에서 사용됩니다. 반도체는 전자 회로의 발전과 함께 발전하고 있습니다.
```  
