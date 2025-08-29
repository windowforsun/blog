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
`LCEL` 은 `LangChain` 프레임워크 내에서 체인, 프롬프트, 모델의 연결과 조작을 더 쉽고 강력하게 만들어주는 도구적 언어이다. 
`LCEL` 에서 제공하는 표현식을 이용해 다양한 `LLM` 워크플로우를 선언적이고 직관적으로 구성할 수 있게 한다. 

주요 특징은 아래와 같다. 

- 간결함 : 복잡한 체인 구조를 간단한 문법으로 표현
- 표현력 : 다양한 체인 조합, 브랜칭, 반복, 동시 실행 등 쉽게 구현
- 유지보수성 : 파이프라인을 선언적으로 작성하여 코드를 더 읽기 쉽고 관리하기 쉽게 만듦
- 직관적 연결 : 다양한 컴포넌트(프롬프트, 모델, 출력 파서 등)를 함수처럼 연결(`|`)하여 구성

`LCEL` 의 기본적인 구성 요소는 다음과 같다. 

- `Runnable` : `LCEL` 의 모든 구성 요소의 기반이 되는 인터페이스이다. 
- `Chain` : 여러 `Runnable` 을 순차적으로 연결한 실행 파이프라인의 상위 개념이다. 
- `RunnableMap` : 여러 `Runnable` 을 병렬로 실행하여 여러 개의 결과를 동시에 반환한다. 
- `RunnableSequence` : 여러 `Runnable` 을 순차적으로 연결하는 객체이다. 
- `RunnableLambda` : 파이프라인 내에서 `Python` 함수 등 임의의 함수를 래핑하여 사용할 수 있도록 한다. 

`LCEL` 의 추가적인 구현체로는 다음과 같은 것들이 있다. 

- `RunnableBranch` : 입력 조건에 따라 여러 `Runnable` 중 하나를 선택하여 실행한다. 
- `RunnablePassthrough` : 입력을 그대로 다음 단계로 넘기는 역할을 한다. (특별한 처리 없이 전달)
- `RunnableParallel` : `RunaableMap` 과 유사하게 여러 `Runnable` 을 병렬로 실행하여 결과를 반환한다.
- `RunnableEach` : 리스트 형태의 입력에 대해 각 요소마다 동일한 `Runnable` 을 실행한다. 
- `RunnableRetry` : 실패 시 재시도 로직을 포함한 래퍼이다. 
- `RunnableWithFallbacks` : 실패 시 대체 `Runnable` 을 차례로 시도한다.  

사용 가능한 `runnables` 의 전체 목록은 [여기](https://python.langchain.com/api_reference/core/runnables.html)
에서 확인 가능하다.  

`LCEL` 을 사용하면 `프롬프트 -> LLM -> 파서` 와 같이 다양한 컴포넌트를 직렬로 간편하게 연결해 구성할 수 있다. 
모든 체인을 연결하는 방식을 사용하기 때문에 복잡한 처리도 `LCEL` 을 통해 간단하게 구현할 수 있다. 
간단한 사용 예시는 다음과 같다. 


```python
from langchain.chat_models import init_chat_model
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser

os.environ["GROQ_API_KEY"] = "api key"
model = model = init_chat_model("llama-3.3-70b-versatile", model_provider="groq")
prompt = ChatPromptTemplate.from_template("다음 질의 혹은 주제에 대해 설명해, 답변은 최소 30자 최대100자로: {query}")
output_parser = StrOutputParser()

chain = prompt | model | output_parser

result = chain.invoke({"query": "LangChain"})
# LangChain은 AI와 언어 모델을 위한 오픈 소스 플랫폼으로, 개발자들이 자신의 언어 모델을 쉽게 구축하고 배포할 수 있도록 지원합니다.
```  

`LCEL` 의 장점은 다음과 같다. 

- 가독성 : 복잡한 체인도 파이프 연산자로 직관적으로 표현
- 재사용성 : 각 컴포넌트(프롬프트, 모델 등)는 독립적으로 재사용 가능
- 유연성 : 조건 분기, 반복, 병렬 등 다양한 워크플로우 유연하게 구현
- 디버깅 용이 : 각 단계별로 결과를 쉽게 확인할 수 있음

### RunnableMap
`RunnableMap` 은 여러 개의 `Runnable` 을 병렬로 실행하여 각기 다른 결과를 딕셔너리 형태로 반환하는 역할을 한다. 
하나의 입력을 받아 여러 개 처리(분석, 생성 등)를 동시에 실행하고, 
각 다른 결과를 `key-value` 쌍으로 모아 최종적으로는 딕셔너리 형태로 반환한다.  

`RunnableMap` 은 아래와 같은 경우 사용할 수 있다.

- 동시에 여러 작업을 수행하고 싶을 때
- 중복 실행을 피하면서, 다양한 분석/생성을 병렬 처리하고 싶을 때
- 여러 `LLM` 체인 결과를 한 번에 모아서 관리하고 싶을 때 

아래는 자용자의 질의를 요약하는 결과와 감정을 분석하는 처리를 모두 수행하는 `RunnableMap` 의 예시이다. 

```python
from langchain_core.runnables import RunnableMap
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

summary_template = """당신은 요약 전문가 입니다. 사용자 질의를 10자 이내로 요약하세요. 

# 질의
{question}
"""
summary_prompt = ChatPromptTemplate.from_template(summary_template)

summary_chain = summary_prompt | model | StrOutputParser()

setiment_template = """당신은 감정분석 전문가 입니다. 사용자 질의를 긍정, 부정으로 분류하세요.

# 질의
{question}

# 답변은 반드시 긍정 혹은 부정으로만 답해야 합니다.
"""
sentiment_prompt = ChatPromptTemplate.from_template(setiment_template)

sentiment_chain = sentiment_prompt | model | StrOutputParser()

multi_chain = RunnableMap({
    "요약" : summary_chain,
    "감정분석" : sentiment_chain
})

result = multi_chain.invoke({"question" : "오늘 아침에 일어나 사과를 먹었는데 맛이 너무 좋아 친구들에게 알려줬어요."})
# {'요약': '사과 먹고 친구한테 말함', '감정분석': '긍정'}
```  


### RunnableSequence
`RunnableSequence` 는 여러 개의 `Runnable` 을 순차적으로 연결하여 실행하는 체인 역할을 한다. 
입력을 받아 첫 번째 `Runnable` 에 전달한 뒤 그 결과를 두 번째 `Runnable` 에 전달하고 마지막 `Runnable` 을 통해 최종 결과를 만드는 
파이프라인 또는 직렬 체인구조를 구현할 수 있다. 

`RunnableSequence` 는 아래와 같은 경우 사용할 수 있다.

- 여러 단계를 거쳐 데이터를 처리하고 싶을 때
- 단계별로 서로 다른 처리가 필요한 복합 워크플로우를 설계할 때
- 체인의 각 단계를 재사용하고 싶을 때 

아래는 사용자의 질의를 요약하고 그 결과를 영어로 번역하는 `RunnableSequence` 의 예제이다. 

```python
from langchain_core.runnables import RunnableSequence
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

summary_template = """당신은 요약 전문가 입니다. 사용자 질의를 10자 이내로 요약하세요.

# 질의
{question}
"""
summary_prompt = ChatPromptTemplate.from_template(summary_template)

summary_chain = summary_prompt | model | StrOutputParser()

translate_template = """한글을 영어로 번역하는 전문가입니다. 사용자의 질의를 영어로 번역하세요.

# 질의
{question}

"""
translate_prompt = ChatPromptTemplate.from_template(translate_template)

translate_chain = translate_prompt | model | StrOutputParser()

sequence_chain = RunnableSequence(
    first=summary_chain,
    last=translate_chain
)
# or equal
# sequence_chain = summary_chain | translate_chain

result = sequence_chain.invoke({"question" : "오늘 아침에 일어나 사과를 먹었는데 맛이 너무 좋아 친구들에게 알려줬어요."})
# I told my friend that the apple was delicious.
```


### RunnableSequence
`RunnableSequence` 는 여러 개의 `Runnable` 을 순차적으로 연결하여 실행하는 체인 역할을 한다.
입력을 받아 첫 번째 `Runnable` 에 전달한 뒤 그 결과를 두 번째 `Runnable` 에 전달하고 마지막 `Runnable` 을 통해 최종 결과를 만드는
파이프라인 또는 직렬 체인구조를 구현할 수 있다.

`RunnableSequence` 는 아래와 같은 경우 사용할 수 있다.

- 여러 단계를 거쳐 데이터를 처리하고 싶을 때
- 단계별로 서로 다른 처리가 필요한 복합 워크플로우를 설계할 때
- 체인의 각 단계를 재사용하고 싶을 때

아래는 사용자의 질의를 요약하고 그 결과를 영어로 번역하는 `RunnableSequence` 의 예제이다.

```python
from langchain_core.runnables import RunnableSequence
from langchain.prompts import ChatPromptTemplate
from langchain.chat_models import init_chat_model
from langchain_core.output_parsers import StrOutputParser

summary_template = """당신은 요약 전문가 입니다. 사용자 질의를 10자 이내로 요약하세요.

# 질의
{question}
"""
summary_prompt = ChatPromptTemplate.from_template(summary_template)

summary_chain = summary_prompt | model | StrOutputParser()

translate_template = """한글을 영어로 번역하는 전문가입니다. 사용자의 질의를 영어로 번역하세요.

# 질의
{question}

"""
translate_prompt = ChatPromptTemplate.from_template(translate_template)

translate_chain = translate_prompt | model | StrOutputParser()

sequence_chain = RunnableSequence(
    first=summary_chain,
    last=translate_chain
)
# or equal
# sequence_chain = summary_chain | translate_chain

result = sequence_chain.invoke({"question" : "오늘 아침에 일어나 사과를 먹었는데 맛이 너무 좋아 친구들에게 알려줬어요."})
# I told my friend that the apple was delicious.
```



### RunnableLambda
`RunnableLambda` 는 임의의 `Python` 함수를 체인 내에서 실행 가능한 `Runnable` 로 래핑하는 역할을 한다.
기존에 구현한 `Python` 함수나 한단한 `Lambda` 연산을 `LCEL` 파이프라인 안에 자연스럽게 편입시켜
체인의 한 단계로 활용할 수 있도록 도와준다.

`RunnableLambda` 는 아래와 같은 경우 사용할 수 있다.

- 데이터 전처리/후처리 : 입력값을 변환하거나, 출력값을 가공하고 싶을 때
- 체인 사이의 값 변환 : 프롬프트, 모델, 파서 사이에서 값의 형태를 맞출 필요가 있을 때
- 간단한 조건 분기, 필터링, 로직 삽입 : 복잡한 `Runnable` 을 따로 만들 필요 없이, `inline` 으로 간단한 처리를 하고 싶을 때
- 기존 `Python` 함수의 재사용 : 이미 만들어둔 함수를 체인에 그대로 넣고 싶을 때

먼저 사용자 정의 함수를 실행하는 예시에 대해 알아본다.
주의할 점은 사용자 정의 함수가 받을 수 있는 인자는 1개라는 점을 기억해야 한다.
여러 인수를 받는 함수를 구현하고 싶을 경우, 단일 입력을 받아들이고 이를 여러 인수로 풀어내는 래퍼 및 처리가 필요하다.


```python
from operator import itemgetter
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableLambda
from langchain_core.output_parsers import StrOutputParser
from langchain.chat_models import init_chat_model

def str_len(text):
  return len(text)

def _multiply_str_len(x, y):
  return str_len(x) * str_len(y)

def multiply_str_len(_dict):
  return _multiply_str_len(_dict['x'], _dict['y'])

prompt = ChatPromptTemplate.from_template('{input1} + {input2} = ?')
os.environ["GROQ_API_KEY"] = "api key"
model = model = init_chat_model("llama-3.3-70b-versatile", model_provider="groq")

prompt_chain = prompt | model

chain = (
    {
        'input1' : itemgetter('text1') | RunnableLambda(str_len),
        'input2' : {'x' : itemgetter('text1'), 'y' : itemgetter('text2')} | RunnableLambda(multiply_str_len),
    }
    | prompt
    | model
    | StrOutputParser() 
)

chain.invoke({'text1' : '안녕', 'text2' : '하세요'})
# 2 + 6 = 8.
```  

`text1` 의 `안녕` 은 `RunnableLambda(str_len)` 를 통해 문자열 길이 `2` 로 `input1` 에 전달된다.
그리고 `input2` 는 `RunnableLambda(multiply_str_len)` 을 통해 `안녕` 의 문자열 길이와 `하세요` 문자열 길이를 곱한 `6` 이 전달된다.




### RunnablePassthrough
`RunnablePassthrought` 는 입력받은 값을 아무런 변경 없이 그대로 다음 단계에 전달하는 역할을 한다.
이는 데이터를 입력 받아 가공이나 추가 처리 없이 그대로 파이프라인의 다음 단계로 전달하는데 사용 가능하다.

`RunnablePassthrought` 는 아래와 같은 경우 사용할 수 있다.

- 파이프라인 구성 중 일부 단계에서 입력을 그대로 넘겨야 할 때
- 조건부 분기(`brach`) 혹은 병렬(`map`) 구조에서 특정 경로에서 아무런 가공이 필요 업을 때
- 테스트나 디버깅 목적으로 값의 흐름을 명확히 하고 싶을 때
- 다른 `Runnable` 과의 인터페이스를 맞추기 위해

단순한 사용 예시로 `RunnableParallel` 과 함께 사용해 데이터를 그대로 전달하는 경우와 키를 추가하는 경우를 보여준다.

```python
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

runnable = RunnableParallel(
    # passed 속성은 입력값 그대로 전달
    passed=RunnablePassthrough(),
    # extra 속성은 입력값에 키를 추가하여 전달
    extra=RunnablePassthrough.assign(mult=lambda x : x['num'] * 10),
    # modified 속성은 입력값에 1을 더한 값을 전달
    modified=lambda x : x['num'] + 1
)

runnable.invoke({"num": 1})
# {'passed': {'num': 1}, 'extra': {'num': 1, 'mult': 10}, 'modified': 2}
```  

다음은 검색기(`RAG`)에서 `RunnablePassthrough` 를 사용하는 예시에 대해 알아본다.
아래 예제에서 `RunnablePassthrough` 는 검색 체인에서 `question` 키에 해당하는 값을 그대로 전달하기 위해 사용된다.
즉 사용자 질의에 해당하는 `question` 의 내용을 입력 그대로 아무런 가공 없이 그대로 `prompt` 에 전달하는 목적을 수행한다.

```python
from langchain_chroma import Chroma
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_huggingface.embeddings import HuggingFaceEndpointEmbeddings
from langchain_huggingface.embeddings import HuggingFaceEmbeddings
from langchain.chat_models import init_chat_model

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

template = """사용자 질의에 답변을 하세요. 답변은 반드시 아래 문서만 고래해야 합니다. 
# 문서
{context}

# 질의
{question}
"""

prompt = ChatPromptTemplate.from_template(template)
os.environ["GROQ_API_KEY"] = "api key"
model = model = init_chat_model("llama-3.3-70b-versatile", model_provider="groq")

def format_docs(docs):
  return "\n".join([doc.page_content for doc in docs])

retrieval_chain = (
    {'context' : retriever | format_docs, 'question' : RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

retrieval_chain.invoke('사과의 색과 rgb 를 알려줘')
# 사과의 색은 초록입니다. RGB 색상 코드에서 초록은(typically) RGB(0, 128, 0)으로 표현됩니다.

retrieval_chain.invoke('토마토의 색과 rgb 를 알려줘')
# 토마토의 색은 검정입니다. RGB 색상 코드에서 검정은(typically) RGB(0, 0, 0)으로 표현됩니다.
```  


### RunnableBranch
`RunnableBranch` 는 입력값에 따라 여러 개의 `Runnable` 중 하나를 선택적으로 실행하는 조건 분기 역할을 한다.
입력을 받아 미리 정의된 조건(함수)들을 순서대로 평가하고,
조건을 만족하는 첫 번째 `Runnable` 만 실행해 그 결과를 반환하는 조건분기 체인을 구현할 수 있다.

`RunnableBranch` 는 아래와 같은 경우 사용할 수 있다.

- 입력값에 따라 체인 실행 경로를 다르게 하고 싶을 경우
- `if-else`, `switch-case` 와 유사한 분기 처리가 필요할 때
- 특정 조건에 따라 서로 다른 `프롬프트/LLM/파서/함수` 를 실행하고 싶을 때
- 복잡한 워크플로우에서 로직 분기를 선언적으로 관리하고 싶을 때

이렇듯 `RunnableBranch` 는 입력 데이터에 따라 동적으로 로직을 분기할 수 있게 해주는 도구로,
복잡한 의사 결정 트리를 간단하게 구현할 수 있다.
이를 통해 코드의 가독성과 유지보수성이 향상되고, 모듈화와 재사용성을 높일 수 있다.
런타임에 분기 조건을 평가해 적절한 처리 루팅을 선택할 수 있어,
다양한 도메인과 데이터 변동성이 큰 애플리케이션에서 유용하게 활용할 수 있다.

`RunnableBranch` 의 주된 특징은 조건부 실행은 `RunnableBranch` 를 사용하지 않더라도 구현은 가능하다.
비교를 위해 사용하지 않고 구현하는 경우 아래와 같은 내용이 필요하다.

예제는 사용자 질의를 `수학`, `과학`, `IT`, `기타` 분류 하고,
각 분류에 맞는 체인을 조건부로 실행하는 예시이다.


```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import PromptTemplate
from operator import itemgetter
from langchain_core.runnables import RunnableLambda

classification_prompt = PromptTemplate.from_template(
    """
    주어진 사용자 질의에 대해 `수학`, `과학`, `IT`, `기타` 중 하나로 분류하세요. 한 단어 이상으로 응답하지 마세요.

    <question>
    {question}
    </question>

    Classification:
    """
)

classification_chain = (
    classification_prompt
    | model
    | StrOutputParser()
)

classification_chain.invoke({"question" : "1+1 은 ?"})
# 수학

classification_chain.invoke({"question" : "langchain 에 대해 알고 싶어"})
# IT

classification_chain.invoke({"question" : "빛의 속도로 태양까지 얼마나 걸릴까 ?"})
# 과학

classification_chain.invoke({"question" : "부동산 가격이 오를까 ?"})
# 기타


# 각 분류별 체인 정의
math_prompt = PromptTemplate.from_template(
    """
    당신은 수학 전문가 입니다. 수학관련 사용자 질의에 대해 답변하세요. 
    답변은 반드시 다음 문자으로 시작해야 합니다. "피타고라스께서 말씀하시길 .."

    Question: {question}
    Answer:
    """
)

math_chain = (
        math_prompt
        | model
        | StrOutputParser()
)

it_prompt = PromptTemplate.from_template(
    """
    당신은 IT 전문가 입니다. IT관련 사용자 질의에 대해 답변하세요. 
    답변은 반드시 다음 문자으로 시작해야 합니다. "빌게이츠께서 말씀하시길 .."

    Question: {question}
    Answer:
    """
)

it_chain = (
        it_prompt
        | model
        | StrOutputParser()
)

science_prompt = PromptTemplate.from_template(
    """
    당신은 과학 전문가 입니다. 과학관련 사용자 질의에 대해 답변하세요. 
    답변은 반드시 다음 문자으로 시작해야 합니다. "뉴턴께서 말씀하시길 .."

    Question: {question}
    Answer:
    """
)

science_chain = (
        science_prompt
        | model
        | StrOutputParser()
)

etc_prompt = PromptTemplate.from_template(
    """
    당신은 다양한 분야의 전문가 입니다. 사용자 질의에 대해 답변하세요. 
    답변은 반드시 다음 문자으로 시작해야 합니다. "슈퍼 컴퓨터께서 말씀하시길 .."

    Question: {question}
    Answer:
    """
)

etc_chain = (
        it_prompt
        | model
        | StrOutputParser()
)
```  

위 코드 까지는 `RunnableBranch` 를 사용하는 경우와 그렇지 않는 경우 모두 공통으로 사용되는 코드이다.


아래는 `RunnableBranch` 를 사용하지 않는 경우 어떻게 구현이 가능한지 살펴본다.

```python
# RunnableBranch 를 사용하지 않는 경우 필요한 조건 분기 함수
def route(info):
    if '수학' in info['topic'].lower():
        return math_chain
    elif '과학' in info['topic'].lower():
        return science_chain
    elif 'it' in info['topic'].lower():
        return it_chain
    else:
        return etc_chain

# RunnableBranch 를 사용하지 않고 구현한 조건분기 체인
full_chain = (
        {'topic' : classification_chain, 'question' : itemgetter('question')}
        | RunnableLambda(route)
        | StrOutputParser()
)

full_chain.invoke({'question' : '1 +1 은 ?'})
# 피타고라스께서 말씀하시길, 1 + 1은 2입니다. 이는 가장 기본적인 산수 연산 중 하나로, 두 개의 동일한 개체를 합쳤을 때의 총 개수를 계산하는 것을 의미합니다. 따라서, 1 + 1은 2라는 결과가 나옵니다.

full_chain.invoke({'question' : 'langchain 이 뭐야?'})
# 빌게이츠께서 말씀하시길, LangChain은 인공지능과 언어 모델을 활용하여 다양한 TASK를 수행할 수 있는 프레임워크입니다. 이는 사용자들이 자신의 언어 모델과 인공지능 시스템을 쉽게 통합하고, 관리할 수 있도록 설계되어 있습니다. LangChain은 다양한 라이브러리와 도구를 제공하여, 개발자들이 효율적으로 언어 모델을 개발하고, 활용할 수 있도록 지원합니다. 또한, LangChain은 다양한 용途에 적용될 수 있으며, 예를 들어 챗봇, 언어 번역, 문서 요약 등 다양한 TASK에 활용될 수 있습니다. 이는 인공지능과 언어 모델의 발전에 따라, 더욱 많은 응용 분야에서 사용될 것으로 예상됩니다.
```  

이어서 `RunnableBranch` 를 사용해서 구현할 때를 알아본다.

```python
from langchain_core.runnables import RunnableBranch

branch = RunnableBranch(
    (lambda x : '수학' in x['topic'].lower(), math_chain),
    (lambda x : '과학' in x['topic'].lower(), science_chain),
    (lambda x : 'it' in x['topic'].lower(), it_chain),
    etc_chain
)

branch_full_chain = (
    {'topic' : classification_chain, 'question' : itemgetter('question')}
    | branch
    | StrOutputParser()
)

branch_full_chain.invoke({'question' : '1+1 은 ?'})
# 피타고라스께서 말씀하시길, 1+1 은 기본적인 산수 연산의 결과로서 2로 계산됩니다. 이는 수학의 기본 원리 중 하나로, 두 개의 단위를 합쳤을 때 그 결과는 2가 됩니다. 따라서, 1+1의 결과는 2입니다.

branch_full_chain.invoke({'question' : 'langchain 이 뭐야?'})
# 빌게이츠께서 말씀하시길, LangChain은 인공지능과 언어모델을 활용하여 다양한 태스크를 수행할 수 있는 프레임워크입니다. LangChain은 주로 대규모 언어 모델(LLM, Large Language Model)과 같은 인공지능 모델을 활용하여 텍스트 생성, 번역, 요약, 질의 응답 등 다양한 언어 관련 태스크를 처리할 수 있습니다. LangChain의 주요 특징은 사용자가 쉽게 언어 모델을 통합하고, 커스텀한 태스크를 수행할 수 있는 플랫폼을 제공한다는 것입니다. 이를 통해 개발자들은 인공지능과 언어 모델을更加 효율적으로 활용할 수 있습니다.

branch_full_chain.invoke({'question' : '빛의 속도로 태양까지 가면 얼마나 걸려?'})
# 뉴턴께서 말씀하시길, 빛의 속도는 대략 초당 약 299,792,000 미터입니다. 태양까지의 평균 거리는 약 1.496억 킬로미터(149,600,000 km)입니다. 이를 바탕으로 계산을 해보면, 빛이 태양까지 가는 데 걸리는 시간은 약 500초, 즉 약 8분 20초 정도가 됩니다. 이 계산은 태양과 지구의 평균 거리를 기준으로 하며, 실제로 태양과 지구의 거리는 궤도상에서 약 1.38억 킬로미터에서 1.52억 킬로미터까지変化합니다. 따라서, 실제 시간은 약 8분 3초에서 8분 38초 사이로變化할 수 있습니다.
```


### RunnableParallel
`RunnableParallel` 은 여러 `Runnable` 을 동시에 병렬로 실행하는 역할을 한다.
하나 또는 여러 입력을 받아 각 `Runnable` 에 병렬로 전달하고,
각 `Runnable` 의 결과를 리스트, 튜플 또는 딕셔너리 등으로 모아 한 번에 반환한다.
이는 `RunnableMap` 과 유사하지만 `RunnableParallel` 은 리스트/튜플 기반(순서 중심)으로 결과를 반환하는 특징이 있다.
반면 `RunnableMap` 은 `key-value`(딕셔너리)로 결과를 반환한다.

`RunnableParallel` 은 아래와 같은 경우 사용할 수 있다.

- 동일한 입력을 여러 `Runnable` 에 동시에 전달해 결과를 나란히 받고 싶을 때
- 여러 체인을 병렬로 실행해 순서대로 결과를 받고 싶을 때
- 여러 `LLM/프롬프트/파서`의 결과를 일괄 비교하거나 합쳐서 활용하고 싶을 때
- 입력값이 리스트/튜플일 때 각 값마다 다른 `Runnable` 을 적용하고 싶을 때

`RunnableParallel` 은 알게 모르게 기존 예제에서도 많이 사용 됐는데 아래 예제를 살펴본다.
아래와 같이 시퀀스 내에서 하나의 `Runnable` 의 출력을 다음 `Runnable` 의 입력 형식에 맞게 조작하는데 유용하게 사용할 수 있다.

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

prompt = ChatPromptTemplate.from_template(
    """
    사용자 질의에 대해 반드시 아래 문맥만 고려해 답변하세요.

    context: {context}
    question: {question}
    """
)

retrieval_chain = (
        # RunnableParallel 사용되는 부눈
        {'context': retriever, 'question' : RunnablePassthrough()}
        | prompt
        | model
        | StrOutputParser()
)

retrieval_chain.invoke('사과 색상에 해당하는 rgb는 ?')
# 사과는 초록색입니다. 
# 초록색의 RGB 값은 (0, 128, 0)입니다.
```  

위 코드 중 `retrieval_chain` 에서 `RunnableParallel` 이 사용 됐다.
아래 3개지는 모두 동일한 처리를 하는 코드이다.

- `{'context': retriever, 'question' : RunnablePassthrough()}`(`RunnableParallel` 로 래핑됨)
- `RunnableParallel({'context': retriever, 'question' : RunnablePassthrough()})`
- `RunnableParallel(context=retriever, question=RunnablePassthrough())`

만약 사용자의 입력이 2개 이상인 경우 [itemgetter](https://docs.python.org/3/library/operator.html#operator.itemgetter)
를 사용해 딕셔터리에서 원하는 데이터를 추출할 수 있다.

```python
from operator import itemgetter

prompt = ChatPromptTemplate.from_template(
    """
    사용자 질의에 대해 반드시 아래 문맥만 고려해 답변하세요. 답변은 언어에 맞춰 번역하세요.

    context: {context}
    question: {question}
    lang: {lang}
    """
)

retrieval_chain = (
    {
     'context': itemgetter('question') | retriever, 
     'question' : itemgetter('question'),
     'lang' : itemgetter('lang')
    }
    | prompt
    | model
    | StrOutputParser()
)

retrieval_chain.invoke({'question' : '사과 색상에 해당하는 rgb는 ?', 'lang' :'en'})
# The color corresponding to 사과 (apple) is 초록 (green). The RGB value for green is (0, 128, 0).
```  

아래와 같이 앞서 알아본 `RunnableMap` 과 같이 여러 체인을 병렬로 실행하는 용도로도 사용할 수 있다.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel

captial_chain = (
    ChatPromptTemplate.from_template("{country} 의 수도는 어디입니까?")
    | model
    | StrOutputParser()
)


area_chain = (
    ChatPromptTemplate.from_template("{country} 의 면적은 얼마입니까?")
    | model
    | StrOutputParser()
)

popul_chain = (
    ChatPromptTemplate.from_template("{country} 의 인구는 얼마입니까?")
    | model
    | StrOutputParser()
)

parallel_chain = RunnableParallel(capital=captial_chain, area=area_chain, popul=popul_chain)

parallel_chain.invoke({'country' : '한국'})
# {'capital': '한국의 수도는 서울입니다.',
# 'area': '한국의 면적은 약 100,363km²입니다.',
# 'popul': '한국의 인구는 약 5,200만 명입니다. 2023년 6월 기준으로, 한국 통계청이 발표한 인구 통계에 따르면, 한국의 총 인구는 51,811,167명입니다.'}
```  

위와 같이 체인이 모두 같은 템플릿 변수를 사용하는 경우도 가능하고,
아래와 같이 서로드란 템플릿 변수를 사용해도 의도된 동작으로 실행 가능하다.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel

captial_chain = (
    ChatPromptTemplate.from_template("{country1} 의 수도는 어디입니까?")
    | model
    | StrOutputParser()
)


area_chain = (
    ChatPromptTemplate.from_template("{country2} 의 면적은 얼마입니까?")
    | model
    | StrOutputParser()
)

popul_chain = (
    ChatPromptTemplate.from_template("{country3} 의 인구는 얼마입니까?")
    | model
    | StrOutputParser()
)

parallel_chain = RunnableParallel(capital=captial_chain, area=area_chain, popul=popul_chain)

parallel_chain.invoke({'country1' : '한국', 'country2' : '미국', 'country3' : '영국'})
# {'capital': '한국 의 수도는 서울입니다.',
#  'area': '미국의 면적은 약 9,833,517 제곱킬로미터(3,805,927 제곱마일)입니다. 미국은 캐나다와 멕시코와 국경을 공유하며, 북아메리카 대륙의 대부분을 차지합니다.\n\n미국의 면적은 다음과 같이 세분화될 수 있습니다.\n\n* 육지 면적: 9,161,928 제곱킬로미터(3,537,438 제곱마일)\n* 수역 면적: 671,589 제곱킬로미터(259,489 제곱마일)\n\n미국은 세계에서 3번째로 큰 국가로, 러시아와 캐나다에 이어집니다.',
#  'popul': '영국의 인구는 약 6,700만 명입니다(2020년 기준).'}
```  


### RunnableEach
`RunnableEach` 는 리스트(배열) 형태의 입력값의 각 요소마다 동일한 `Runnable` 을 적용하는 컴포넌트이다.
입력값이 여러 개일 때 각 요소를 한 번씩 같은 방식으로 처리하고 처리된 결과를 다시 리스트로 모아 반환한다.

`RunnableEach` 는 아래와 같은 경우 사용할 수 있다.

- 여러 입력값을 같은 `체인/프롬프트/LLM` 등에 일괄 적용하고 싶을 때
- 데이터셋을 `LLM` 이나 체인으로 한 번에 처리할 때
- 여러 개별 요청을 한 번에 일괄 변환/분석/생성할 때
- 반복문 없이 체인 스타일로 일괄 처리를 구현하고 싶을 때

아래는 국가이름을 질의로 받으면 해당 국가의 수도를 답하는 체인을 사용해 여러 국가를 한번에 전달해 답변을 받는 `RunnableEach` 의 예시이다.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables.base import RunnableEach

capital_chain = (
    ChatPromptTemplate.from_template("{country} 의 수도는 어디입니까?")
    | model
    | StrOutputParser()
)

each_chain = RunnableEach(bound=capital_chain)

countries = ['한국', '미국', '영국', '스페인', '이집트']

each_chain.invoke(countries)
# ['한국의 수도는 **서울**입니다.',
# '미국의 수도는 워싱턴 D.C.입니다.',
# '영국 의 수도는 런던입니다.',
# '스페인의 수도는 마드리드입니다. 마드리드는 스페인 중부에 위치하고 있으며, 국가의 정치, 경제, 문화의 중심지 역할을 합니다. 마드리드는 인구 약 320만 명의 대도시로, 유럽에서 가장 큰 도시 중 하나입니다.',
# '이집트 의 수도는 카이로입니다.']
```

### RunnableRetry
`RunnableRetry` 는 실행 중(`LLM` 호출 등) 오류나 예외가 발생할 경우,
지정한 횟수만큼 자동으로 재시도하는 기능을 제공한다.
체인 내에서 특정 `Runnable` 이 실패할 경우 미리 정한 횟수만큼 같은 입력으로 반복 실행한다.
성공시 그 결과를 반환하고 모든 시도에서 실패하면 마지막 예외를 그대로 전달해 안정성과 내결함성을 높여주는 역할을 한다.

`RunnableRetry` 는 아래와 같은 경우 사용할 수 있다.

- `LLM`, `API` 호출 등 외부 서비스가 일시적으로 실패할 수 있을 때
- 네트워크 오류, 일시적 `RateLimit` 초과, 간헐적 예외가 예상될 때
- 워크플로우의 신뢰성과 성공률을 높이고 싶을 때
- 실패 상황에서 자동으로 재시도 로직을 일관되게 적용하고 싶을 때

간단한 사용 예시는 아래와 같다.
재시도 수행 최대 횟수와 재시도 수행 주기도 설정할 수 있다.

```python
import random, logging, datetime
from langchain_core.runnables import RunnableLambda
from langchain_core.runnables.retry import RunnableRetry

def foo(input):
  value = random.random()
  logging.info(f"value : {value}")
  print(f"value : {value}, time : {datetime.datetime.now()}")
  if value > 0.8:
    return input
  else:
    raise ValueError("failed")

runnable = RunnableLambda(foo)

runnable_retry = RunnableRetry(
    bound=runnable,
    retry_exception_types=(ValueError,),
    max_attempt_number=10,
    wait_exponential_jitter=True,
    exponential_jitter_params={'initial': 2}
)

runnable_retry.invoke('재시도 수행!')
# value : 0.69321661519523, time : 12:12:36.169931
# value : 0.7290873830446856, time : 12:12:38.719092
# value : 0.2821680875499617, time : 12:12:42.800953
# value : 0.8679592745698509, time : 12:12:51.038793
# 재시도 수행!
```  

`RunnableRetry` 를 직접 사용하지 않고 `Runnable` 구현체라면 `with_retry()` 를 사용해 동일한 재시도 동작을 구현할 수 있다.

```python
runnable_with_retry = runnable.with_retry(
    retry_if_exception_type=(ValueError,),
    wait_exponential_jitter=True,
    exponential_jitter_params={'initial': 2},
    stop_after_attempt=10
)

runnable_with_retry.invoke('재시도 수행!')
# value : 0.25226005335713353, time : 12:16:16
# value : 0.4511207248548069, time : 12:16:18
# value : 0.07975708951682481, time : 12:16:23
# value : 0.26525273785183157, time : 12:16:32
# value : 0.6358726157130956, time : 12:16:48
# value : 0.8607426064017667, time : 12:17:21
# 재시도 수행!
```  

재시도 동작의 경우 재시도가 수행되는 범위를 가능한 작게 유지하는 것이 좋다.
즉 여러 체인이 연결된 최종 체인에 재시도를 적용하는 것보다 개별 체인에 재시도를 적용하는 것이 좋다.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableParallel

capital_chain = (
    ChatPromptTemplate.from_template("{country1} 의 수도는 어디입니까?")
    | model
    | StrOutputParser()
)


area_chain = (
    ChatPromptTemplate.from_template("{country2} 의 면은 얼마입니까?")
    | model
    | StrOutputParser()
)

popul_chain = (
    ChatPromptTemplate.from_template("{country3} 의 인구는 얼마입니까?")
    | model
    | StrOutputParser()
)

# bad
parallel_chain = RunnableParallel(capital=capital_chain, area=area_chain, popul=popul_chain)
parallel_chain_retry = parallel_chain.with_retry()

# good
parallel_chain_retry = RunnableParallel(capital=capital_chain.with_retry(), 
                                        area=area_chain.with_retry(), 
                                        popul=popul_chain.with_retry())
```  
