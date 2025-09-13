--- 
layout: single
classes: wide
title: "[LangChain] LangChain LCEL 2nd"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain 프레임워크 내에서 체인, 프롬프트, 모델의 연결과 조작을 더 쉽고 강력하게 만들어주는 도구적 언어인 LCEL 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
    - LCEL
    - Expression Language
    - configurable_fields
    - configurable_alternatives
    - RunnableWithMessageHistory
    - Runnable Graph
    - Chain Decorator
    - Custom Generator
    - Runnable Arguments Binding
toc: true
use_math: true
---  

## LangChain Expression Language



### configurable_fields
`configurable_fields` 는 `Runnable` 의 내부 구성요소를 외부에서 동적으로 구성/변경할 수 있도록 지정하는 속성이다. 
체인을 만들 때, 특정 필드/파라미터를 고정하지 않고 런타임 시점 또는 체인 생성 시점에 유연하게 값을 설정할 수 있도록 한다. 
이를 활용하면 동일한 `Runnable`(체인) 구조에 다양한 입력/옵션/환경을 쉽게 적용할 수 있다. 

`configurable_fields` 의 주요 특징은 아래와 같다. 

- 동적 파라미터화 : `체인/Runnable` 의 일부 속성을 런타임에 외부에서 변경 가능
- 유연한 재사용 : 동일한 체인 구조를 다양한 상황/환경/입력에 맞춰 재사용 가능
- 코드 간소화 : 여러 옵션/환경에 맞는 체인 클래스를 일일이 만들 필요 없음
- 빠른 실험 : 파라미터 변경을 통한 실험(A/B 테스팅, 하이퍼파라미터 탐색 등)이 쉬움

모델을 생성할때 `configurable_fields` 를 사용하면 모델의 종류 및 제공처, 하이퍼파라미터 등을 런타임에 수정할 수 있다. 

```python
from langchain.chat_models import init_chat_model
from langchain_core.runnables import ConfigurableField
from langchain_core.prompts import PromptTemplate
import os

os.environ["GROQ_API_KEY"] = "api key"
model = init_chat_model("llama-3.3-70b-versatile", model_provider="groq").configurable_fields(
    model_name=ConfigurableField(
        id="version",
        name="version of llm",
        description="offical model name"
    )
)

# 처음에 지정한 llama-3.3-70b-versatile 모델 사용
model.invoke("1+1 은?").__dict__
# {'content': '1+1 은 2입니다.',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 9,
#                                        'prompt_tokens': 40,
#                                        'total_tokens': 49,
#                                        'completion_time': 0.032727273,
#                                        'prompt_time': 0.003357174,
#                                        'queue_time': 0.205331267,
#                                        'total_time': 0.036084447},
#                        'model_name': 'llama-3.3-70b-versatile',
#                        'system_fingerprint': 'fp_3f3b593e33',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--778a623e-5d4b-431f-96d5-ceb1b220de9a-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 40,
#                     'output_tokens': 9,
#                     'total_tokens': 49}}

# 런타임에 llama3-8b-8192 모델로 변경해서 실행
model.invoke(
    "1+1 은?",
    config={'configurable' : {'version': 'llama3-8b-8192'}}
).__dict__
# {'content': '😊\n\n1 + 1 = 2',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 11,
#                                        'prompt_tokens': 15,
#                                        'total_tokens': 26,
#                                        'completion_time': 0.009166667,
#                                        'prompt_time': 0.002428753,
#                                        'queue_time': 0.08527328299999999,
#                                        'total_time': 0.01159542},
#                        'model_name': 'llama3-8b-8192',
#                        'system_fingerprint': 'fp_dadc9d6142',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--4dc98968-be8d-4b13-9ea1-28c192b59e18-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 15,
#                     'output_tokens': 11,
#                     'total_tokens': 26}}

# with_config() 를 사용해서도 모델을 변경할 수 있다. 
model.with_config(configurable={'version':'gemma2-9b-it'}).invoke('1+1 은?').__dict__
# {'content': '1 + 1은 2 입니다. 😄\n',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 13,
#                                        'prompt_tokens': 14,
#                                        'total_tokens': 27,
#                                        'completion_time': 0.024148918,
#                                        'prompt_time': 0.002076425,
#                                        'queue_time': 0.065609686,
#                                        'total_time': 0.026225343},
#                        'model_name': 'gemma2-9b-it',
#                        'system_fingerprint': 'fp_10c08bf97d',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--ed2eb76d-acce-4701-9a35-e13654f577c8-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 14,
#                     'output_tokens': 13,
#                     'total_tokens': 27}}
```  

모델 뿐만아니라 체인에도 `configurable_fields` 를 사용할 수 있다.  

```python
prompt = PromptTemplate.from_template("{query} 에 대해 100자 이내로 설명하세요.")

chain = (prompt | model)

# 모델 생성시 설정한 llama-3.3-70b-versatile 사용
chain.invoke({"query" : "langchain"}).__dict__
# {'content': 'LangChain은 언어 모델과 AI를 활용한 개발 플랫폼입니다.',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 18,
#                                        'prompt_tokens': 48,
#                                        'total_tokens': 66,
#                                        'completion_time': 0.096358423,
#                                        'prompt_time': 0.002315397,
#                                        'queue_time': 0.20693972,
#                                        'total_time': 0.09867382},
#                        'model_name': 'llama-3.3-70b-versatile',
#                        'system_fingerprint': 'fp_3f3b593e33',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--760ab497-ad29-485c-bd45-8f8e04db5a52-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 48,
#                     'output_tokens': 18,
#                     'total_tokens': 66}}

# 체인 실행 시점에 gemma2-9b-it 모델로 변경해서 실행
chain.with_config(configurable={'version' : 'gemma2-9b-it'}).invoke({'query':'langchain'}).__dict__
# {'content': 'LangChain은 대화형 AI 앱을 구축하기 위한 프레임워크입니다. \n\n텍스트 생성, 질의응답, 요약, 번역 등 다양한 자연어 처리(NLP) 작업을 위한 툴과 모듈을 제공하며, 외부 데이터와 통합하여 강력한 챗봇이나 인공지능 애플리케이션을 개발할 수 있습니다. \n',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 93,
#                                        'prompt_tokens': 24,
#                                        'total_tokens': 117,
#                                        'completion_time': 0.169090909,
#                                        'prompt_time': 0.002118285,
#                                        'queue_time': 0.08310903,
#                                        'total_time': 0.171209194},
#                        'model_name': 'gemma2-9b-it',
#                        'system_fingerprint': 'fp_10c08bf97d',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--385cc6f0-5e15-493f-86bc-07910c2277e1-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 24,
#                     'output_tokens': 93,
#                     'total_tokens': 117}}
```  

`HubRunnable` 과 `configurable_fields` 를 사용하면 `LangChain Hub` 에 있는 다양한 프롬프트를 
상황에 맞게 쉽게 변경해 사용할 수 있다. 

```python
from langchain.runnables.hub import HubRunnable

prompt = HubRunnable("hardkothari/text_summary").configurable_fields(
    owner_repo_commit=ConfigurableField(
        id="hub_commit",
        name="Hub Commit",
        description="descroption"
    )
)
prompt.invoke(
    {
        'word_count':100,
        'target_audience' :
            'children', 'text': '오늘 밥을 먹고 맛이 있어 레시피를 블로그에 작성해 올렸다'
    }
)
# ChatPromptValue(messages=[SystemMessage(content='You are an expert summarizer and analyzer who can help me.', additional_kwargs={}, response_metadata={}), HumanMessage(content="Generate a concise and coherent summary from the given Context. \n\nCondense the context into a well-written summary that captures the main ideas, key points, and insights presented in the context. \n\nPrioritize clarity and brevity while retaining the essential information. \n\nAim to convey the context's core message and any supporting details that contribute to a comprehensive understanding. \n\nCraft the summary to be self-contained, ensuring that readers can grasp the content even if they haven't read the context. \n\nProvide context where necessary and avoid excessive technical jargon or verbosity.\n\nThe goal is to create a summary that effectively communicates the context's content while being easily digestible and engaging.\n\nSummary should NOT be more than 100 words for children audience.\n\nCONTEXT: 오늘 밥을 먹고 맛이 있어 레시피를 블로그에 작성해 올렸다\n\nSUMMARY: ", additional_kwargs={}, response_metadata={})])

# with_config 를 사용해 프롬프트 변경
prompt = prompt.with_config(
    configurable={'hub_commit' : 'langchain/summary-memory-summarizer'}
)
prompt.invoke(
    {
        'summary' : '오늘 밥을 먹고 맛이 있어 레시피를 블로그에 작성해 올렸다',
        'new_lines' : '\n'
    }
)
# StringPromptValue(text='Progressively summarize the lines of conversation provided, adding onto the previous summary returning a new summary.\n\nEXAMPLE\nCurrent summary:\nThe human asks what the AI thinks of artificial intelligence. The AI thinks artificial intelligence is a force for good.\n\nNew lines of conversation:\nHuman: Why do you think artificial intelligence is a force for good?\nAI: Because artificial intelligence will help humans reach their full potential.\n\nNew summary:\nThe human asks what the AI thinks of artificial intelligence. The AI thinks artificial intelligence is a force for good because it will help humans reach their full potential.\nEND OF EXAMPLE\n\nCurrent summary:\n오늘 밥을 먹고 맛이 있어 레시피를 블로그에 작성해 올렸다\n\nNew lines of conversation:\n\n\n\nNew summary:')
```   


### configurable_alternatives
`configurable_alternatives` 는 하나의 `Runnable` 내에 여러 대인(`Alternative`)을 선언하고, 
실행 시점에 그 중 어떤 것을 사용할지 선택할 수 있게 해주는 속성이다. 
동잉ㄹ한 체인 구조에서 여러 대체 가능한 실행 경로(`LLM`, 프롬프트, 파서 등)를 필요에 따라 쉽게 바꿔가며 사용할 수 있는 기능이다.  

`configurable_alternatives` 의 주요 특징은 아래와 같다.

- 멀티 백엔드/옵션 지원 : 하나의 체인에서 여러 `LLM`, 프롬프트, 파서 등을 쉽게 바꿔치기
- 실행 경로 스위칭 : 상황에 맞게 다른 대체 경로로 바로 실행 가능
- 실험 및 롤백 : 여러 대안 체인을 쉽게 실험하고, 실패 시 빠르게 롤백 가능
- 코드 일관성 : 실행 경로를 외부 설정만으로 바꿀 수 있어 코드 수정 없이 유연성 강화 

가장 먼저 모델을 생성할 때 `configurable_alternatives` 를 사용하면 미리 정의한 대체안들을 상황에 따라 빠르게 사용할 수 있다.  

```python
from langchain.prompts import PromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.runnables import ConfigurableField

model = init_chat_model(
    "llama-3.3-70b-versatile", model_provider="groq"
).configurable_alternatives(
    ConfigurableField(id='llm'),
    default_key='versatile',
    gemma2=init_chat_model("gemma2-9b-it", model_provider="groq"),
    llama3=init_chat_model("llama3-8b-8192", model_provider="groq")
)

prompt = PromptTemplate.from_template('{query} 에 대해 100자 이내로 설명하세요.')

chain = (prompt | model)

# 체인 생성 시점에 지정한 llama-3.3-70b-versatile 모델 사용
chain.invoke({'query':'langchain'}).__dict__
# {'content': 'LangChain: 대화형 AI 플랫폼입니다.',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 14,
#                                        'prompt_tokens': 48,
#                                        'total_tokens': 62,
#                                        'completion_time': 0.077501423,
#                                        'prompt_time': 0.002599862,
#                                        'queue_time': 0.212741663,
#                                        'total_time': 0.080101285},
#                        'model_name': 'llama-3.3-70b-versatile',
#                        'system_fingerprint': 'fp_6507bcfb6f',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--b9339ba3-dd52-42b6-857b-b1ae587b4157-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 48,
#                     'output_tokens': 14,
#                     'total_tokens': 62}}

# 런타임에 gemma2-9b-it 모델로 변경해서 실행
chain.with_config(configurable={'llm':'gemma2'}).invoke({'query':'langchain'}).__dict__
# {'content': 'LangChain은 대규모 언어 모델(LLM)을 사용하여 앱을 구축하는 프레임워크입니다. \n\nLLM의 능력을 확장하고, 데이터베이스와 같은 외부 시스템과 연결하여 실용적인 애플리케이션을 만들 수 있도록 돕습니다. \n\n\n',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 75,
#                                        'prompt_tokens': 24,
#                                        'total_tokens': 99,
#                                        'completion_time': 0.136363636,
#                                        'prompt_time': 0.003373743,
#                                        'queue_time': 0.12341072400000001,
#                                        'total_time': 0.139737379},
#                        'model_name': 'gemma2-9b-it',
#                        'system_fingerprint': 'fp_10c08bf97d',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--805f5943-33d7-43f2-a45c-ddb8590eddec-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 24,
#                     'output_tokens': 75,
#                     'total_tokens': 99}}

# 런타임에 llama3-8b-8192 모델로 변경해서 실행
chain.with_config(configurable={'llm':'llama3'}).invoke({'query':'langchain'}).__dict__
# {'content': 'LangChain은 AI-powered language model platform입니다. 다양한 언어 모델을 지원하며, 텍스트 생성, 문서 생성, 대화bot 생성 등 다양한 기능을 제공합니다. LangChain은 기존의 NLP 기술을结合하여, 사용자에게 高品質의 언어 모델을 제공합니다. 또한, LangChain은 개방형 프레임워크를 제공하여, 개발자들이 새로운 기능을 추가하거나, 기존 기능을 개선할 수 있습니다.',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 99,
#                                        'prompt_tokens': 23,
#                                        'total_tokens': 122,
#                                        'completion_time': 0.0825,
#                                        'prompt_time': 0.003345114,
#                                        'queue_time': 0.085356573,
#                                        'total_time': 0.085845114},
#                        'model_name': 'llama3-8b-8192',
#                        'system_fingerprint': 'fp_179b0f92c9',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--1ac4f881-8d2d-48bb-acb3-ca9b40b9af42-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 23,
#                     'output_tokens': 99,
#                     'total_tokens': 122}}
```  

`configurable_alternatives` 는 프롬프트에도 적용해 미리 정의한 대체안들을 상황에 따라 빠르게 사용할 수 있다.  

```python

prompt = PromptTemplate.from_template(
    '{query} 에 대해 100자 이내로 설명하세요.'
).configurable_alternatives(
    ConfigurableField(id='prompt'),
    default_key='anwser',
    world_count=PromptTemplate.from_template('"{query}" 의 글자수를 알려주세요.'),
    translate=PromptTemplate.from_template('"{query}" 를 영어로 번역해주세요.')
)

chain = prompt | model

# 질문에 대한 답변 프롬프트 사용
chain.invoke({'query':'langchain'})
# AIMessage(content='LangChain: 언어 모델링을 위한 오픈소스 플랫폼입니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 21, 'prompt_tokens': 48, 'total_tokens': 69, 'completion_time': 0.103156962, 'prompt_time': 0.002634311, 'queue_time': 0.205285559, 'total_time': 0.105791273}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'stop', 'logprobs': None}, id='run--5cc88ad8-7728-4629-8fe7-52b095cc305d-0', usage_metadata={'input_tokens': 48, 'output_tokens': 21, 'total_tokens': 69})

# 글자수 프롬프트 사용
chain.with_config(configurable={'prompt':'world_count'}).invoke({'query': 'langchain'})
# AIMessage(content='"langchain"의 글자수는 9개입니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 15, 'prompt_tokens': 46, 'total_tokens': 61, 'completion_time': 0.054545455, 'prompt_time': 0.003765855, 'queue_time': 0.24772715199999998, 'total_time': 0.05831131}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'stop', 'logprobs': None}, id='run--bf1534fe-79fc-40b4-94a1-dabea34a7899-0', usage_metadata={'input_tokens': 46, 'output_tokens': 15, 'total_tokens': 61})

# 번역 프롬프트 사용
chain.with_config(configurable={'prompt':'translate'}).invoke({'query': '대한민국의 현대사'})
# AIMessage(content='"대한민국의 현대사"를 영어로 번역하면 "Modern history of South Korea" 또는 "Contemporary history of South Korea"로 번역할 수 있습니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 37, 'prompt_tokens': 50, 'total_tokens': 87, 'completion_time': 0.134545455, 'prompt_time': 0.002437236, 'queue_time': 0.207094754, 'total_time': 0.136982691}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--ea17a9e9-0f28-4f66-b03a-b8fa4de492a1-0', usage_metadata={'input_tokens': 50, 'output_tokens': 37, 'total_tokens': 87})
```  

앞서 먼저 모델에 대해 `configurable_alternatives` 를 사용해 대체안을 설정하고, 
프롬프트에도 `configurable_alternatives` 를 사용해 대체안을 설정했다. 
이제 런타임에 모델, 프롬프트 모두 필요에 따라 대체안을 선택해 실행하 수 있다.  

```python
chain.with_config(
    configurable={
        'llm':'gemma2',
        'prompt':'world_count'
    }
).invoke({'query': '대한민국의 현대사'}).__dict__
# {'content': '"대한민국의 현대사"의 글자 수는 **11글자**입니다. \n\n\n* **대한민국:** 6글자\n* **의:** 1글자\n* **현대사:** 4글자 \n',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 57,
#                                        'prompt_tokens': 27,
#                                        'total_tokens': 84,
#                                        'completion_time': 0.103636364,
#                                        'prompt_time': 0.002136625,
#                                        'queue_time': 0.083296095,
#                                        'total_time': 0.105772989},
#                        'model_name': 'gemma2-9b-it',
#                        'system_fingerprint': 'fp_10c08bf97d',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--e7dc0395-4cdc-44cc-b8df-f0cd048f7b46-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 27,
#                     'output_tokens': 57,
#                     'total_tokens': 84}}

chain.with_config(
    configurable={
        'llm':'llama3',
        'prompt':'translate'
    }
).invoke({'query': '대한민국의 현대사'}).__dict__
# {'content': 'The phrase "대한민국의 현대사" can be translated to English as "Modern History of South Korea" or "Contemporary History of South Korea".',
#  'additional_kwargs': {},
#  'response_metadata': {'token_usage': {'completion_tokens': 32,
#                                        'prompt_tokens': 25,
#                                        'total_tokens': 57,
#                                        'completion_time': 0.026666667,
#                                        'prompt_time': 0.009414188,
#                                        'queue_time': 1.986721293,
#                                        'total_time': 0.036080855},
#                        'model_name': 'llama3-8b-8192',
#                        'system_fingerprint': 'fp_179b0f92c9',
#                        'finish_reason': 'stop',
#                        'logprobs': None},
#  'type': 'ai',
#  'name': None,
#  'id': 'run--3046f45b-71ce-4de1-b362-d62488914ae9-0',
#  'example': False,
#  'tool_calls': [],
#  'invalid_tool_calls': [],
#  'usage_metadata': {'input_tokens': 25,
#                     'output_tokens': 32,
#                     'total_tokens': 57}}
```  




### RunnableWithMessageHistory
`RunnableWithMessageHistory` 는 `Runnable` 에 대화 히스토리를 자동으로 관리/주입하여 연속적인 대화 흐름을 자연스럽게 구현할 수 있게 한다. 
현재 입력값과 함께 과거 주고받은 메시지를 `LLM`, 프롬프트 등 다음 단계에 자동으로 넘겨
연속된 대화 컨텍스트를 유지하며 응답을 생성하는 역할을 한다.  

`RunnableWithMessageHistory` 는 아래와 같은 경우 사용할 수 있다.

- 챗봇, 멀티턴, 대화, 컨텍스트가 중요한 `LLM` 워크플로우를 만들 때 
- 대화 이력을 자동 관리해, 매번 직접 전달하지 않고도 자연스러운 대화 흐름을 원할 때 
- 사용자별, 세션별로 독립적인 대화 상태를 유지해야 할 때 
- 연속된 질의응답/상황 기반 `LLM` 파이프라인을 설계할 때

메시지 기록 저장은 메모리, 로컬 저장소, 외부(`Redis`) 저장소에 저장할 수 있다. 
먼저 `BaseChatMessageHistory` 를 사용해 메모리에 저장해 히스토리를 보존하는 방법을 알아본다. 
`BaseChatMessageHistory` 는 메시지 기록을 관리하기 위한 객체로, 메시지 기록을 저장, 검색, 업데이트하는 데 사용된다. 
메시지 기록은 대화의 맥락을 유지하고 사용자의 이전 입력에 기반한 응답을 생성하는데 도움을 준다.  

메모리 내에서 메시지 기록을 관리하기 위해 `ChatMessageHistory` 라는 `BaseChatMessageHistory` 의 구현체를 사용한다.  

```python
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

model = init_chat_model(
    "llama-3.3-70b-versatile", model_provider="groq"
)
prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "당신은 {ability} 에 능숙한 전문사 어시스턴트입니다. 20자 이내로 답변하세요."
        ),
        # 대화 기록용 변수
        MessagesPlaceholder(variable_name="history"),
        ("human", "{input}")
    ]
)

chain = prompt | model
```  

위 체인을 사용해 메모리에 메시지 기록을 괸라하도록 설정한다.  

```python
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_core.chat_history import BaseChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

store = {}

def get_session_history(session_ids: str) -> BaseChatMessageHistory:
  print(f"id : {session_ids}")

  if session_ids not in store:
    store[session_ids] = ChatMessageHistory()

  return store[session_ids]

with_message_history = (
    RunnableWithMessageHistory(
        chain,
        get_session_history,
        # 입력 메시지로 처리될 키
        input_message_key="input",
        # 이전 메시지를 추가할 키
        history_messages_key="history"
    )
)

with_message_history.invoke(
    {"ability" : "IT", "input" : "LangChain 에 대해 요약해서 설명해줘"},
    config={'configurable':{'session_id' : 1}}
)
# id : 1
# AIMessage(content='LLaMA와 같은 AI 모델을 활용하여 개발자들이 더 쉽게 개발할 수 있도록 도와주는 프레임워크입니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 30, 'prompt_tokens': 72, 'total_tokens': 102, 'completion_time': 0.154697818, 'prompt_time': 0.003717552, 'queue_time': 0.205502147, 'total_time': 0.15841537}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--721c9729-53e1-4397-83da-270f15c2a60f-0', usage_metadata={'input_tokens': 72, 'output_tokens': 30, 'total_tokens': 102})

print(store)

with_message_history.invoke(
    {'ability': 'IT', 'input' : '이전 답변을 영어로 답변해줘'},
    config={'configurable':{'session_id' : 1}}
)
# id : 1
# AIMessage(content='LangChain is a framework that helps developers build AI applications using models like LLaMA.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 19, 'prompt_tokens': 121, 'total_tokens': 140, 'completion_time': 0.084681832, 'prompt_time': 0.008310919, 'queue_time': 0.24781782900000002, 'total_time': 0.092992751}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_9a8b91ba77', 'finish_reason': 'stop', 'logprobs': None}, id='run--baa45c79-3800-4ca4-808c-2fba6c9940a0-0', usage_metadata={'input_tokens': 121, 'output_tokens': 19, 'total_tokens': 140})

print(store)
# {1: InMemoryChatMessageHistory(messages=[HumanMessage(content='LangChain 에 대해 요약해서 설명해줘', additional_kwargs={}, response_metadata={}), AIMessage(content='LLaMA와 같은 AI 모델을 활용하여 개발자들이 더 쉽게 개발할 수 있도록 도와주는 프레임워크입니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 30, 'prompt_tokens': 72, 'total_tokens': 102, 'completion_time': 0.154697818, 'prompt_time': 0.003717552, 'queue_time': 0.205502147, 'total_time': 0.15841537}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--721c9729-53e1-4397-83da-270f15c2a60f-0', usage_metadata={'input_tokens': 72, 'output_tokens': 30, 'total_tokens': 102})])}
```  

이전 대화 내용을 `store` 에 관리하기 때문에 이전 답변 맥락을 유지하며 질의를 수행할 수 있다.   


앞선 예제에서는 메시지 기록을 추적하고 관리하는 키로 `session_id` 를 사용했다. 
이는 별다른 설정을 하지 않으면 기본을 사용되는 키로 필요하다면 아래와 같이 커스텀이 가능하다. 
아래는 `user_id` 와 `conversation_id` 2개의 키로 메시지 기록을 관리하는 예제이다.  

```python
from langchain_core.runnables import ConfigurableFieldSpec

store_2 = {}

def get_session_history_2(user_id: str, conversation_id: str) -> BaseChatMessageHistory:
  if (user_id, conversation_id) not in store_2:
    store_2[(user_id, conversation_id)] = ChatMessageHistory()

  return store_2[(user_id, conversation_id)]

with_message_history_2 = RunnableWithMessageHistory(
    chain,
    get_session_history_2,
    input_messages_key='input',
    history_messages_key='history',
    history_factory_config=[
        ConfigurableFieldSpec(
            id='user_id',
            annotation=str,
            name="User ID",
            description='사용자 식별자',
            default="",
            is_shared=True
        ),
        ConfigurableFieldSpec(
            id='conversation_id',
            annotation=str,
            name='Conversation ID',
            description='대화 식별자',
            default='',
            is_shared=True
        )
    ]
)

with_message_history_2.invoke(
    {'ability':'IT', 'input' : '"LangChain 에 대해 요약해서 설명해줘'},
    config={
        'configurable':{
            'user_id' : 'user1',
            'conversation_id' : 'conv1'
        }
    }
)
# AIMessage(content='.LangChain은 AI와 프로그래밍을 연결합니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 14, 'prompt_tokens': 73, 'total_tokens': 87, 'completion_time': 0.061840539, 'prompt_time': 0.005160955, 'queue_time': 0.205330767, 'total_time': 0.067001494}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--cb5f68dd-54a2-4416-94f1-367faa6024d0-0', usage_metadata={'input_tokens': 73, 'output_tokens': 14, 'total_tokens': 87})

with_message_history_2.invoke(
    {'ability':'IT', 'input' : '이전 답변을 영어로 번역해줘'},
    config={
        'configurable':{
            'user_id' : 'user1',
            'conversation_id' : 'conv1'
        }
    }
)
# AIMessage(content='LangChain connects AI and programming.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 8, 'prompt_tokens': 107, 'total_tokens': 115, 'completion_time': 0.029090909, 'prompt_time': 0.005804578, 'queue_time': 0.207296262, 'total_time': 0.034895487}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_9a8b91ba77', 'finish_reason': 'stop', 'logprobs': None}, id='run--87a73fb2-7748-44ed-a9aa-2db9724f8906-0', usage_metadata={'input_tokens': 107, 'output_tokens': 8, 'total_tokens': 115})
```  

`RunnableWithMessageHistory` 를 사용할 때 입력과 출력의 형태를 필요에 따라 조정하며 사용할 수 있다. 
아래는 `Message` 객체를 입력으로 사용하고 결과는 딕셔너리로 받는 예제이다.  

```python
from langchain_core.messages import HumanMessage
from langchain_core.runnables import RunnableParallel

chain_2 = RunnableParallel({'output_message' : model})

with_message_history_3 = RunnableWithMessageHistory(
    chain_2,
    get_session_history,
    # 입력으로 Message 객체를 넣기 때문에 별도로 input_message_key 를 지정하지 않는다. 
    output_message_key='output_message'
)

with_message_history_3.invoke(
    [HumanMessage(content='langchain 에 대해 요약해서 설명해줘')],
    config={'configurable': {'session_id' : 's1'}}
)
# {'output_message': AIMessage(content='Langchain은 대규모 언어 모델(Large Language Model, LLM)과 같은 인공지능 기술을 쉽게 사용하고 확장할 수 있는 프레임워크입니다. \n\nLangchain은 다음과 같은 특징을 가지고 있습니다.\n\n1. **언어 모델 통합**: Langchain은 다양한 언어 모델을 지원하여 개발자가 쉽게 자신의 프로젝트에 통합할 수 있습니다.\n2. **사용자 정의 가능**: Langchain은 개발자가 자신의 언어 모델을 정의하고, 학습하고, 평가할 수 있는 강력한 도구를 제공합니다.\n3. **확장성**: Langchain은 대규모 데이터셋과 복잡한 모델을 처리할 수 있는 확장성 높은 아키텍처를 가지고 있습니다.\n4. **시각화 도구**: Langchain은 개발자가 모델의 성능을 시각화하고, 분석할 수 있는 도구를 제공합니다.\n\nLangchain을 사용하면 개발자는 다음과 같은 일들을 쉽게 할 수 있습니다.\n\n* 대규모 언어 모델을 쉽게 통합하고 사용할 수 있습니다.\n* 자신의 언어 모델을 정의하고, 학습하고, 평가할 수 있습니다.\n* 모델의 성능을 시각화하고, 분석할 수 있습니다.\n\nLangchain은 자연어 처리, 대화 시스템, 텍스트 생성 등 다양한 분야에서 유용하게 사용될 수 있습니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 288, 'prompt_tokens': 46, 'total_tokens': 334, 'completion_time': 1.047272727, 'prompt_time': 0.003877805, 'queue_time': 0.20691707399999998, 'total_time': 1.051150532}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--6fd79132-ff13-4509-b5f2-4323cf8c2dd1-0', usage_metadata={'input_tokens': 46, 'output_tokens': 288, 'total_tokens': 334})}

with_message_history_3.invoke(
    [HumanMessage(content='이전 답변을 영어로 번역해줘')],
    config={'configurable': {'session_id' : 's1'}}
)
# {'output_message': AIMessage(content='Langchain is a framework that allows for easy use and extension of artificial intelligence technologies such as large language models (LLMs).\n\nLangchain has the following features:\n\n1. **Language Model Integration**: Langchain supports various language models, making it easy for developers to integrate them into their projects.\n2. **Customizability**: Langchain provides powerful tools for developers to define, train, and evaluate their own language models.\n3. **Scalability**: Langchain has a highly scalable architecture that can handle large datasets and complex models.\n4. **Visualization Tools**: Langchain provides tools for developers to visualize and analyze the performance of their models.\n\nBy using Langchain, developers can easily:\n\n* Integrate and use large language models\n* Define, train, and evaluate their own language models\n* Visualize and analyze the performance of their models\n\nLangchain can be useful in various fields such as natural language processing, conversational systems, and text generation.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 193, 'prompt_tokens': 354, 'total_tokens': 547, 'completion_time': 0.701818182, 'prompt_time': 0.052687606, 'queue_time': 0.248290259, 'total_time': 0.754505788}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'stop', 'logprobs': None}, id='run--8b7d9f42-ab8c-4248-b6f1-7028d083e49c-0', usage_metadata={'input_tokens': 354, 'output_tokens': 193, 'total_tokens': 547})}
```  

다음은 `Message` 객체를 입력으로 사용하고, `Message` 객체를 출력으로 받는 예제이다. 


```python
with_message_history_4 = RunnableWithMessageHistory(
    model,
    get_session_history
    # 입력으로 Message 객체를 넣기 때문에 별도로 input_message_key 를 지정하지 않는다. 
    # output_message_key 도 지정하지 않으면 기본적으로 Message 객체로 출력된다.
)

with_message_history_4.invoke(
    [HumanMessage(content='langchain 에 대해 요약해서 설명해줘')],
    config={'configurable':{'session_id':'s2'}}
)
# AIMessage(content='LangChain은 인공지능과 블록체인 기술을 결합하여 개발된 플랫폼입니다. LangChain은 사용자에게 더智能적이고 자동화된 서비스를 제공하기 위해 개발되었습니다. LangChain의 주요 특징은 다음과 같습니다:\n\n1. **인공지능 통합**: LangChain은 다양한 인공지능 모델을 통합하여 사용자에게 더 정확하고智能적인 서비스를 제공합니다.\n2. **블록체인 기반**: LangChain은 블록체인 기술을 기반으로 하여 데이터의 보안성과 투명성을 제공합니다.\n3. **자동화**: LangChain은 자동화된 프로세스를 통해 사용자의 요청을 처리하여 효율성을 높입니다.\n4. ** 확장성**: LangChain은 확장성이 뛰어나므로 사용자 수의 증가에 따라 쉽게 확장할 수 있습니다.\n\nLangChain은 여러 분야에서 응용될 수 있습니다. 예를 들어, LangChain을 사용하여 다음과 같은 서비스를 개발할 수 있습니다:\n\n*智能적인 고객 서비스 챗봇\n* 자동화된 데이터 분석 및 보고 시스템\n* 보안성이 높은 데이터 저장 및 관리 시스템\n* 개인화된 추천 시스템\n\nLangChain은 개발자와 사용자 모두에게 편리하고 효율적인 서비스를 제공하는 플랫폼입니다. 그러나 LangChain의详细한 기능과 사용법은 더 연구하고 학습해야 합니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 297, 'prompt_tokens': 46, 'total_tokens': 343, 'completion_time': 1.08, 'prompt_time': 0.003504893, 'queue_time': 0.247403755, 'total_time': 1.083504893}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_9a8b91ba77', 'finish_reason': 'stop', 'logprobs': None}, id='run--7f524edf-b771-4536-8461-571679e345d2-0', usage_metadata={'input_tokens': 46, 'output_tokens': 297, 'total_tokens': 343})

with_message_history_4.invoke(
    [HumanMessage(content='이전 답변을 영어로 번역해줘')],
    config={'configurable':{'session_id':'s2'}}
)
# AIMessage(content='LangChain is a platform that combines artificial intelligence and blockchain technology. It was developed to provide users with more intelligent and automated services. The main features of LangChain are:\n\n1. **Artificial Intelligence Integration**: LangChain integrates various AI models to provide users with more accurate and intelligent services.\n2. **Blockchain-based**: LangChain is based on blockchain technology, providing security and transparency for data.\n3. **Automation**: LangChain processes user requests through automated processes, increasing efficiency.\n4. **Scalability**: LangChain is highly scalable, making it easy to expand as the number of users increases.\n\nLangChain can be applied in various fields. For example, LangChain can be used to develop the following services:\n\n* Intelligent customer service chatbots\n* Automated data analysis and reporting systems\n* Secure data storage and management systems\n* Personalized recommendation systems\n\nLangChain is a platform that provides convenient and efficient services for both developers and users. However, more research and learning are needed to understand the detailed features and usage of LangChain.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 210, 'prompt_tokens': 363, 'total_tokens': 573, 'completion_time': 0.763636364, 'prompt_time': 0.023587732, 'queue_time': 0.205756393, 'total_time': 0.787224096}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_9a8b91ba77', 'finish_reason': 'stop', 'logprobs': None}, id='run--4a2719b4-6322-44b3-93c4-96b4e3a524bb-0', usage_metadata={'input_tokens': 363, 'output_tokens': 210, 'total_tokens': 573})
```  

마지막으로 입력과 출력을 모두 딕셔너리 형태로 사용하는 예제이다. 

```python
from operator import itemgetter

with_message_history_5 = RunnableWithMessageHistory(
    # 입력으로 들어오는 딕셔너리에서 input_message 키를 사용해 모델에 전달한다. 
    itemgetter('input_message') | model,
    get_session_history,
    # 입력 메시지로 사용할 키를 지정한다. 
    input_messages_key='input_message'
)

with_message_history_5.invoke(
    {'input_message' : 'langchain 에 대해 요약해서 설명해줘'},
    config={'configurable':{'session_id':'s3'}}
)
# AIMessage(content='Langchain은 인공지능(AI) 기반의 자연어 처리(NLP) 플랫폼입니다. Langchain은 사용자와 대화형으로 상호작용하며, 사용자의 입력을 받아서 이해하고, 해당하는 답변을 제공합니다.\n\nLangchain의 주요 기능은 다음과 같습니다:\n\n1. **자연어 이해**: Langchain은 자연어를 이해하고, 이를 기반으로 사용자의 의도와 요구를 파악합니다.\n2. **대화형 상호작용**: Langchain은 사용자와 대화형으로 상호작용하며, 사용자의 질문이나 요청에 답변을 제공합니다.\n3. **문서 생성**: Langchain은 사용자의 요청에 따라 문서를 생성할 수 있습니다.\n4. **번역**: Langchain은 다중 언어를 지원하며, 사용자의 언어를 자동으로 번역할 수 있습니다.\n\nLangchain은 다양한 분야에서 활용될 수 있습니다. 예를 들어, 고객 서비스, 교육, 의료 등에서 사용될 수 있습니다. 또한, Langchain은 개발자들이 인공지능 기반의 애플리케이션을 쉽게 개발할 수 있도록 도와주는 도구입니다.\n\nLangchain의 장점은 다음과 같습니다:\n\n1. **고유한 아키텍처**: Langchain은 고유한 아키텍처를 갖고 있으며, 이는 다른 플랫폼과 차별화됩니다.\n2. **고성능**: Langchain은 고성능을 제공하며, 빠른 속도로 사용자의 요청을 처리할 수 있습니다.\n3. **다양한 언어 지원**: Langchain은 다중 언어를 지원하며, 사용자의 언어를 자동으로 번역할 수 있습니다.\n\nLangchain은 계속해서 발전하고 있으며, 새로운 기능과 성능을 추가하여 사용자에게 더 좋은 서비스를 제공하고 있습니다.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 386, 'prompt_tokens': 46, 'total_tokens': 432, 'completion_time': 1.403636364, 'prompt_time': 0.00249485, 'queue_time': 0.20549788, 'total_time': 1.406131214}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_9a8b91ba77', 'finish_reason': 'stop', 'logprobs': None}, id='run--bc9d121f-b849-4bd0-9aa2-d78b3fdd5f00-0', usage_metadata={'input_tokens': 46, 'output_tokens': 386, 'total_tokens': 432})

with_message_history_5.invoke(
    {'input_message' : '이전 답변을 영어로 번역해줘'},
    config={'configurable':{'session_id':'s3'}}
)
# AIMessage(content="Here is the translation of the previous answer:\n\nLangchain is an artificial intelligence (AI) based natural language processing (NLP) platform. Langchain interacts with users in a conversational manner, understanding their input and providing relevant answers.\n\nThe main features of Langchain are as follows:\n\n1. **Natural Language Understanding**: Langchain understands natural language and identifies the user's intent and requirements.\n2. **Conversational Interaction**: Langchain interacts with users in a conversational manner, providing answers to their questions or requests.\n3. **Document Generation**: Langchain can generate documents based on user requests.\n4. **Translation**: Langchain supports multiple languages and can automatically translate user language.\n\nLangchain can be applied in various fields, such as customer service, education, and healthcare. Additionally, Langchain is a tool that helps developers easily develop AI-based applications.\n\nThe advantages of Langchain are as follows:\n\n1. **Unique Architecture**: Langchain has a unique architecture that differentiates it from other platforms.\n2. **High Performance**: Langchain provides high performance and can process user requests quickly.\n3. **Multi-Language Support**: Langchain supports multiple languages and can automatically translate user language.\n\nLangchain is continuously evolving, adding new features and performance to provide better services to users.", additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 258, 'prompt_tokens': 452, 'total_tokens': 710, 'completion_time': 0.938181818, 'prompt_time': 0.028555753, 'queue_time': 0.206971923, 'total_time': 0.966737571}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'stop', 'logprobs': None}, id='run--c1fcd36a-adf8-4608-b26e-dc7d371caaf1-0', usage_metadata={'input_tokens': 452, 'output_tokens': 258, 'total_tokens': 710})
```  

이번에는 메모리에 메시지 기록을 관리하는 것이 아닌 `Redis` 와 같은 외부 저장소에 영속성이 있도록 메시지 기록을 관리하는 방법에 대해 알아본다. 
이때는 `BaseChatMessageHistory` 의 구현체인 `RedisChatMessageHistory` 를 사용한다.  

```python
from langchain_community.chat_message_histories import RedisChatMessageHistory

REDIS_URL = "redis://...."

def get_redis_message_history(session_id: str) -> RedisChatMessageHistory:
  return RedisChatMessageHistory(session_id, url=REDIS_URL)

redis_with_message_history = RunnableWithMessageHistory(
    chain,
    get_redis_message_history,
    input_messages_key='input',
    history_messages_key='history'
)

redis_with_message_history.invoke(
    {'ability':'IT', 'input':'langchain 에 대해 요약해서 설명해줘'},
    config={'configurable':{'session_id':'rs1'}}
)
# AIMessage(content='LLaMA 에서 파생된 대화형 AI 프레임워크', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 17, 'prompt_tokens': 72, 'total_tokens': 89, 'completion_time': 0.095742698, 'prompt_time': 0.003685556, 'queue_time': 0.248086022, 'total_time': 0.099428254}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--bbe91fec-ee2b-494a-a9d0-43aaca41c96e-0', usage_metadata={'input_tokens': 72, 'output_tokens': 17, 'total_tokens': 89})

redis_with_message_history.invoke(
    {'ability':'IT', 'input':'이전 답변을 영어로 번역해줘'},
    config={'configurable':{'session_id':'rs1'}}
)
# AIMessage(content='A conversational AI framework derived from LLaMA.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 12, 'prompt_tokens': 109, 'total_tokens': 121, 'completion_time': 0.050341463, 'prompt_time': 0.008292484, 'queue_time': 0.248558983, 'total_time': 0.058633947}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--7d7c0fb9-82ba-45d7-a747-9fe90b341035-0', usage_metadata={'input_tokens': 109, 'output_tokens': 12, 'total_tokens': 121})

redis_with_message_history.invoke(
    {'ability':'IT', 'input':'이전 답변을 영어로 번역해줘'},
    config={'configurable':{'session_id':'rs9999999'}}
)
# AIMessage(content='There is no previous answer.', additional_kwargs={}, response_metadata={'token_usage': {'completion_tokens': 7, 'prompt_tokens': 72, 'total_tokens': 79, 'completion_time': 0.042750819, 'prompt_time': 0.004015454, 'queue_time': 0.206939182, 'total_time': 0.046766273}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_6507bcfb6f', 'finish_reason': 'stop', 'logprobs': None}, id='run--dbc37055-edf3-4b0f-a892-5dd77aa86e46-0', usage_metadata={'input_tokens': 72, 'output_tokens': 7, 'total_tokens': 79})
```  

이후 `Redis` 에 접속해 키를 확인하면 아래와 같이 `session_id` 로 사용한 키가 생성된 것을 확인할 수 있다.  

```bash
$ redis-cli -u redis://...

redis> keys *
1) "message_store:rs9999999"
2) "message_store:rs1"
```  


### Runnable Graph
앞서 살펴 본 다양한 `LCEL` 을 사용해 체인을 구성했을 때 
전체의 내부 구조와 실행 흐름을 그래프 형태로 시각화/분석할 수 있도록 체인의 구성(노드/엣지 등)을 
객체로 반환하는 메서드이다. 
복잡한 `LCEL` 체인이 어떻게 연결(조합)되어 있는지 
노드와 엣지 단위로 그래프 객체로 표현해 구조 파악, 디버깅, 시각화, 문서화 등에 활용할 수 있게 한다.  

그래프 예제를 위해 아래와 같이 `LCEL` 체인을 구성한다. 
`RAG` 를 사용해 질의에 대한 답변을 생성하는 체인이다.  

```python
from langchain_chroma import Chroma
from langchain_huggingface.embeddings import HuggingFaceEndpointEmbeddings
from langchain_huggingface.embeddings import HuggingFaceEmbeddings
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough


os.environ["HUGGINGFACEHUB_API_TOKEN"] = "api key"
model_name = "BM-K/KoSimCSE-roberta"
hf_endpoint_embeddings = HuggingFaceEndpointEmbeddings(
    model=model_name,
    huggingfacehub_api_token=os.environ["HUGGINGFACEHUB_API_TOKEN"],
)

hf_embeddings = HuggingFaceEmbeddings(
    model_name=model_name,
    encode_kwargs={'normalize_embeddings':True},
)

vectorstore = Chroma.from_texts(
    [
        "사과는 초록",
        "바나나는 빨강",
        "딸기는 파랑",
        "수박은 노랑",
        "토마토는 검정"
    ],
    embedding=hf_embeddings,
)

retriever = vectorstore.as_retriever()

template = """
다음 내용만 고려해 질의에 맞는 답변을 제공하세요.

context: {context}

question: {question}
"""

prompt = ChatPromptTemplate.from_template(template)

chain = (
    {'context' : retriever, 'question' : RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)
```  

`chain.get_graph()` 메서드는 체인의 실행 그래프를 반환한다. 
그래프의 노드는 체인의 각 단계를 나타내며, 엣지는 각 단계 간의 데이터 흐름을 나타낸다. 
먼저 `chain.get_graph().nodes` 을 통해 체인의 그래프에서 노드 정보를 가져올 수 있다.  

```python
chain.get_graph().nodes
# {'9a07e3cfde514a1fbe3dca8428549a2a': Node(id='9a07e3cfde514a1fbe3dca8428549a2a', name='Parallel<context,question>Input', data=<class 'langchain_core.runnables.base.RunnableParallel<context,question>Input'>, metadata=None),
# '9509fef1025d403ab6208a1c4d3138b8': Node(id='9509fef1025d403ab6208a1c4d3138b8', name='Parallel<context,question>Output', data=<class 'langchain_core.utils.pydantic.RunnableParallel<context,question>Output'>, metadata=None),
# '62bacd224938463d814b617b3cf0aeb0': Node(id='62bacd224938463d814b617b3cf0aeb0', name='VectorStoreRetriever', data=VectorStoreRetriever(tags=['Chroma', 'HuggingFaceEmbeddings'], vectorstore=<langchain_chroma.vectorstores.Chroma object at 0x7df1b6b30890>, search_kwargs={}), metadata=None),
# '78ab3ed409e64de7b50b16f2dfbc3258': Node(id='78ab3ed409e64de7b50b16f2dfbc3258', name='Passthrough', data=RunnablePassthrough(), metadata=None),
# 'bb09dcd82dc34c27b4136350397229f3': Node(id='bb09dcd82dc34c27b4136350397229f3', name='ChatPromptTemplate', data=ChatPromptTemplate(input_variables=['context', 'question'], input_types={}, partial_variables={}, messages=[HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['context', 'question'], input_types={}, partial_variables={}, template='\n다음 내용만 고려해 질의에 맞는 답변을 제공하세요. \n\ncontext: {context}\n\nquestion: {question}\n'), additional_kwargs={})]), metadata=None),
# 'b5b63cec5cf34fa39c89a1aede6516dc': Node(id='b5b63cec5cf34fa39c89a1aede6516dc', name='ChatGroq', data=ChatGroq(client=<groq.resources.chat.completions.Completions object at 0x7df2fb576150>, async_client=<groq.resources.chat.completions.AsyncCompletions object at 0x7df2fb577110>, model_name='llama-3.3-70b-versatile', model_kwargs={}, groq_api_key=SecretStr('**********')), metadata=None),
# 'b39f13acf8044e3db3319cb375fe556b': Node(id='b39f13acf8044e3db3319cb375fe556b', name='StrOutputParser', data=StrOutputParser(), metadata=None),
# '93cd8025d0f44686bbd4bcd04dac927e': Node(id='93cd8025d0f44686bbd4bcd04dac927e', name='StrOutputParserOutput', data=<class 'langchain_core.output_parsers.string.StrOutputParserOutput'>, metadata=None)}
```  

`chain.get_graph().edges` 는 체인의 그래프에서 엣지 정보를 가져올 수 있다.  

```python
chain.get_graph().edges
# [Edge(source='fd4b91571d7647db989794ebe69acf4b', target='78eecced4041474ba0766aaade312ec9', data=None, conditional=False),
#  Edge(source='78eecced4041474ba0766aaade312ec9', target='28a9109205c6472682f3836f66b90c75', data=None, conditional=False),
#  Edge(source='fd4b91571d7647db989794ebe69acf4b', target='91bcfaf624f745fcaa0688bc43054461', data=None, conditional=False),
#  Edge(source='91bcfaf624f745fcaa0688bc43054461', target='28a9109205c6472682f3836f66b90c75', data=None, conditional=False),
#  Edge(source='28a9109205c6472682f3836f66b90c75', target='4925ab0ceb034ade86d275c973eff3e6', data=None, conditional=False),
#  Edge(source='4925ab0ceb034ade86d275c973eff3e6', target='db3878b3487e4474994ebd9aa80596ec', data=None, conditional=False),
#  Edge(source='aaa556d794a147e484e7e4be04f53d40', target='d488eda0ce414775866ab1707428ae3e', data=None, conditional=False),
#  Edge(source='db3878b3487e4474994ebd9aa80596ec', target='aaa556d794a147e484e7e4be04f53d40', data=None, conditional=False)]
```  

시각화된 그래프 형태로 확인하고 싶은 경우 `chain.get_graph().print_ascii()` 메서드를 사용하면 된다.  

```python
chain.get_graph().print_ascii()
#           +---------------------------------+        
#           | Parallel<context,question>Input |        
#           +---------------------------------+        
#                    ***             **                
#                  **                  ***             
#                **                       **           
# +----------------------+            +-------------+  
# | VectorStoreRetriever |            | Passthrough |  
# +----------------------+            +-------------+  
#                    ***             **                
#                       **        ***                  
#                         **    **                     
#           +----------------------------------+       
#           | Parallel<context,question>Output |       
#           +----------------------------------+       
#                             *                        
#                             *                        
#                             *                        
#                  +--------------------+              
#                  | ChatPromptTemplate |              
#                  +--------------------+              
#                             *                        
#                             *                        
#                             *                        
#                       +----------+                   
#                       | ChatGroq |                   
#                       +----------+                   
#                             *                        
#                             *                        
#                             *                        
#                   +-----------------+                
#                   | StrOutputParser |                
#                   +-----------------+                
#                             *                        
#                             *                        
#                             *                        
#                +-----------------------+             
#                | StrOutputParserOutput |             
#                +-----------------------+
```  

마지막으로 `chain.get_prompts()` 메서드를 사용하면 체인에서 사용되는 프롬프트의 정보를 가져올 수 있다.  

```python
chain.get_prompts()
# [ChatPromptTemplate(input_variables=['context', 'question'], input_types={}, partial_variables={}, messages=[HumanMessagePromptTemplate(prompt=PromptTemplate(input_variables=['context', 'question'], input_types={}, partial_variables={}, template='\n다음 내용만 고려해 질의에 맞는 답변을 제공하세요. \n\ncontext: {context}\n\nquestion: {question}\n'), additional_kwargs={})])]
```  


### @chain Decorator
`@chain` 데코레이터는 `LCEL` 에서 일반적인 파이썬 함수를 `LCEL` 의 `Runnable` 체인 객체로 변환해주는 데코레이터이다. 
이는 `RunnableLambda` 로 래핑하는 것과 기능적으로 동일하다. 
기존의 함수를 몇 줄의 코드 수정 없이 `LCEL` 파이프라인의 한 단계로 쉽게 조립/확장할 수 있게 해주는 `함수 -> 체인(Runnable`)` 자동 변환 장치이다.  

`@chain` 데코레이터는 아래와 같은 경우 사용할 수 있다. 

- 기존 파이썬 함수/로직을 `LCEL` 체인에 손쉽게 통합하고 싶을 때 
- 함수형 처리(전처리, 후처리, 조건부 처리 등)를 체인 구성에 자연스럽게 녹이고 싶을 때 
- 직접 `Runnable` 클래스를 만들지 않고도, 빠르게 체인 요소를 추가하고 싶을 때 
- `LCEL` 의 파이프라인 연산자(`|`)와 함께 사용해, `함수-프롬프트-LLM-파서` 등을 자연스럽게 조합하고 싶을 때 

`@chain` 활용 예시를 보이기 위해 아래와 같은 2개의 프롬프트를 사용한다.  

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import chain

prompt1 = ChatPromptTemplate.from_template("{query} 에 대해 짧게 설명하세요.")
prompt2 = ChatPromptTemplate.from_template("{setence} 를 emoji 를 사용해 꾸며주세요.")
```  

그리고 `@chain` 데코레이터로 사용자 저으이 함수를 데코레이팅 하여, 일반 파이썬 함수를 `Runnable` 객체로 변환한다.  

```python
@chain
def custom_chain(text):
  chain1 = prompt1 | model | StrOutputParser()
  output1 = chain1.invoke({"query" : text})

  chain2 = prompt2 | model | StrOutputParser()

  return chain2.invoke({"setence" : output1})
```  

`custom_chain` 은 앞서 정의된 2개의 프롬프트를 사용해서 사용자 질의에 대한 설명 답변을 생성하고, 
그 답변을 다시 이모지를 사용해 꾸미는 결과를 만들어낸다. 
그리고 `custom_chain` 은 실행 가능한 `Runnable` 객체이기 때문에 `invoke()` 를 사용해 실행할 수 있다.  

```python
custom_chain.invoke('langchain')
# 🤖 LangChain은 인공지능을 쉽게 사용할 수 있도록 도와주는 라이브러리입니다 📚. LangChain은 Python으로 작성되었으며 🐍, 자연어 처리(NLP) 📝, 대화 시스템 💬, 그리고 언어 모델을 위한 다양한 도구와 기능을 제공합니다 🎉. LangChain을 사용하면 개발자가 효율적으로 인공지능을 활용하여 다양한 애플리케이션을 개발할 수 있습니다 💻. 🚀 개발자들의 인공지능 활용을 쉽게 만들어주는 LangChain은 인공지능 개발의未来를 밝혀줄 것입니다 💫!
```


### Custom Generator
`Custom Generator` 는 파이썬의 `Generator` 기능(`yield`를 사용하는 함수)과 `LCEL` 의 체인(`Runnable`)시스템을 결합하여, 
데이터를 한 번에 모두 처리하는 것이 아니라 순차적(스트리밍) 생성하는 사용자 정의 실행 단위를 의미한다. 
입력값을 받아 처리 결과를 `yield` 를 통해 한 단계씩 반환하고 `LCEL` 파이프라인 내에서, 
대용량 처리/점진적 응답/실시간 피드백 등 다양한 스트림 기반 워크플로우를 구현할 때 핵심 역할을 한다. 

`Custom Generator` 는 아래와 같은 경우 사용할 수 있다.

- `LLM`, `API` 등에서 응답을 한 번에 모두 받지 않고, 토큰 단위/문장 단위로 점진적 출력이 필요할 때 
- 데용량 데이터를 일괄처리 하지 않고, 한 줄씩 처리/전송할 때
- 스트림 기반 파이프라인을 만들고 싶을 때 
- 메모리 사용을 최소화하며, 데이터 흐름을 효율적으로 제어하고 싶을 때 

또한 사용자 정의 출력 파서 구현 및 이전 단계 출력을 수정하면서 스트리밍 기능을 유지하고 싶을 때 사용할 수 있다. 

`Custom Generator` 예시를 위해 아래와 같은 체인을 구현한다.  

```python
from typing import Iterator, List
from langchain.prompts.chat import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template(
    "{query} 에 맞는 주요한 키워드 5개를 쉽표로 구분된 목록으로 작성하세요."
)

str_chain = prompt | model | StrOutputParser()
```  

`stream()` 과 `invoke()` 결과를 확인하면 아래와 같다.  

```python
for chunk in str_chain.stream({"query": "langchain"}):
    print(chunk, end="", flush=True)
# LLaMA, AI, 언어 모델, 인공지능, 챗봇

str_chain.invoke({"query":"langchain"})
# LLaMA, 인공지능, 채팅봇, 언어 모델, AI 플랫폼
```  

아래 `split_into_list` 함수는 사용자 제너레이터는 `LLM` 토큰의 `Iterator` 입력아로 받아 쉼표로 구분된 문자열 리스트의 `Iterator` 를 반환한다.  

```python
def split_into_list(input: Iterator[str]) -> Iterator[List[str]]:
    buffer = ""
    
    for chunk in input:
        buffer += chunk
        while "," in buffer:
            comma_index = buffer.index(",")
            yield [buffer[:comma_index].strip()]
            buffer = buffer[comma_index + 1 :]
            
    yield [buffer.strip()]
```  

`split_into_list` 사용자 제너레이터를 파이프(`|`) 연산자를 사용해 `str_chain` 에 연결한다.  

```python
list_chain = str_chain | split_into_list
```  

사용자 제너레이터와 연결된 체인을 `stream()` 과 `invoke()` 로 실행하면 아래와 같이 
`LLM` 의 응답을 쉼표로 구분된 리스트 형태로 변환한 결과를 확인할 수 있다.  

```python
for chunk in list_chain.stream({"query": "langchain"}):
    print(chunk, flush=True)
# ['LLaMA']
# ['언어 모델']
# ['인공지능']
# ['챗봇']
# ['AI']

list_chain.invoke({"query" : "langchain"})
# ['Large Language Model', '인공지능', '챗봇', '자연어 처리', '언어 모델링']
```  

`astream()` 과 `ainvoke()` 와 같이 비동기 함수를 사용한다면 아래와 같이 사용자 제너레이터를 구현해 사용할 수 있다.  

```python
from typing import AsyncIterator

async def asplit_into_list(input: AsyncIterator[str]) -> AsyncIterator[List[str]]:
    buffer = ""
    
    async for chunk in input:
        buffer += chunk
        while "," in buffer:
            comma_index = buffer.index(",")
            yield [buffer[:comma_index].strip()]
            buffer = buffer[comma_index + 1:]
            
    yield [buffer.strip()]

alist_chain = str_chain | asplit_into_list

async for chunk in alist_chain.astream({"query":"langchain"}):
    print(chunk, flush=True)
['AI']
['언어 모델']
['챗봇']
['자연어 처리']
['기계 학습']

await alist_chain.ainvoke({"query":"langchain"})
# ['LLaMA', 'AI', '챗봇', '자연어 처리', '프로그래밍']
```  

### Runnable Arguments Binding
`Runnable Arguents Binding` 은 `Runnable` 체인 실행 시점에 동적으로 인자(파라미터, 옵션 등)를 주입하여 
동일한 체인 구조라도 매번 다르게 동작하게 만드는 기능을 의미한다. 
체인 정의 시 고정된 값이 아니라 실행 시점에 외부에서 필요한 값을(프롬프트 변수, 모델 옵션 등)을 전달해 
각 단계별로 원하는 입력, 옵션, 변수, 설정 등을 유연하게 변경 가능하다. 

`Runnable Arguments Binding` 은 아래와 같은 경우 사용할 수 있다.

- 프롬프트 내 변수 값, `LLM` 파라미터, 체인 옵션 등을 실행할 때마다 다르게 지정하고 싶은 경우
- 동일한 파이프라인 구조로 다양한 입력값/환경/옵션을 실험하거나 운영하고 싶을 때
- 동적 워크플로우, 사용자 맞춤 실행, A/B 테스트 등 상황별로 체인 동작을 바꾸고 싶을 때
- 체인 재사용성, 유연성, 확장성을 극대화하고 싶을 때 


`Runnable Arguments Binding` 예시를 위해 아래와 같은 체인을 구현한다.
자연어로 수식을 작성하면 이를 방정식을 만들고 결과를 도출하는 프롬프트이다.  

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "대수 기호를 사용해 다음 방벙식을 작성한 다음 풀이하세요."
            "최종 결과는 아래를 따르세요."
            "수식 : .."
            "답 : ..."
        ),
        (
            "human",
            "{query}"
        )
    ]
)

runnable = (
    {"query" : RunnablePassthrough()} | prompt | model | StrOutputParser()
)

runnable.invoke({"query" : "x제곱 더하기 7은 12"})
# 수식 : x^2 + 7 = 12
# 답 : x = ±√5
```  

여기서 `bind()` 의 `stop` 파라미터를 사용해 최종 답변에서 특정 토큰까지만 출력하도록 설정할 수 있다. 
아래 예시는 `수식`까지만 출력하고 `답` 은 출력하지 않도록 설정한 것이다.  

```python
runnable_with_bind = (
    {"query" : RunnablePassthrough()}
    | prompt
    | model.bind(stop="답")
    | StrOutputParser()
)


runnable_with_bind.invoke({"query" : "x제곱 더하기 7은 12"})
# 수식 : x^2 + 7 = 12
```  

`binding` 의 유용한 활용으로는 `Functions` 기능을 연결하는 것이다. 
아래와 같이 `수식` 과 `답` 을 출력하는 `Function` 을 정의한다.  

```python
function = {
    "name" : "solver",
    "description" : "Formulates and solves an equation",
    "parameters" : {
        "type" : "object",
        "properties" : {
            "equation" : {
                "type" : "string",
                "description" : "The algebraic expression of the equation",
            },
            "solution" : {
                "type" : "string",
                "description" : "The solution to the equation"
            }
        },
        "required" : ["equation", "solution"]
    }
}
```  

그리고 `LLM` 모델에 `bind()` 메서드를 사용해 정의한 함수 호출을 모델에 바인딩한다.  

```python
function_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "대수 기호를 사용해 다음 방정식을 작성한 다음 풀이하세요."
        ),
        (
            "human",
            "{query}"
        )
    ]
)

function_runnable = model.bind(function_call={'name':'solver'}, functions=[function])

result = function_runnable.invoke("x제곱 더하기 7은 12")
# AIMessage(content='', additional_kwargs={'function_call': {'arguments': '{"equation": "x^2 + 7 = 12", "solution": "x = sqrt(5)"}', 'name': 'solver'}}, response_metadata={'token_usage': {'completion_tokens': 29, 'prompt_tokens': 279, 'total_tokens': 308, 'completion_time': 0.105454545, 'prompt_time': 0.024568217, 'queue_time': 0.231275051, 'total_time': 0.130022762}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'function_call', 'logprobs': None}, id='run--6bd7a989-da4d-4683-8304-38ef91f84a49-0', usage_metadata={'input_tokens': 279, 'output_tokens': 29, 'total_tokens': 308})

funciton_result = result.additional_kwargs['function_call']['arguments']
# {"equation": "x^2 + 7 = 12", "solution": "x = sqrt(5)"}
```  

또 다른 활용법으로는 `tools` 를 연결해 활용하는 방법이 있다. 
`tools` 를 정의해 모델에 바인딩하면 다양한 기능을 간편하게 사용할 수 있다. 
아래와 같은 지역의 날씨를 확인하는 `tool` 을 정의한다.  

```python
tools = [
    {
        "type" : "function",
        "function" : {
            "name" : "get_current_weather",
            "description" : "주어진 위치의 현재 날씨를 가져옵니다.",
            "parameters" : {
                "type" : "object",
                "properties" : {
                    "location" : {
                        "type" : "string",
                        "description " : "도시와 주, e.g) San Francisco, CA"
                    },
                    "unit" : {
                        "type" : "string",
                        "enum" : ["celsius", "fahrenheit"]
                    },
                }
            },
            "required" : ["location"]
        }
    }
]
```  

정의한 툴을 모델에 `bind()` 메서드를 사용해 바인딩한다. 
그리고 여러 지역을 질의에 넣어 질문하면 아래와 같이 툴에 맞는 결과를 얻을 수 있다.  

```python
tools_model = model.bind(tools=tools)

result = tools_model.invoke("서울, 샌프란시스코의 현재 날씨에 대해 알려줘")
# AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_d75x', 'function': {'arguments': '{"location":"서울", "unit":"celsius"}', 'name': 'get_current_weather'}, 'type': 'function'}, {'id': 'call_gw66', 'function': {'arguments': '{"location":"San Francisco, CA", "unit":"fahrenheit"}', 'name': 'get_current_weather'}, 'type': 'function'}]}, response_metadata={'token_usage': {'completion_tokens': 42, 'prompt_tokens': 268, 'total_tokens': 310, 'completion_time': 0.152727273, 'prompt_time': 0.023455216, 'queue_time': 0.23160565500000002, 'total_time': 0.176182489}, 'model_name': 'llama-3.3-70b-versatile', 'system_fingerprint': 'fp_3f3b593e33', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run--753185e7-32a9-4294-9dab-2ca6782e9f6f-0', tool_calls=[{'name': 'get_current_weather', 'args': {'location': '서울', 'unit': 'celsius'}, 'id': 'call_d75x', 'type': 'tool_call'}, {'name': 'get_current_weather', 'args': {'location': 'San Francisco, CA', 'unit': 'fahrenheit'}, 'id': 'call_gw66', 'type': 'tool_call'}], usage_metadata={'input_tokens': 268, 'output_tokens': 42, 'total_tokens': 310})

tool_result = result.additional_kwargs['tool_calls']
# [{'id': 'call_nph9',
#   'function': {'arguments': '{"location": "서울", "unit": "celsius"}',
#                'name': 'get_current_weather'},
#   'type': 'function'},
#  {'id': 'call_v519',
#   'function': {'arguments': '{"location": "San Francisco, CA", "unit": "fahrenheit"}',
#                'name': 'get_current_weather'},
#   'type': 'function'}]
```  



---  
## Reference
[Understanding LangChain Runnables](https://mirascope.com/blog/langchain-runnables/)  
[runnables](https://python.langchain.com/api_reference/core/runnables.html)  
[LCEL](https://wikidocs.net/233781)  

