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
