--- 
layout: single
classes: wide
title: "[LangChain] LangChain Prompt"
header:
  overlay_image: /img/langchain-bg-2.png
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


