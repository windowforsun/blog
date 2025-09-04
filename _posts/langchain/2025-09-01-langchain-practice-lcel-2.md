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

