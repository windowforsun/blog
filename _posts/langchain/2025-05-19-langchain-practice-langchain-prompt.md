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
    - Prompt
    - PromptTemplate
    - ChatPromptTemplate
    - MessagesPlaceholder
toc: true
use_math: true
---  

## Prompt
`LangChain` 에서 `Prompt` 는 언어 모델에 대한 입력을 구조화하는 템플릿을 의미한다. 
이를 통해 특정 값으로 자리 표시자국을 채워 동적이고 재사용 가능한 프롬프트를 생성할 수 있다. 
이는 일관되고 구조화된 쿼리를 언어 모델에 생성하는데 도움을 준다. 

프롬프트의 주요 구성요소는 템플릿과 변수가 있는데, 
템플릿은 실제 값으로 채워질 자리 표시자가 있는 문자열을 의미한다. 
그리고 변수는 템플릿의 자리 표시자로, 프롬프트가 호출될 때 실제 값으로 대체된다.   

프롬프트의 필요성을 정리하면 아래와 같다. 

- 구조화된 입력 : 프롬프트는 입력을 일관되고 구조화된 형식으로 만들어 언어 모델이 더 정확하게 이해하고 처리할 수 있도록 돕는다. 
- 재사용성 : 템플릿과 변수를 사용해 다양한 상황에서 동일한 형식을 재사용할 수 있어 효율적이다. 
- 동적 입력 : 자리 표시자를 사용해 다양한 입력 값을 동적으로 채울 수 있어 유연한 쿼리 생성이 가능하다. 
- 맥락 제공 : 언어 모델에 추가적인 맥락을 제공하여 더 정확하고 관련성 높은 응답을 생성할 수 있게 한다. 

프폼프트의 중요성을 정리하면 아래와 같다. 

- 일관성 향상 : 표준화된 프롬프트 형식을 통해 언어 모델은 일관된 형태의 응답을 생성한다. 이는 안정적인 서비스 제공에 중요한 역할을 한다. 
- 모델 성능 최적화 : 잘 설계된 프롬프트는 언어 모델의 응답 품질을 크게 향상시킨다. 직절한 지시와 제약 조건을 통해 모델이 원하는 방식으로 응답하도록 유도한다. 
- RAG와 통합 : 검색 증상 생성(`RAG`) 시스템에서 프롬프트는 검색된 정보를 효과적으로 통합하는 핵심 요소이다. 검색 결과와 사용자 쿼리를 조합하여 정확하고 정보가 풍부한 응답을 생성할 수 있다. 
- 복잡한 워크플로우 구현 : 여러 프롬프트를 체인으로 연결하여 복잡한 추론 과정을 구현할 수 있다. 멀티 스텝 작업, 중간 검증, 조건부 처리 등을 가능하게 한다. 
- 문제 해결 및 디버깅 용이성 : 명확하게 정의된 프롬프트는 `AI` 시스템의 동작을 추적하고 이해하기 쉽게 만든다. 문제가 발생했을 때 원인 파악과 수정이 용이하다.  

`LangChain` 에서 `Prompt` 는 몇 클래스와 방법을 사용해 구현할 수 있다. 
예제를 통해 생성 및 사용법에 대해 알아본다. 

### PromptTemplate
`PromptTemplate` 은 언어 모델과 상호작용하기 위한 구조화된 템플릿을 생성한다. 
고정된 텍스트와 변수 자리표시자로 구성된 문자열 템플릿을 정의하고, 
실행 시점에 변수에 실제 값을 주입하여 완성된 프롬프트를 생성할 수 있다. 
이를 통해 일관된 형식으로 언어 모델에 입력을 제공할 수 있게 한다.  

`from_template()` 메서드를 사용하면 템플릿을 기반으로 프롬프트를 생성할 수 있다.  

```python
from langchain_core.prompts import PromptTemplate

template = "당신은 계산기입니다. {exp} 의 결과를 알려주세요"

prompt = PromptTemplate.from_template(template)
# PromptTemplate(input_variables=['exp'], input_types={}, partial_variables={}, template='당신은 계산기입니다. {exp} 의 결과를 알려주세요')

prompt.format(exp="1 + 1")
# 당신은 계산기입니다. 1 + 1 의 결과를 알려주세요

chain = prompt | model

# 변수가 1개인 경우 별도로 딕셔너리로 값을 전달하지 않아도 된다. 
chain.invoke("1 + 1").content
# 😊
# 
# 1 + 1 = 2
```  

`PrompteTemplate` 객체 생성과 동시에 템플릿을 정의할 수 있다. 
생성자의 인자로는 `input_variables` 가 있는데, 
이는 프롬프트 템플릿에서 사용되는 변수를 정의한다. 
템플릿에서 실행횔 때 반드시 값이 제공되어야 하는 변수들의 목록을 의미한다. 
`from_template()` 을 사용할 때 자동으로 감지되지만, 직접 지정하는 것도 가능하다. 

```python
prompt = PromptTemplate(
    template = template,
    input_variables = ["exp"]
)
# PromptTemplate(input_variables=['exp'], input_types={}, partial_variables={}, template='당신은 계산기입니다. {exp} 의 결과를 알려주세요')

prompt.format(exp="2 * 2")
# 당신은 계산기입니다. 2 * 2 의 결과를 알려주세요
```  

다음으로 사용할 수 있는 생성자 인자는 `partial_variables` 가 있다. 
이는 미리 값이 할당된 변수를 정의할 수 있다.
템플릿을 사용할 때마다 제공할 필요가 없는 고정된 값을 가진 변수나 
혹은 반복적으로 사용되는 값이나 다른 함수의 결과값을 미리 저장할 때 유용하다.  

```python
template = "당신은 계산기입니다. {exp1} + {exp2} 의 결과를 알려주세요"

prompt = PromptTemplate(
    template = template,
    input_variables=["exp1"],
    partial_variables={
        "exp2" : "2 * 2"
    }
)
# PromptTemplate(input_variables=['exp1'], input_types={}, partial_variables={'exp2': '2 * 2'}, template='당신은 계산기입니다. {exp1} + {exp2} 의 결과를 알려주세요')

prompt.format(exp1="1 + 1")
# 당신은 계산기입니다. 1 + 1 + 2 * 2 의 결과를 알려주세요

prompt_partial = prompt.partial(exp2="3 * 3")
# PromptTemplate(input_variables=['exp1'], input_types={}, partial_variables={'exp2': '3 * 3'}, template='당신은 계산기입니다. {exp1} + {exp2} 의 결과를 알려주세요')

prompt_partial.format(exp1="1 + 1")
# 당신은 계산기입니다. 1 + 1 + 3 * 3 의 결과를 알려주세요

chain = prompt_partial | model

chain.invoke("1 + 1").content
# Let's calculate!
# 
# First, we need to follow the order of operations (PEMDAS):
# 
# 1. Multiply 3 and 3: 3 * 3 = 9
# 2. Add 1 + 1: 1 + 1 = 2
# 3. Add 2 and 9: 2 + 9 = 11
# 
# So, the result is: 11 

chain.invoke({"exp1" : "2 + 3", "exp2" : "3 * 2"}).content
# I'd be happy to calculate the result for you! 😊
# 
# First, let's follow the order of operations (PEMDAS):
# 
# 1. Multiply 3 and 2: 3 * 2 = 6
# 2. Add 2 and 3: 2 + 3 = 5
# 3. Add 5 and 6: 5 + 6 = 11
# 
# So, the result is: 11! 🎉
```  

앞서 언급한 것처럼 `partial_variables` 를 사용하면 함수를 호출해 변수 값을 지정할 수 있다. 
가장 대표적인 것이 바로 실시간 반영이 필요한 날짜나 시간이다.  

```python
from datetime import datetime

template = "당신은 세계의 모든 뉴스를 모니터링하는 전문가입니다. 오늘 날짜 {today}의 가장 대표적인 키워드만 {num}개 한글로 나열하세요. 설명은 제외하세요."

def get_today():
  return datetime.now().strftime("%Y-%m-%d")

prompt = PromptTemplate(
    template = template,
    input_variables = ["num"],
    partial_variables = {
        "today" : get_today
    }
)

prompt.format(num=3)
# 당신은 세계의 모든 뉴스를 모니터링하는 전문가입니다. 오늘 날짜 2025-03-23의 가장 대표적인 키워드만 3개 한글로 나열하세요. 설명은 제외하세요.

chain = prompt | model
chain.invoke(3).content
# 1. 인공지능
# 2. 우주개발
# 3. 기후변화

chain.invoke({"today" : "2018-12-31", "num" : 4}).content
# 1. 연말
# 2. 불법주차
# 3. 세이브더칠드런
# 4. 아시아나항공
```  

프롬프트의 템플릿을 파일에서 로드하는 방법도 제공한다. 
앞서 사용한 계산기를 예제로 예를 들면 아래와 같은 형식들로 작성해 사용할 수 있다.  

- 일반 텍스트

```text
당신은 계산기입니다. {exp1} + {exp2} 의 결과를 알려주세요
```

- `yaml` 파일

```yaml
_type: prompt
template: |
  당신은 계산기입니다. 
  {exp1} + {exp2} 의 결과를 알려주세요
input_variables:
  - exp1
partial_variables:
  exp2: 2 * 2
```  

- `json` 파일

```json
{
  "_type": "prompt",
  "template": "당신은 계산기입니다. {exp1} + {exp2} 의 결과를 알려주세요",
  "input_variables": ["exp1"],
  "partial_variables": {
    "exp2": "2 * 2"
  }
}
```  
