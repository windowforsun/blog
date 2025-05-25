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

`json` 프롬프트 파일을 읽고 체인을 생성해 실행하는 예제는 아래와 같다.  

```python
from langchain_core.prompts import load_prompt

prompt = load_prompt("calculator.json")
# PromptTemplate(input_variables=['exp1'], input_types={}, partial_variables={'exp2': '2 * 2'}, template='당신은 계산기입니다. {exp1} + {exp2} 의 결과를 알려주세요')

prompt.format(exp1="1 - 1")
# 당신은 계산기입니다. 1 - 1 + 2 * 2 의 결과를 알려주세요

chain = prompt | model
chain.invoke("1 - 1").content
# 1 - 1 + 2 * 2 의 결과를 계산해 보겠습니다.
# 
# 1. 먼저, 곱셈을 계산합니다: 2 * 2 = 4
# 2. 다음, 뺄셈과 덧셈을 계산합니다: 1 - 1 = 0
# 3. 마지막으로, 결과를 더합니다: 0 + 4 = 4
# 
# 따라서, 1 - 1 + 2 * 2 의 결과는 4입니다.
```

### ChatPromptTemplate
`ChatPromptTemplate` 은 대화형 언어 모델(`ChatGPT`, `Claude` 등) 과 상호작용하기 위해 특별히 설계된 `LangChain` 의 프롬프트 클래스이다. 
`PromptTemplate` 은 단일 문자열 템플릿을 다룬다면, 
`ChatPromptTemplate` 은 각기 다른 역할을 가진 여러 메시지로 구성된 대화 구조를 관리한다. 

- 역할 기반 메시지 : `system`, `user`, `assistant` 등 역할 구분
- 대화 시퀀스 : 여러 메시지의 순차적 배열
- 변수 삽입 : 각 메시지 내용에 동적 변수 삽입 가능

주요 메시지 타입으로는 아래와 같은 것들이 있다. 

- `SystemMessage` : 시스템 메시지로, 사용자에게 보이지 않는 모델의 전반적인 동작과 성격을 정의하는 정보를 전달한다.
- `HumanMessage` : 사용자 메시지로, 사용자의 입력을 나타낸다.
- `AIMessage` : `AI` 메시지로, 모델의 응답을 나타낸다.

`PrompteTemplate` 과 차이를 정리하면 아래와 같다. 

- 구조적 차이로는 `ChatPromptTemplate` 은 여러 역할을 가진 시퀀스를 사용한다느 점이 있다. 
- 출력의 차이로는 단일 문자열이 아닌 메시지 객체 리스트를 반환한다. 
- 용도의 차이로는 대화형 모델에 최적화 되어 있다. 

이렇게 `ChatPromptTemplate` 을 사용하면 대화 흐름을 더 자연스럽게 설계하고 모델의 페르소나와 대화 맥락을 효과적으로 제어할 수 있다.  
`ChatPromptTemplate` 의 생성은 메시지 클래스를 직업 사용하는 방법이 있고, 문자열 튜플로 간단하게 생성하는 방법이 있다. 
사용 예시는 아래와 같다. 

```python

from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate, AIMessagePromptTemplate

chat_prompt = ChatPromptTemplate.from_template("당신은 계산기입니다. {exp} 의 결과를 알려주세요.")
# ChatPromptTemplate(input_variables=['exp'], input_types={}, partial_variables={}, messages=[HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['exp'], input_types={}, partial_variables={}, template='당신은 계산기입니다. {exp} 의 결과를 알려주세요.'), additional_kwargs={})])

chat_prompt.format(exp="1 + 1")
# Human: 당신은 계산기입니다. 1 + 1 의 결과를 알려주세요.

chat_template = ChatPromptTemplate.from_messages(
    [
        ("system", "당신은 유능한 계산기입니다. 결과에 무조건 {num}을 더하세요."),
        ("human", "가능한 연산의 종류는 무엇인가요 ?"),
        ("ai", "덧셈, 뺄셈, 곱셈, 나눗셈 입니다."),
        ("human", "{exp}의 결과를 알려주세요.")
    ]
)
# or
chat_template = ChatPromptTemplate.from_messages(
    [
        SystemMessagePromptTemplate.from_template("당신은 유능한 계산기입니다. 결과에 무조건 {num}을 더하세요."),
        HumanMessagePromptTemplate.from_template("가능한 연산의 종류는 무엇인가요 ?"),
        AIMessagePromptTemplate.from_template("덧셈, 뺄셈, 곱셈, 나눗셈 입니다."),
        HumanMessagePromptTemplate.from_template("{exp}의 결과를 알려주세요.")
    ]
)


message = chat_template.format_messages(
    num="1", exp="1 + 1"
)
# [SystemMessage(content='당신은 유능한 계산기입니다. 결과에 무조건 1을 더하세요.', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='가능한 연산의 종류는 무엇인가요 ?', additional_kwargs={}, response_metadata={}),
# AIMessage(content='덧셈, 뺄셈, 곱셈, 나눗셈 입니다.', additional_kwargs={}, response_metadata={}),
# HumanMessage(content='1 + 1의 결과를 알려주세요.', additional_kwargs={}, response_metadata={})]

model.invoke(message).content
# 1 + 1 = 2에 1을 더하면 3입니다.

chain = chat_template | model
chain.invoke({
    "num" : "2",
    "exp" : "1 + 1"
}).content
# 1 + 1 = 2에 2를 더하면 4입니다.
```  

`ChatPromptTemplate` 에서 `MessagesPlaceholder` 는
메시지 전체를 동적으로 삽입할 수 있게 해주는 특별한 자리표시자이다. 
일반적으로 자리표시자는 `{var}` 와 같이 문자열 내에서 값을 치환하는 하지만, 
`MessagesPlaceholder` 는 메시지 객체 자체를 동적으로 삽입한다. 
이를 통해 메시지 시퀀스 내에 변수로 제공된 전체 메시지를 포함할 수 있다. 
주요 특징으로는 아래와 같은 것들이 있다. 

- 대화 기록 관리 : 이전 대화 내용으 유지하며 새 입력에 컨텍스트를 제공할 수 있다. 
- 동적 메시지 삽입 : 메시지 객체 리스트를 그대로 삽입할 수 있다. 
- 복잡한 대화 구조 : 다중 턴 대화나 복잡한 메시지 구조 구현에 적합하다.  

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import MessagesPlaceholder

chat_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "당신는 사용자의 가장 친한 친구입니다. 친구 처럼 대화를 이어가세요."),
        MessagesPlaceholder(variable_name="chat_history"),
        ("human", "{human_input} 알려줘")
    ]
)
# ChatPromptTemplate(input_variables=['chat_history', 'human_input'], input_types={'chat_history': list[typing.Annotated[typing.Union[typing.Annotated[langchain_core.messages.ai.AIMessage, Tag(tag='ai')], typing.Annotated[langchain_core.messages.human.HumanMessage, Tag(tag='human')], typing.Annotated[langchain_core.messages.chat.ChatMessage, Tag(tag='chat')], typing.Annotated[langchain_core.messages.system.SystemMessage, Tag(tag='system')], typing.Annotated[langchain_core.messages.function.FunctionMessage, Tag(tag='function')], typing.Annotated[langchain_core.messages.tool.ToolMessage, Tag(tag='tool')], typing.Annotated[langchain_core.messages.ai.AIMessageChunk, Tag(tag='AIMessageChunk')], typing.Annotated[langchain_core.messages.human.HumanMessageChunk, Tag(tag='HumanMessageChunk')], typing.Annotated[langchain_core.messages.chat.ChatMessageChunk, Tag(tag='ChatMessageChunk')], typing.Annotated[langchain_core.messages.system.SystemMessageChunk, Tag(tag='SystemMessageChunk')], typing.Annotated[langchain_core.messages.function.FunctionMessageChunk, Tag(tag='FunctionMessageChunk')], typing.Annotated[langchain_core.messages.tool.ToolMessageChunk, Tag(tag='ToolMessageChunk')]], FieldInfo(annotation=NoneType, required=True, discriminator=Discriminator(discriminator=<function _get_type at 0x7d10cdc151c0>, custom_error_type=None, custom_error_message=None, custom_error_context=None))]]}, partial_variables={}, messages=[SystemMessagePromptTemplate(prompt=PromptTemplate(input_variables=[], input_types={}, partial_variables={}, template='당신는 사용자의 가장 친한 친구입니다. 친구 처럼 대화를 이어가세요.'), additional_kwargs={}), MessagesPlaceholder(variable_name='chat_history'), HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['human_input'], input_types={}, partial_variables={}, template='{human_input} 알려줘'), additional_kwargs={})])

# 완성된 메시지 문자열로 확인
formatted_chat_prompt = chat_prompt.format(
    human_input="내가 가장 좋아하는 음식",
    chat_history=[
        ("human", "철수야 반가워"),
        ("ai", "안녕!"),
        ("human", "우리 10살 때 내가 가장 좋아하는 햄버거 먹었던 거 기억나 ?"),
        ("ai", "당연하지, 너 혼자서 2개씩이나 먹었잖아"),
        ("human", "우리 햄버거 먹고 수영장도 갔었어"),
        ("ai", "맞아 내가 너 수영 알려줬었어")
    ]
)
# System: 당신는 사용자의 가장 친한 친구입니다. 친구 처럼 대화를 이어가세요.\nHuman: 철수야 반가워\nAI: 안녕!\nHuman: 우리 10살 때 내가 가장 좋아하는 햄버거 먹었던 거 기억나 ?\nAI: 당연하지, 너 혼자서 2개씩이나 먹었잖아\nHuman: 우리 햄버거 먹고 수영장도 갔었어\nAI: 맞아 내가 너 수영 알려줬었어\nHuman: 내가 가장 좋아하는 음식 알려줘

chain = chat_prompt | model | StrOutputParser()

chain.invoke({
    "human_input":"내가 가장 좋아하는 음식",
    "chat_history":[
        ("human", "철수야 반가워"),
        ("ai", "안녕!"),
        ("human", "우리 10살 때 내가 가장 좋아하는 햄버거 먹었던 거 기억나 ?"),
        ("ai", "당연하지, 너 혼자서 2개씩이나 먹었잖아"),
        ("human", "우리 햄버거 먹고 수영장도 갔었어"),
        ("ai", "맞아 내가 너 수영 알려줬었어")
    ]
})
# 햄버거! 당연히 햄버거야, 그때 2개 먹었잖아 ㅋㅋㅋㅋ
```  


---  
## Reference
[Prompt Templates](https://python.langchain.com/docs/concepts/prompt_templates/)  
[PromptTemplate](https://python.langchain.com/api_reference/core/prompts/langchain_core.prompts.prompt.PromptTemplate.html)  
[ChatPromptTemplate](https://python.langchain.com/api_reference/core/prompts/langchain_core.prompts.chat.ChatPromptTemplate.html)  
[01. 프롬프트(Prompt)](https://wikidocs.net/233351)  



