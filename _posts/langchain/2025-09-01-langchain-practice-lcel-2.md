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
