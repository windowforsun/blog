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

