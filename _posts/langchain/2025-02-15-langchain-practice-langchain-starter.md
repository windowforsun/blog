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


### LCEL 인터페이스
`LCEL` 에서는 사용자가 정의한 체인을 실행하고 결과를 반환하는 다양한 인터페이스를 제공한다. 
아래는 표준 인터페이스로 사용자 정의 체인을 정의하고 표준 방식으로 호출하는 것을 손쉽게 할 수 있다.  

- `invoke()` : 입력에 대해 체인을 호출한다. 
- `stream()` : 응답의 청크를 스트리밍한다. 
- `batch()` : 입력 목록에 대해 체인을 호출한다. 
- `astream()` : 비동기적으로 응답의 청크를 스트리밍한다. 
- `ainvoke()` : 비동기적으로 입력에 대해 체인을 호출한다. 
- `abatch()` : 비동기적으로 입력 목록에 대해 체인을 호출한다. 
- `astream_log()` 최종 응답뿐만 아니라 발생하는 중간 단계를 스트리밍한다. 


#### invoke()
`invoke()` 메서드는 입력에 대해 체인을 호출하고 결과를 반환한다.  

```python
chain.invoke(input)

# 😊
# 
# 반도체(Semiconductor)는 전기 전달을 위한 소재입니다. 일반적으로는 실리콘(Silicon)이나 게르마늄(Germanium) 등에서 만들어집니다.
# 
# 반도체는 전기 전달에 사용되는 두 가지 특징을 가집니다.
# 
# 1. 전기전달 능력: 반도체는 전기 전달을 할 수 있습니다. 이를테면, 전류가 흐를 수 있습니다.
# 2. 절연 능력: 반도체는 전기적 경계를 가질 수 있습니다. 이를테면, 전류가 흐를 수 없는 상태를 만들 수 있습니다.
# 
# 이러한 특징으로 인해 반도체는 다양한 용도로 사용됩니다.
# 
# * 전자기기: 컴퓨터, 스마트폰, TV 등에 사용됩니다.
# * 전자부품: 반도체는 다양한 전자부품을 생산하는 데 사용됩니다. 이를테면, 트랜지스터, 다이오드, 인버터 등.
# * 반도체 반도체는 또한 광학, 의료, 에너지 등 다양한 분야에서 사용됩니다.
# 
# 반도체의 작동은 다음과 같은 과정을 거칩니다.
# 
# 1. 도핑(doping): 반도체에 도핑 물질을 추가하여 전기적 성질을 조절합니다.
# 2. 충전(charge): 반도체에 전기적 충전을 가하여 전류를 흐르게 합니다.
# 3. 전기 전달(electrical conduction): 충전된 반도체에 전류가 흐를 수 있습니다.
# 4. 절연(isolation): 반도체의 절연 능력을 사용하여 전류의 흐름을 제어합니다.
# 
# 이러한 과정으로 인해 반도체는 다양한 전자기기와 부품을 생산하는 데 사용됩니다.
```  

#### stream()
`stream()` 메서드는 응답의 청크를 스트리밍한다. 
아래 예제는 `end='|'` 를 사용해 구분자를 지정해 청크를 나누는 방법을 보여준다. 
그리고 `flush=True` 를 사용해 출력 버퍼를 즉시 비우도록 한다. 

```python
stream_answer = chain.stream(input)

for chunk in stream_answer:
  print(chunk, end="|", flush=True)

# |Here|'s| an| explanation| of| sem|icon|duct|ors| in| Korean|:
# 
# |**|반|도|체|(|semicolon|tr|)**|
# 
# |반|도|체|는| 전|기|적| 성|질|이| 반|도|체|인| 물|질|을| 일|컫|는| 용|어|입니다|.| 일반|적으로|는| 이|온|화|된| 금|속|이나| 실|리|콘| 등|에서| 생산|되는| 물|질|을| 가|리|킵|니다|.
# 
# |반|도|체|는| 전|기|적| 성|질|이| 반|도|체|적| 성|질|을| 나타|내|는| 것이|지요|.| 즉|,| 전|기|적| 성|질|이| 금|속|과| 도|체| 사이|에| 있는| 물|질|입니다|.| 이를|테|면|,| 금|속|은| 전|기|적| 성|질|이| 높|고|,| 도|체|는| 전|기|적| 성|질|이| 낮|는| 반|면|,| 반|도|체|는| 전|기|적| 성|질|이| 중|간|에| 있습니다|.
# 
# |반|도|체|는| 전|자| 기|기|에서| 중요한| 성|분|입니다|.| 예|를| 들어|,| 반|도|체|는| 전|자| 컴|퓨터|의| 메|모|리|,| 프로|세|서|,| 저장| 장|치| 등|에서| 사용|됩니다|.| 이러한| 반|도|체|는| 전|자| 기|기의| 성|능|과| 속|도를| 결정|하는| 중요한| 요|소|입니다|.
# 
# |반|도|체|의| 응|용| FIELD|는| 다음과| 같습니다|.
# 
# |*| 전|자| 컴|퓨터|:| 메|모|리|,| 프로|세|서|,| 저장| 장|치| 등|에서| 사용|됩니다|.
# |*| 전|자|통신|:| 전화|,| 인터넷|,| TV| 등|에서| 사용|됩니다|.
# |*| 에|너|지| 저장|:| 배|터|리|,| 태|양|전|지| 등|에서| 사용|됩니다|.
# |*| 의|료|기|기|:| 의|료| 기|기|,| 생|체| 센|서| 등|에서| 사용|됩니다|.
# 
# |따|라|서| 반|도|체|는| 전|자| 기|기의| 발|전|과| 성|장|에| 중요한| 역|할|을| 하는| 물|질|입니다|.||
```  

#### batch()
`batch()` 는 딕셔너리를 원소를 갖는 리스트를 인자로 받아, 각 딕셔너리에 있는 입력에 대해 체인을 호출해 일괄 처리를 수행한다. 
또한 `max_concurrency` 를 사용해 동시에 처리할 수 있는 최대 작업 수를 설정 가능하다. 

```python
answer = chain.batch([{"item" : "스마트폰"}, {"item" : "컴퓨터"}])

print(answer)
# ['스마트폰(영어: smartphone)은 일반적인 전화기에서 제공하는 기능 외에도 더 많은 기능을 제공하는 전화기입니다. 일반적으로는 컴퓨터와 같은 기능을 제공하며, 인터넷 접근, 음성통화, 문자 전송, 사진 촬영, 동영상 촬영, 게임 플레이, 애플리케이션 설치, GPS 등 다양한 기능을 제공합니다.\n\n스마트폰은 다음과 같은 특징을 가지고 있습니다.\n\n1. 터치 스크린: 스마트폰은 터치 스크린을 사용하여 사용자가 다양한 기능을 수행할 수 있습니다. 예를 들어, 스마트폰을 탭하여 전화번호를 입력하거나, 스마트폰을 슬라이드하여 웹 브라우저를 열 수 있습니다.\n2. 애플리케이션: 스마트폰에는 다양한 애플리케이션이 설치되어 있습니다. 이러한 애플리케이션은 문자 전송, 음성통화, 웹 브라우저, 게임, 카메라 등 다양한 기능을 제공합니다.\n3. 웹 브라우저: 스마트폰에는 웹 브라우저가 있습니다. 이 웹 브라우저를 사용하여 인터넷을 접근할 수 있습니다.\n4. 카메라: 스마트폰에는 카메라가 있습니다. 이 카메라를 사용하여 사진을 촬영하거나, 동영상을 촬영할 수 있습니다.\n5. GPS: 스마트폰에는 GPS가 있습니다. 이 GPS를 사용하여 위치를 확인하거나, 길을 찾을 수 있습니다.\n6. Wi-Fi 및 3G/LTE: 스마트폰에는 Wi-Fi와 3G/LTE 네트워크가 있습니다. 이 네트워크를 사용하여 인터넷을 접근하거나, 데이터를 전송할 수 있습니다.\n\n스마트폰은 다양한 기능을 제공하여, 사용자의 생활을 더 편리하게 만들 수 있습니다. 예를 들어, 스마트폰을 사용하여 banking 을 하거나, 쇼핑을 할 수 있습니다. 또한, 스마트폰을 사용하여 음악을 듣거나, 영화를 볼 수도 있습니다.', '컴퓨터는 무엇일까요? 컴퓨터는 전자 기기로서, 인간이 정보를 저장, 처리, 전송하는 데 사용하는 장치입니다.\n\n컴퓨터의 기본 구성 요소는 다음과 같습니다.\n\n1. **중앙 처리 장치 (CPU)**: 컴퓨터의 마음이라고 할 수 있는 부분으로, 정보를 처리해주는 것입니다.\n2. **메모리 (RAM)**: 컴퓨터가 현재 사용하는 정보를 저장하는 공간입니다.\n3. **저장 매체 (HDD, SSD)**: 컴퓨터가 長期 저장하는 정보를 저장하는 장치입니다.\n4. **입출력 장치**: 컴퓨터와 외부 기기와의 상호작용을 하는 장치입니다. (예: 키보드, 마우스, 모니터)\n\n컴퓨터는 다음과 같은 기능을 합니다.\n\n1. **인터넷 접근**: 인터넷에 접근하여 정보를 수집, 전송할 수 있습니다.\n2. **프로그래밍**: 컴퓨터가 해야 할 일, 즉 알고리즘을 작성하여 실행할 수 있습니다.\n3. **게임**, **음악**, **영상** 등 다양한 콘텐츠를 즐길 수 있습니다.\n4. **사무** 및 **업무** 지원: 문서 작성, 계산, 전자 메일 등의 업무를 지원합니다.\n\n컴퓨터는 다양한 형태로 사용되는데, 예를 들어 다음과 같습니다.\n\n1. **데스크톱 컴퓨터**: 일반적으로 사용되는 컴퓨터입니다.\n2. **노트북 컴퓨터**: 휴대성 있는 컴퓨터입니다.\n3. **모바일 컴퓨터**: 스마트폰, 태블릿 등 휴대용 컴퓨터입니다.\n4. **서버 컴퓨터**: 네트워크에 연결된 컴퓨터입니다.\n\n이러한 컴퓨터의 다양한 기능과 형태로, 컴퓨터는 현대社會에서 중요한 역할을 합니다.']

answer = chain.batch(
    [
        {"item" : "스마트폰"},
        {"item" : "컴퓨터"},
        {"item" : "AI"},
        {"item" : "LLM"},
        {"item" : "반도체"},
    ],
    config = {"max_concurrency" : 3}
)

print(answer)
# ['Here is a description of a smartphone in Korean:\n\n스마트폰은 컴퓨터와 전화기, 카메라, 음악 플레이어, 웹 브라우저 등 다양한 기능을 갖춘 휴대용 전자 기기입니다. 일반적으로는 손가락으로 터치하여 작동하며, 스크린에 표시되는 아이콘을 탭하여 다양한 앱을 사용할 수 있습니다.\n\n스마트폰에는 다음과 같은 기능이 있습니다.\n\n* 전화: 스마트폰으로 다른 사람과 통화할 수 있습니다.\n* 문자 메시지: 다른 사람에게 메시지를 보낼 수 있습니다.\n* 이메일: 이메일을 읽고 쓰고 발송할 수 있습니다.\n* 웹 브라우저: 인터넷에 접근하여 웹 페이지를 읽을 수 있습니다.\n* 앱: 다양한 앱을 설치하여 음악을 듣거나 사진을 찍거나 게임을 플레이할 수 있습니다.\n* 카메라: 스마트폰에 내장된 카메라를 사용하여 사진을 찍을 수 있습니다.\n* 음악 플레이어: 음악을 듣거나 음악을 저장할 수 있습니다.\n* GPS: 스마트폰의 위치를 찾을 수 있습니다.\n\n스마트폰은 일반적으로 다음과 같은 특징을 갖추고 있습니다.\n\n* 터치 스크린: 손가락으로 터치하여 작동합니다.\n* 애플리케이션: 다양한 앱을 설치하여 사용할 수 있습니다.\n* 인터넷 접근: 인터넷에 접근하여 웹 페이지를 읽을 수 있습니다.\n* 멀티 태스크: 여러 가지 작업을 동시에 수행할 수 있습니다.\n* 고성능: 빠른 처리速度과 많은 저장 공간을 제공합니다.\n\n따라서 스마트폰은 컴퓨터와 전화기, 카메라, 음악 플레이어, 웹 브라우저 등 다양한 기능을 갖추고 있어 편리하고 유용한 휴대용 전자 기기입니다.', '컴퓨터는 전 세계에서 가장 널리 사용되는 정보 처리 장치입니다. 컴퓨터는 사람의 지시를 받고 이를 수행하는 데 사용되는 소프트웨어를 실행하여 다양한 작업을 수행할 수 있습니다.\n\n컴퓨터의 기본 구성 요소는 다음과 같습니다.\n\n* **CPU(Central Processing Unit)**: 컴퓨터의 중앙 처리 장치입니다. CPU는 정보를 처리하고 실행하는 데 사용됩니다.\n* **메모리(Memory)**: 컴퓨터가 작업을 수행하는 데 사용되는 저장장치입니다. 메모리는 정보를 저장하고 접근하는 데 사용됩니다.\n* **하드 디스크(Hard Disk)**: 컴퓨터가 저장하는 데이터의 저장장치입니다. 하드 디스크는 컴퓨터의 모든 정보를 저장할 수 있습니다.\n* **모니터(Monitor)**: 컴퓨터의 출력 장치입니다. 모니터는 컴퓨터가 실행하는 정보를 표시합니다.\n* **키보드(Keyboard)**: 컴퓨터에 입력하는 정보를 입력하는 장치입니다. 키보드는 컴퓨터에 명령을 전달하는 데 사용됩니다.\n* **마우스(Mouse)**: 컴퓨터에 입력하는 정보를 선택하고 이동하는 장치입니다. 마우스는 컴퓨터에 명령을 전달하는 데 사용됩니다.\n\n컴퓨터는 다음과 같은 다양한 기능을 수행할 수 있습니다.\n\n* **문서 작업**: 컴퓨터는 문서를 작성하고 편집하는 데 사용됩니다.\n* **계산**: 컴퓨터는 수학 계산과 같은 다양한 계산을 수행할 수 있습니다.\n* **게임**: 컴퓨터는 다양한 게임을 실행할 수 있습니다.\n* **인터넷 접근**: 컴퓨터는 인터넷에 접근하여 다양한 정보를 제공합니다.\n* **뮤직 플레이**: 컴퓨터는 음악을 재생할 수 있습니다.\n\n이러한 기능을 수행하는 데는 다양한 소프트웨어가 필요합니다. 소프트웨어는 컴퓨터가 실행하는 데 사용되는 명령을 정의합니다. 예를 들어, 웹 브라우저는 컴퓨터가 인터넷에 접근하는 데 사용되는 소프트웨어입니다.\n\n컴퓨터는 다음과 같은 장점을 있습니다.\n\n* **빠른 처리速度**: 컴퓨터는 빠른 처리速度으로 다양한 작업을 수행할 수 있습니다.\n* **큰 저장 공간**: 컴퓨터는 큰 저장 공간을 제공하여 다양한 데이터를 저장할 수 있습니다.\n* **다양한 기능**: 컴퓨터는 다양한 기능을 제공하여 다양한 작업을 수행할 수 있습니다.\n\n그러나 컴퓨터도 다음과 같은 단점을 있습니다.\n\n* **고장 가능성**: 컴퓨터는 고장할 수 있습니다.\n* **보안 문제**: 컴퓨터는 보안 문제가 발생할 수 있습니다.\n* **고장 비용**: 컴퓨터는 고장될 경우 고장 비용이 발생할 수 있습니다.\n\n따라서 컴퓨터는 주의하여 사용해야 하며, 주기적으로 백업을 하여 데이터를 안전하게 저장해야 합니다.', 'AI(Artificial Intelligence)는 인공 지능을 의미하는 영어 단어입니다. 인공 지능은 인공적으로 만들어진 지능을 나타내는 것이고, 일반적으로는 컴퓨터 프로그램이나 로봇 등에 도입하여 인간의 지능을 모방하거나 초월하는 기능을 제공하는 것을 말합니다.\n\nAI는 다양한 분야에서 적용할 수 있습니다. 예를 들어, 자연어 처리(NLP), 이미지 처리, 음성 인식, 로봇공학, 의료 분야 등에서 AI를 적용하여 인간의 삶을 편리하게 만들 수 있습니다.\n\nAI는 크게 다음과 같은 세 가지로 분류할 수 있습니다.\n\n1. Narrow AI: 특정한 분야에서만 적용되는 AI입니다. 예를 들어, 특정 음성 인식, 이미지 처리 등입니다.\n2. General AI: 모든 분야에서 적용되는 AI입니다. 그러나 아직은 이러한 AI가 개발되지 않았습니다.\n3. Super AI: 인간의 지능을 초월하는 AI입니다. 이러한 AI는 아직은 개발되지 않았습니다.\n\nAI의 적용은 다음과 같습니다.\n\n1. 자동화: 인간이 하는 일을 컴퓨터가 대신 수행하는 것을 말합니다. 예를 들어, 문서 작성, 계산 등의 업무를 자동화할 수 있습니다.\n2. 협업: 인간과 AI가 함께 작업하는 것을 말합니다. 예를 들어, 의사와 AI가 함께 환자를 진료하는 경우 등입니다.\n3. 예측: AI가 미래의 상황을 예측하는 것을 말합니다. 예를 들어, 주식 시장의 예측, 날씨 예측 등입니다.\n\n그러나 AI도 다음과 같은 문제점을 가지고 있습니다.\n\n1. 고용 문제: AI가 인간의 일을 대신 수행하는 데 따라 인간의 고용 문제가 발생할 수 있습니다.\n2. 윤리 문제: AI가 인간의 의사를 해석하는 데 따라 윤리적 문제가 발생할 수 있습니다.\n3. 안정성 문제: AI가 안정적인지 확인하는 것이 중요합니다. 예를 들어, AI가 시스템을 장애로 만드는 경우 등입니다.\n\n따라서 AI는 이러한 문제점을 해결하는 것이 중요합니다. AI의 개발과 적용을 통해 인간의 삶을 편리하게 만들 수 있습니다.', "Here's an explanation of LLM in Korean:\n\n**LLM**는 **Large Language Model**의 약자로, 기계학습 기반의 언어 모델입니다. LLM은 텍스트 데이터를 학습하여 언어의 구조, 문법, 의미 등을 이해하는 능력을 개발합니다.\n\nLLM은 다음과 같은 기능을 갖추고 있습니다.\n\n1. **자연어 처리**: LLM은 텍스트 데이터를 입력받아 이를 이해하고, 의미를 분석하여 출력을 생성할 수 있습니다.\n2. **언어 생성**: LLM은 입력된 텍스트를 기반으로 새로운 텍스트를 생성할 수 있습니다. 예를 들어, 입력된 문장에 대한 답을 생성하거나, 새로운 문장을 생성할 수 있습니다.\n3. **자연어 이해**: LLM은 텍스트를 읽어 의미를 이해하고, 문맥을 분석하여 궁극적으로는 질문에 대한 답을 찾을 수 있습니다.\n\nLLM은 다양한 분야에서 응용 가능합니다.\n\n1. **Chatbot**: LLM을 기반으로 하는 Chatbot은 사용자와 대화를 나눌 수 있습니다.\n2. **Text summarization**: LLM은 긴 텍스트를 요약할 수 있습니다.\n3. **Language translation**: LLM은 언어를 번역할 수 있습니다.\n4. **Content generation**: LLM은 새로운 콘텐츠를 생성할 수 있습니다.\n\n그러나 LLM도 다음과 같은 제한을 가집니다.\n\n1. **언어의 복잡성**: LLM은 언어의 복잡성, 특히 문법과 의미의 복잡성을 이해하는 데 제한이 있습니다.\n2. **데이터 품질**: LLM의 성능은 입력된 데이터의 품질에 따라 결정됩니다.\n3. **선택적 이해**: LLM은 특정 문맥에서 특정 의미를 이해하는 데 제한이 있습니다.\n\n따라서 LLM은 다음과 같은 방향으로 발전해야 합니다.\n\n1. **언어의 복잡성 이해**: LLM은 언어의 복잡성을 더 잘 이해해야 합니다.\n2. **데이터 품질 향상**: LLM의 입력된 데이터의 품질을 높이려면 더 많은 데이터를 수집하고, 데이터의 품질을 평가해야 합니다.\n3. **선택적 이해 보완**: LLM은 특정 문맥에서 특정 의미를 이해하는 데 제한이 있는 경우, 이를 보완하는 방법을 개발해야 합니다.\n\n이러한 방향으로 LLM은 향후 더 많은 분야에서 응용될 수 있습니다.", '반도체는 semiconductor materials를 사용하여 반도체 부품을 생산하는 산업입니다. 반도체 부품은 전자 기기, 컴퓨터, 스마트폰 등 전자 제품에 사용되는 기본 구성 요소입니다.\n\n반도체는 다공성 물질을 가질 수 있습니다. 즉, 전자가 이동할 수 있는 곳과 이동할 수 없는 곳이 있습니다. 이러한 특징으로 인해 반도체는 전자 회로의 기본 구성 요소로 사용됩니다.\n\n반도체 부품은 다음과 같은 기능을 수행할 수 있습니다.\n\n1. 전류 제어: 반도체 부품은 전류의 흐름을 제어할 수 있습니다. 예를 들어, 스위치, 트랜지스터, 소자 등이 있습니다.\n2. 전압 제어: 반도체 부품은 전압의 값을 제어할 수 있습니다. 예를 들어, 증폭기, 제네레이터 등이 있습니다.\n3. 데이터 저장: 반도체 부품은 데이터를 저장할 수 있습니다. 예를 들어, 메모리, 하드디스크 등이 있습니다.\n\n반도체 산업은 빠르게 발전하고 있습니다. 새로운 반도체 공정 기술, 반도체 재료 개발, 반도체 설계 소프트웨어 등이 개발되고 있습니다. 이로 인해 반도체 부품의 기능이 향상되고, 가격이 낮아지고 있습니다.\n\n암호화, Artificial Intelligence, Internet of Things(IoT) 등 새로운 기술에서도 반도체 부품이 중요한 역할을 수행하고 있습니다. 따라서 반도체 산업의 발전은 전자 제품의 발전을 좌우하는 요소입니다.']
```  

#### astream()
`astream()` 메서드는 비동기적으로 응답의 청크를 스트리밍한다. 

```python
astream_answer = chain.astream({"item" : "커피"})

async for chunk in astream_answer:
  print(chunk, end="|", flush=True)


# 커피|(|caf|ē|)는| 세계|에서| 가장| популяр|한| 음|료|之一|입니다|.| 커피|는| 커피|나무|(C|off|ea| arab|ica|)|나| 커피|나무|(C|off|ea| can|eph|ora|)|에서| 추|출|된|콩|의| 씨|앗|을| 로|스|팅|하여| 얻|은| 향|이| 있는| 열|매|입니다|.
# 커피|는| 다양한| 방법|으로| 즐|길| 수| 있습니다|.| 가장| 일반|적인| 방법|은| 커피| 로|스|팅|하여| 에|스|프|레|소|를| 만들|거나|,| 커피| 그|라인|더|를| 사용|하여| 커피|粉|을| 만들|고|,| 물|에| 녹|인| 후| 마|시는| 방법|입니다|.| 또한| 커피|는| 차|,| 미국|식| 커피|,| 아이|티|식| 커피| 등| 다양한|样|式|로| 즐|길| 수| 있습니다|.
# 커피|에는| 다양한| 향|과| 맛|이| 있습니다|.| 일반|적으로| 커피|는| 스펀|디|드|하게| 향|하는| 초|록|색|,| 보|랏|빛|,| 또는| 브라|운|빛|의| 향|을| 지|니|고| 있습니다|.| 커피|의| 맛|은| 종|류|,| 로|스|팅|,| 추|출| 방법|에| 따라| 다|르|며|,| 일부|는| 짙|은| 향|,| 일부|는| 경|쾌|한| 향|을| 지|니|고| 있습니다|.
# 커피|는| 세계|적으로|는| 다양한| 문화|와| 전|통|으로| 즐|겨| 먹| __("|C|af|é| culture|"|는| 커피|를| 즐|기는| 문화|입니다|.| 커피|는| 사회|에서| 중요한| 지|목|으로|,| 친구|와| 가족|과| 함께| 커피|를| 즐|기는| 것은| 일반|적|입니다|.||
```  

#### ainvoke()
`ainvoke()` 메서드는 비동기적으로 체인을 호출한다. 

```python
await chain.ainvoke({"item" : "CPU"})

# CPU(Central Processing Unit)는 컴퓨터의 중앙 처리 장치로, 프로그램의 명령을 실행하고 데이터를 처리하는主要의 장치입니다.
# 
# CPU는 다음과 같은 기능을 수행합니다.
# 
# 1. **명령의 실행**: CPU는 컴퓨터가 실행하는 프로그램의 명령을 읽어들여, 해당하는 작업을 수행합니다.
# 2. **데이터 처리**: CPU는 입력된 데이터를 처리하여, 필요한 작업을 수행합니다. 예를 들어, 수식을 계산하거나, 데이터를 저장하거나, 데이터를 전송하는 등입니다.
# 3. **기억장치와의 통신**: CPU는 컴퓨터의 기억장치(rams, hard drive, etc.)와 통신하여, 필요한 데이터를 읽거나, 저장합니다.
# 4. **제어**: CPU는 컴퓨터의 모든 부품을 제어하여, 프로그램이 실행되는 것을 관리합니다.
# 
# CPU의 구성 요소는 다음과 같습니다.
# 
# 1. **Control Unit**: CPU의 제어부문으로, 명령을 읽어들여, 실행하는 데 필요한 데이터를 조정합니다.
# 2. **Arithmetic Logic Unit (ALU)**: CPU의 산술논리부문으로, 수식을 계산하거나, 논리적 조건을 확인합니다.
# 3. **Registers**: CPU의 레지스터는, CPU가 실행하는 프로그램의 데이터를 저장하는 일종의 메모리입니다.
# 4. **Cache Memory**: CPU의 캐시 메모리는, CPU가 자주 사용하는 데이터를 저장하는 일종의 메모리입니다.
# 
# CPU의 성능은 다음과 같은 요수로 결정됩니다.
# 
# 1. **Clock Speed**: CPU의 클록 속도는, CPU가 1초에 처리할 수 있는 명령의 수를 나타냅니다.
# 2. **Number of Cores**: CPU의 코어 수는, CPU가 실행할 수 있는 프로그램의 수를 나타냅니다.
# 3. **Cache Memory Size**: CPU의 캐시 메모리 크기는, CPU가 자주 사용하는 데이터를 저장할 수 있는 크기를 나타냅니다.
# 4. **Threads**: CPU의 스레드는, CPU가 실행할 수 있는 프로그램의 수를 나타냅니다.
# 
# 따라서, CPU는 컴퓨터의 중앙 처리 장치로, 프로그램의 명령을 실행하고 데이터를 처리하는主要의 장치입니다. CPU의 성능은 다양한 요소에 의해 결정됩니다.
```  

#### abatch()
`abatch()` 메서드는 비동기적으로 일련의 작업을 일괄 처리한다. 

```python
answer = chain.abatch([{"item" : "HBM"}, {"item" : "감기"}])

await answer

# ['HBM(Human Brain Mapping)은 인간의 뇌 구조 및 기능을 조사하기 위해 사용하는 기술입니다. HBM은 Functional Magnetic Resonance Imaging (fMRI), Electroencephalography (EEG), Magnetoencephalography (MEG), Functional Near-Infrared Spectroscopy (fNIRS), 및 other techniques를 포함하는 broad field입니다.\n\nHBM의 목표는 인간의 뇌 기능을 이해하고, 이를 통해 정신 건강 질환, 학습 및记忆, 언어 및 감정 조절, 및 다른 cognitve processes에 대한 이해를 도울 수 있습니다. HBM은 또한 신경이상, 뇌손상, 및 뇌질환의 diagnose 및 평가를 도와 줄 수 있습니다.\n\nHBM을 사용하는 방법은 다음과 같습니다.\n\n1. fMRI: 뇌의 기능을 조사하기 위해 Blood oxygenation level-dependent (BOLD) signaling을 사용합니다.\n2. EEG: 뇌의 전기적 활동을 조사하기 위해 전기 신호를 측정합니다.\n3. MEG: 뇌의 전자磁気적 활동을 조사하기 위해 전자磁気 신호를 측정합니다.\n4. fNIRS: 뇌의 기능을 조사하기 위해 near-infrared 빛을 사용하여 뇌의 산소 농도를 측정합니다.\n\nHBM은 다양한 field, including neuroscience, psychology, medicine, engineering, 및 computer science에서 사용됩니다. HBM을 통해 얻는 정보를 사용하여 새로운 치료법, 예방책, 및 생명공학 제품을 개발할 수 있습니다.',
# "Here's an explanation of a common cold in Korean:\n\n감기는 일반적으로 바이러스에 의해 일으켜지는 질병입니다. 감기는 Респиратор계통 감염증으로, 해열, 인후통, 호흡곤란 등의 증세를 보입니다. 감기는 일반적으로 7-14일 동안 지속되는 경우가 많습니다.\n\n감기는 다양한 유형이 있습니다. 가장 흔한 유형은 Rhinovirus, Coronaviruses, Adenoviruses, Para-influenza viruses,와 Respiratory syncytial virus입니다. 감기는 감기 바이러스에 의해 일으켜진 경우가 가장 일반적입니다.\n\n감기는 일반적으로 다음과 같은 방법으로 전염됩니다.\n\n* 공기 중에 바이러스가 있는 경우, 호흡을 통해 바이러스가 몸 속으로 들어옵니다.\n* 바이러스가 있는 물체에 접촉하여 바이러스가 몸 속으로 들어옵니다.\n* 바이러스가 있는 사람과 접촉하여 바이러스가 몸 속으로 들어옵니다.\n\n감기는 다음과 같은 방법으로 예방할 수 있습니다.\n\n* 손을 자주 씻어 바이러스가 있는 물체와 접촉을 최소화합니다.\n* 기침이나 통증할 때는 마스크를 쓰고, 방금 기침하거나 통증한 후에는 방금 사용한 마스크를 세척하여 바이러스가 있는 물체와 접촉을 최소화합니다.\n* 바이러스가 있는 사람과 접촉을 최소화합니다.\n* 매일 잘 생활하고, 면역을 강화하여 감기를 예방할 수 있습니다.\n\n감기는 일반적으로 치료가 필요하지 않습니다. 하지만, 감기 증세가 심하거나 호흡곤란 등의 증세가 있을 경우에는 의사에게 방문하여 치료를 받는 것이 좋습니다."]
```

### Runnable
`Runnable` 은 `LangChain` 에서 제공하는 다양한 작업을 실행할 수 있는 구성 요소이다. 
체인내에서 실행 가능한 단위로, 입력을 받아 처리하고 결과를 반환하는 역할을 한다. 


#### RunnablePassthrough
`RunnablePassthrough` 는 입력을 수정하지 않고 다음 구성 요소로 전달하는 `Runnable` 이다. 

```python
from langchain_core.runnables import RunnablePassthrough

RunnablePassthrough().invoke({"num": 4})
# {'num': 4}
```  

체인과 함께 사용하면 아래와 같다. 

```python
from langchain_core.runnables import RunnablePassthrough

template = "{num}의 제곱근은?"
prompt = PromptTemplate.from_template(template)

runnable_chain = {"num" : RunnablePassthrough()} | prompt | model

runnable_chain.invoke(4)

# AIMessage(content='4의 제곱근은 2입니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 12, 'prompt_tokens': 18, 'total_tokens': 30, 'completion_time': 0.01, 'prompt_time': 0.002557211, 'queue_time': 0.018841138, 'total_time': 0.012557211}, 'model_name': 'llama3-8b-8192', 'system_fingerprint': 'fp_a97cfe35ae', 'finish_reason': 'stop', 'logprobs': None}, id='run-2f76e25c-0f99-44a4-9dbe-845865f0e4e3-0', usage_metadata={'input_tokens': 18, 'output_tokens': 12, 'total_tokens': 30})
```  

`RunnablePassthrough.assign()` 을 사용하면 입력 값으로 들어온 `key-value` 쌍과 새롭게 할당된 `key-value` 쌍을 합친다.  

```python
runnable_chain = {"num" : (RunnablePassthrough.assign(new_num=lambda x: x["num"] * 3))} | prompt | model

runnable_chain.invoke({"num" :4})

# AIMessage(content="A nice math question! 😊\n\nGiven the dictionary `{'num': 4, 'new_num': 12}`, we can calculate the square root of each value:\n\n* `num`: sqrt(4) = 2\n* `new_num`: sqrt(12) = √12 ≈ 3.46\n\nSo, the square roots are 2 and approximately 3.46. 👍", additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 83, 'prompt_tokens': 30, 'total_tokens': 113, 'completion_time': 0.069166667, 'prompt_time': 0.004079745, 'queue_time': 0.021964935, 'total_time': 0.073246412}, 'model_name': 'llama3-8b-8192', 'system_fingerprint': 'fp_179b0f92c9', 'finish_reason': 'stop', 'logprobs': None}, id='run-f72e875e-3c33-4009-ae1d-78b532a5fb01-0', usage_metadata={'input_tokens': 30, 'output_tokens': 83, 'total_tokens': 113})
```  


#### RunnableParallel
`RunnableParallel` 은 여러 `Runnable` 구성 요소를 병렬로 실행하고 결과를 수집하는 `Runnable` 이다.  

```python
from langchain_core.runnables import RunnableParallel

runnable = RunnableParallel(
    # 입력 그대로 전달
    passed = RunnablePassthrough(),
    # 입력된 값에 3을 곱한 값을 추가로 전달
    extra = RunnablePassthrough.assign(mult=lambda x : x["num"] * 3),
    # 입력된 값에 1을 더한 값을 반환
    modified=lambda x : x["num"] + 1
)

runnable.invoke({"num":1})

# {'passed': {'num': 1}, 'extra': {'num': 1, 'mult': 3}, 'modified': 2}
```  

체인과 함께 사용하면 아래와 같다.  

```python
chain1 = (
    {"num": RunnablePassthrough()}
    | PromptTemplate.from_template("{num} 의 제곱근은?")
    | model
)
chain2 = (
    {"num": RunnablePassthrough()}
    | PromptTemplate.from_template("{num} 의 제곱은?")
    | model
)

runnable_parallel_chain = RunnableParallel(sqrt=chain1, square=chain2)

runnable_parallel_chain.invoke(4)
# {'sqrt': AIMessage(content='4의 제곱근은 2입니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 12, 'prompt_tokens': 18, 'total_tokens': 30, 'completion_time': 0.01, 'prompt_time': 0.002575389, 'queue_time': 0.021647529999999998, 'total_time': 0.012575389}, 'model_name': 'llama3-8b-8192', 'system_fingerprint': 'fp_179b0f92c9', 'finish_reason': 'stop', 'logprobs': None}, id='run-a6b08b96-4653-49aa-847d-59c4a6b4a485-0', usage_metadata={'input_tokens': 18, 'output_tokens': 12, 'total_tokens': 30}),
# 'square': AIMessage(content='4의 제곱은 16입니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 11, 'prompt_tokens': 17, 'total_tokens': 28, 'completion_time': 0.009166667, 'prompt_time': 0.006663523, 'queue_time': 0.24887254199999997, 'total_time': 0.01583019}, 'model_name': 'llama3-8b-8192', 'system_fingerprint': 'fp_a97cfe35ae', 'finish_reason': 'stop', 'logprobs': None}, id='run-d1e3aef2-9e18-4158-a130-dd285d005e7b-0', usage_metadata={'input_tokens': 17, 'output_tokens': 11, 'total_tokens': 28})}
```  

#### RunnableLambda
`RunnableLambda` 는 사용자 정의 함수를 실행할 수 있는 `Runnable` 이다. 
이는 기본 제공되는 `Runnable` 유형으로 처리되지 않는 특정 작업을 수행해야 할 떄 유요하다.  


```python
from datetime import datetime
from langchain_core.runnables import RunnableLambda, RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

def get_today_str(a):
  return datetime.today().strftime("%Y-%m-%d")

prompt = PromptTemplate.from_template("{today} 에 세계 주요한 이슈 {num}개 한글로 소개해 주세요.")

chain = (
    {"today" : RunnableLambda(get_today_str), "num" : RunnablePassthrough()} | prompt | model | StrOutputParser()
)

chain.invoke(2)

# Here are two major global issues as of March 22, 2025, introduced in Korean:
# 
# **1. 코로나19 9년째, 재유행 가능성 점검**
# 
# As of March 2025, the world is still dealing with the aftermath of the COVID-19 pandemic, which has lasted for nearly 9 years. The virus continues to evolve, and there is a growing concern about the possibility of a new wave of infections. Governments and health organizations around the world are closely monitoring the situation and taking measures to prevent the spread of the virus. The World Health Organization (WHO) has warned that the virus is still a major threat to global public health and that complacency could lead to a resurgence of cases.
# 
# **2. 우크라이나-러시아 전쟁, 유럽안보 위기**
# 
# The conflict between Ukraine and Russia, which began in 2022, continues to escalate, posing a significant threat to European security. The war has caused widespread humanitarian suffering, displacement, and economic devastation, and has also led to a significant increase in tensions between NATO member countries and Russia. The United States, Europe, and other countries have imposed severe economic sanctions on Russia, while Russia has retaliated by cutting off gas supplies to Europe. The situation remains uncertain, and there is a growing concern about the potential for further escalation and the impact on global stability.
```  

`operator` 패키지의 `itemgetter` 를 사용하면 특정 키를 추출해 좀 더 다양한 커스텀한 작업을 수행할 수 있다.  

```python
from operator import itemgetter

def str_to_length(str):
  return len(str)

def multiply_multiple_str_length(_dict):
  return str_to_length(_dict["text1"]) * str_to_length(_dict["text2"])

prompt = PromptTemplate.from_template("{num1} + {num2} 의 계산 결과는?")

chain = (
    {
        "num1" : itemgetter("str1") | RunnableLambda(str_to_length),
        "num2" : {"text1" : itemgetter("str2"), "text2" : itemgetter("str3")} 
        | RunnableLambda(multiply_multiple_str_length),
    }
    | prompt
    | model
)


chain.invoke({"str1": "My", "str2": "name is", "str3" : "Jack"})
# AIMessage(content='2 + 28 = 30', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 8, 'prompt_tokens': 20, 'total_tokens': 28, 'completion_time': 0.006666667, 'prompt_time': 0.00303913, 'queue_time': 0.02161509, 'total_time': 0.009705797}, 'model_name': 'llama3-8b-8192', 'system_fingerprint': 'fp_179b0f92c9', 'finish_reason': 'stop', 'logprobs': None}, id='run-37b6acf1-4dfb-4740-8b3e-f8f1366ac510-0', usage_metadata={'input_tokens': 20, 'output_tokens': 8, 'total_tokens': 28})
```





---  
## Reference
[LangChain 시작하기](https://wikidocs.net/233341)  
[Trace with LangChain (Python and JS/TS)](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_langchain)  


