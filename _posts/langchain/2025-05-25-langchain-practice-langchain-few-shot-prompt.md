--- 
layout: single
classes: wide
title: "[LangChain] LangChain FewShotPrompt"
header:
  overlay_image: /img/langchain-bg-2.jpg
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
    - Prompt
    - PromptTemplate
    - FewShotPromptTemplate
    - ExampleSelector
    - FewShotChatMessagePromptTemplate
toc: true
use_math: true
---  

## FewShotPromptTemplate
`FewShotPromptTemplate` 은 `LangChain` 에서 예시를 통해 모델에게 작업 방법을 가르치는 프롬프트 생성 방식이다. 
`Few-Shot Learning` 은 모델에게 몇 가지 예시를 제시하여 새로운 작업을 수행하도록 유도하는 방법을 의미한다. 
`FewShotPromptTemplate` 은 이런 예시 기반 프롬프트를 쉽게 생성할 수 있게 해준다. 

주요한 매개변수로는 아래와 같은 것들이 있다. 

- `examples` : 모델에게 보여줄 예시 목록(딕셔너리 리스트 형태)
- `example_prompt` : 각 예시의 형식을 정의하는 템플릿
- `prefix` : 예시 전에 표시할 텍스트(지시사항이나 설명)
- `suffix` : 예시 후에 표시할 텍스트(실제 질문이나 태스크)
- `input_variables` : 프롬프트 생성에 필요한 변수들
- `example_separator` : 각 예시 사이에 넣을 구분자(기본값 : `\n\n`

아래 예저는 문장을 보고 `긍정`과 `부정`을 `Few-Shot Learning` 방식으로 학습하는 방법을 보여준다. 

```python
from langchain_core.prompts.few_shot import FewShotPromptTemplate
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain.chat_models import init_chat_model
import getpass
import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_ENDPOINT"] = "https://api.smith.langchain.com"
os.environ["LANGCHAIN_PROJECT"] = "lagnchain-note-prompt"
os.environ["LANGCHAIN_API_KEY"] = getpass.getpass("Enter your LangSmith API key: ")
os.environ["GROQ_API_KEY"] = getpass.getpass("Enter your Groq API key: ")
model = init_chat_model("llama-3.3-70b-versatile", model_provider="groq")

# 예시 형식 템플릿 정의
example_template = """
입력 : {input}
출력 : {output}
"""

example_prompt = PromptTemplate.from_template(example_template)

# 예시 데이터
examples = [
    {"input": "오늘 날씨가 좋네요", "output": "긍정적"},
    {"input": "이 영화는 정말 별로였어요", "output": "부정적"},
    {"input": "그냥 보통이에요", "output": "중립적"},
    {"input": "길 가다가 비 맞았어요", "output": "부정적"},
    {"input": "길 가다가 반가운 친구를 만났어요", "output": "긍정적"},
    {"input": "물건이 떨어져 깨졌어요", "output": "부정적"},
    {"input": "오늘 아침에 커피를 쏟았어요", "output": "부정적"},
    {"input": "새로운 책을 읽기 시작했어요", "output": "긍정적"},
    {"input": "친구와 다퉜어요", "output": "부정적"},
    {"input": "맛있는 저녁을 먹었어요", "output": "긍정적"},
    {"input": "지갑을 잃어버렸어요", "output": "부정적"},
    {"input": "좋은 꿈을 꿨어요", "output": "긍정적"},
    {"input": "시험에서 좋은 점수를 받았어요", "output": "긍정적"},
    {"input": "버스를 놓쳤어요", "output": "부정적"},
    {"input": "새로운 취미를 시작했어요", "output": "긍정적"},
    {"input": "휴대폰이 고장났어요", "output": "부정적"},
    {"input": "오랜만에 가족을 만났어요", "output": "긍정적"},
    {"input": "컴퓨터가 느려졌어요", "output": "부정적"},
    {"input": "좋은 음악을 들었어요", "output": "긍정적"},
    {"input": "차가 막혀서 늦었어요", "output": "부정적"},
    {"input": "새로운 친구를 사귀었어요", "output": "긍정적"},
    {"input": "비가 와서 우울해요", "output": "부정적"},
    {"input": "운동을 해서 기분이 좋아요", "output": "긍정적"},
    {"input": "감기에 걸렸어요", "output": "부정적"},
    {"input": "좋은 영화를 봤어요", "output": "긍정적"},
    {"input": "인터넷이 끊겼어요", "output": "부정적"},
    {"input": "친구와 너무 즐거운 시간을 보냈어요", "output": "긍정적"}
]

# 프롬프트 생성
few_shot_prompt = FewShotPromptTemplate(
    examples = examples,
    example_prompt = example_prompt,
    prefix="다음은 텍스트 감정 분석의 예시입니다:",
    suffix="입력 : {input}\n출력:",
    input_variables = ["input"]
)

few_shot_prompt.format(input="지금 비가 쏟아져요")
# 다음은 텍스트 감정 분석의 예시입니다:\n\n\n입력 : 오늘 날씨가 좋네요\n출력 : 긍정적\n\n\n\n입력 : 이 영화는 정말 별로였어요\n출력 : 부정적\n\n\n\n입력 : 그냥 보통이에요\n출력 : 중립적\n\n\n\n입력 : 길 가다가 비 맞았어요\n출력 : 부정적\n\n\n\n입력 : 길 가다가 반가운 친구를 만났어요\n출력 : 긍정적\n\n\n\n입력 : 물건이 떨어져 깨졌어요\n출력 : 부정적\n\n\n\n입력 : 오늘 아침에 커피를 쏟았어요\n출력 : 부정적\n\n\n\n입력 : 새로운 책을 읽기 시작했어요\n출력 : 긍정적\n\n\n\n입력 : 친구와 다퉜어요\n출력 : 부정적\n\n\n\n입력 : 맛있는 저녁을 먹었어요\n출력 : 긍정적\n\n\n\n입력 : 지갑을 잃어버렸어요\n출력 : 부정적\n\n\n\n입력 : 좋은 꿈을 꿨어요\n출력 : 긍정적\n\n\n\n입력 : 시험에서 좋은 점수를 받았어요\n출력 : 긍정적\n\n\n\n입력 : 버스를 놓쳤어요\n출력 : 부정적\n\n\n\n입력 : 새로운 취미를 시작했어요\n출력 : 긍정적\n\n\n\n입력 : 휴대폰이 고장났어요\n출력 : 부정적\n\n\n\n입력 : 오랜만에 가족을 만났어요\n출력 : 긍정적\n\n\n\n입력 : 컴퓨터가 느려졌어요\n출력 : 부정적\n\n\n\n입력 : 좋은 음악을 들었어요\n출력 : 긍정적\n\n\n\n입력 : 차가 막혀서 늦었어요\n출력 : 부정적\n\n\n\n입력 : 새로운 친구를 사귀었어요\n출력 : 긍정적\n\n\n\n입력 : 비가 와서 우울해요\n출력 : 부정적\n\n\n\n입력 : 운동을 해서 기분이 좋아요\n출력 : 긍정적\n\n\n\n입력 : 감기에 걸렸어요\n출력 : 부정적\n\n\n\n입력 : 좋은 영화를 봤어요\n출력 : 긍정적\n\n\n\n입력 : 인터넷이 끊겼어요\n출력 : 부정적\n\n\n\n입력 : 친구와 너무 즐거운 시간을 보냈어요\n출력 : 긍정적\n\n\n입력 : 지금 비가 쏟아져요\n출력:

chain = few_shot_prompt | model | StrOutputParser()

chain.invoke("지금 비가 쏟아져요")
# 부정적
```  

### ExampleSelector
`ExampleSelector` 는 `FewShotPromptTemplate` 에서 상황에 가장 적합한 예시를 동적으로 선택하는 메커니즘이다. 
모든 예시를 사용하는 대신 특정 기준에 따라 최적의 예시만 선택하여 효율성과 효과를 높일 수 있다. 

`ExampleSelector` 를 사용하면 아래와 같은 장점이 있다. 

- 토큰 절약 : 모든 예시를 포함하면 토큰 한도를 초과할 수 있다. 
- 관련성 향상 : 현재 입력과 관련 있는 예시만 선택하여 모델 성능을 개선할 수 있다. 
- 동적 적용 : 입력에 따라 다른 예시를 선택하여 상활별로 최적화할 수 있다. 

주요한 `ExampleSelector` 유형에는 아래와 같은 것들이 있다. 

- `SemanticSimilarityExampleSelector` : 입력과 의미적으로 가장 유사한 예시를 선택한다. 문맥 이해가 중요한 태크스에 적합하다. 
- `LengthBasedExampleSelector` : 최대 토큰 수를 고려하여 예시를 선택한다. 많은 예시가 있지만 모드 포함할 수 없는 경우에 적합하다. 
- `NGramOverlapExampleSelector` : 입력과 가장 많은 n-gram을 공유하는 예시를 선택한다. 임베딩 모델이 없을 떄 간단한 유사도를 측정해 사용할 수 있다. 
- `MaxMarginalRelevanceExampleSelector` : 쿼리와의 관련성이 높으면서 동시에 선택된 다른 예시들과는 다양성을 유지하는 예시를 선택한다. 다양하면서 중복되는 예시를 피하고 싶은 경우 적합하다. 
- `CustomExampleSelector` : 사용자 정의 로직에 따라 예시를 선택한다. 특수한 선택 로직이 필요하거나, 기준 선택를 조합 및 확장하는 경우 적합하다. 

앞선 감정 분석 예제를 `ExampleSelector` 를 사용해 최적화할 수 있는데, 
이를 위해 임베딩과 벡터 저장소를 활용한다. 


```python
from langchain_community.vectorstores import Chroma
from langchain_nomic import NomicEmbeddings
from langchain_core.example_selectors import (MaxMarginalRelevanceExampleSelector, SemanticSimilarityExampleSelector)

chroma = Chroma("example_selector", embedding_function=NomicEmbeddings(model="nomic-embed-text-v1.5"))

example_selector = MaxMarginalRelevanceExampleSelector.from_examples(
    examples,
    NomicEmbeddings(model="nomic-embed-text-v1.5"),
    chroma,
    # 선택할 예시 수
    k=3
)

selected_examples = example_selector.select_examples({"input" : "친구와 조금 싸웠지만 잘 화해해서 더욱 친해지는 기회였어요"})
# [{'input': '친구와 더욱 친해진 것 같아요', 'output': '긍정적'},
# {'input': '친구와 너무 즐거운 시간을 보냈어요', 'output': '긍정적'},
# {'input': '친구와 다퉜어요', 'output': '부정적'}]

# 프롬프트 생성 및 확인
prompt = FewShotPromptTemplate(
    example_selector = example_selector,
    example_prompt = example_prompt,
    prefix="다음은 텍스트 감정 분석의 예시입니다:",
    suffix="입력 : {input}\n출력:",
    input_variables = ["input"]
)

example_selector_prompt = prompt.format(input="친구와 싸웠어요")
# 다음은 텍스트 감정 분석의 예시입니다:\n\n\n입력 : 친구와 싸웠어요.\n출력 : 부정적\n\n\n\n입력 : 친구와 다퉜어요\n출력 : 부정적\n\n\n\n입력 : 친구와 너무 즐거운 시간을 보냈어요\n출력 : 긍정적\n\n\n입력 : 친구와 싸웠어요\n출력:

chain = prompt | model

chain.invoke("친구와 조금 싸웠지만 잘 화해해서 더욱 친해지는 기회였어요").content
#  긍정적
```

### FewShotChatMessagePromptTemplate
`FewShotChatMessagePromptTemplate` 은 `LangChain` 에서 제공하는 특수한 프롬프트 템플릿으로, 
채팅 모델에 최적화된 방식으로 `few-shot learning` 을 적용할 수 있다. 
`FewShotPromptTemplate` 과 유사하지만, 채팅 모델의 메시지 형식에 맞게 설계됐다는 점에서 차이가 있다. 

채팅 기반 언어 모델에 여러 예시를 메시지 형식으로 제공하여 특정 패턴을 학십시키는데 사용된다. 
일반 텍스트 대신 채팅 메시지 형식(`human/ai')으로 예시를 제공한다. 
