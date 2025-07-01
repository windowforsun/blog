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
toc: true
use_math: true
---  


## Text Splitter
`Text Splitter` 는 큰 텍스트 문서를 언어 모델이 처리할 수 있는 작은 단위(`chunk`)로 나누는 도구이다. 
대부분 `LLM` 은 입력 토큰 수 제한이 있어 큰 문서를 그대로 처리할 수 없기 때문에, 
세분화함으로써 질문에 연관성이 있는 정보만 가져오는 데 도움이 된다. 
각각의 단위는 특정 주제나 내용에 초점을 맞추므로, 관련성이 높은 정보를 제공할 수 있게 된다. 
그리고 전체 문서를 `LLM` 으로 입력하면 그 만큼 비용이 발생하고, 효율적인 답변을 얻기 어려울 수 있다. 
이러한 문제는 할루시네이션으로 이어질 수 있기 때문에 정보량을 줄이면서 질문에 필요한 정보만 발췌해 이를 개선할 수 있다.  

`Test Splitter` 는 다양한 형식으로 작성된 문서에서 구조를 파악한다. 
이는 문서의 헤더, 본문, 푸터, 페이지 번호, 섹션 제목 등이 될 수 있다. 
그리고 문서를 어떤 단위로 나눌지 결정한다. 
이는 문서의 내용과 목적에 따라 페이지, 섹션, 문단 등 다양하게 결정될 수 있다. 
하나의 문서를 몇 개의 토큰 단위로 나눌지 결정해야 하고, 
각 분할마다 맥락이 이어질 수 있도록 얼만큼을 겹치도록 할지도 결정해야 한다.  

공통적으로 `Text Splitter` 는 아래와 같은 주요 매개변수가 있다.  

- `chunk_size` : 각 청크의 최대 크기(문자 또는 토큰 수)
- `chunk_overlap` : 연속돤 청크 간 중복되는 부분의 크기
- `sparators` : 텍스트 분할에 사용할 구분자 목록
- `length_function` : 청크 크기를 측정하는 함수(문자 수 또는 토큰 수)

`Text Splitter` 의 종류는 [여기](https://python.langchain.com/api_reference/text_splitters/index.html)
에서 확인할 수 있다. 

필요에 따라 아래 라이브러리 설치가 필요할 수 있다. 

```bash
$ pip install -qU langchain-text-splitters
```



### CharacterTextSplitter
`CharacterTextSplitter` 는 가장 기본적인 텍스트 분할 도구로, 지정된 문자 또는 문자열을 기준으로 텍스트를 청크로 분할한다. 

작동 방식은 아래와 같다. 

- 구분자 기빈 분할 : 텍스트를 지정된 구분자를 기준으로 분할한다. 
- 청크 크기 제한 : 분할된 각 청크가 지정된 최대 크기를 초과하지 않도록 조절한다. 
- 청크 결합 : 너무 작은 청크는 병합하여 효율적인 크기로 만든다. 
- 청크 중복 : 필요에 따라 인접한 청크 간에 중복을 적용한다.  

`CharacterTextSplitter` 는 주로 간단한 텍스트 분할이 필요하거나, 
텍스트가 명확한 구분자로 나뉘어 있을 때 유용하다. 
그리고 빠른 처리 속도가 필요할 떄도 적합하다. 
만약 복잡한 문서 구조를 다뤄야 한다면 다른 `Text Splitter` 를 고려해야 한다.   

예제는 아래 `.txt` 파일을 사용한다.

<details><summary>CSV 내용</summary>
<div markdown="1">

```text
안녕하세요. LangChain의 TextSplitter를 테스트하기 위한 예제 텍스트입니다.

이 문서는 다양한 문장과 문단으로 구성되어 있으며, 각 TextSplitter의 동작을 비교하는 데 사용됩니다. 
예를 들어, 단순히 문장 단위로 나누는 경우와 특정 길이로 나누는 경우의 결과를 확인할 수 있습니다. 
또한, 이 텍스트는 여러 종류의 데이터를 포함하고 있어 다양한 상황에서 Splitter의 성능을 평가할 수 있습니다.

첫 번째 문단은 간단한 문장들로 구성되어 있습니다. 
예를 들어, "안녕하세요."와 같은 짧은 문장부터, "이 문장은 조금 더 길어서 Splitter가 어떻게 처리하는지 확인할 수 있습니다."와 같은 문장까지 포함됩니다. 
이 문단은 Splitter가 문장 단위로 나누는 경우와 길이 기반으로 나누는 경우의 차이를 확인하는 데 유용합니다.

두 번째 문단은 조금 더 긴 문장과 함께, 쉼표(,)와 마침표(.)를 포함합니다. 
또한, 여러 줄로 구성된 문단입니다. 
예를 들어, "이 문장은 쉼표를 포함하고 있으며, 여러 문장으로 구성되어 있습니다. 
이 문장은 Splitter가 쉼표를 어떻게 처리하는지 확인할 수 있도록 설계되었습니다."와 같은 문장이 포함됩니다. 
이 문단은 Splitter가 문장 내부의 쉼표를 처리하는 방식을 평가하는 데 적합합니다.

세 번째 문단은 특수 문자와 공백을 포함합니다: @, #, $, %, &, *, ( ), -, _, +, =, 그리고 공백. 
특수 문자는 Splitter가 텍스트를 나눌 때 어떤 영향을 미치는지 확인하는 데 유용합니다. 
예를 들어, "이 문장은 특수 문자를 포함하고 있습니다: @, #, $, %, &, *."와 같은 문장이 포함됩니다. 
또한, 공백이 많은 문장도 포함되어 있어 Splitter가 공백을 처리하는 방식을 평가할 수 있습니다.

다음은 테스트를 위한 긴 문장입니다. 
이 문장은 길이가 길어서 여러 TextSplitter가 어떻게 처리하는지 확인하기에 적합합니다. 
예를 들어, 특정 길이로 나누는 Splitter는 이 문장을 여러 조각으로 나눌 것입니다. 
반면, 문장 단위 Splitter는 이 문장을 하나의 단위로 처리할 가능성이 높습니다. 
이 문장은 Splitter가 긴 문장을 처리하는 성능을 평가하는 데 유용합니다.

네 번째 문단은 다국어 텍스트를 포함합니다. 
예를 들어, "This is an English sentence."와 같은 영어 문장과, "これは日本語の文章です。"와 같은 일본어 문장이 포함됩니다. 
또한, 숫자 1234567890과 같은 데이터도 포함되어 있습니다. 
이 문단은 Splitter가 다국어 텍스트와 숫자를 처리하는 방식을 평가하는 데 적합합니다.

다섯 번째 문단은 목록과 번호를 포함합니다:
1. 첫 번째 항목입니다.
2. 두 번째 항목입니다.
3. 세 번째 항목은 조금 더 길어서 Splitter가 어떻게 처리하는지 확인할 수 있습니다. 
   예를 들어, "이 항목은 여러 줄로 구성되어 있습니다."와 같은 문장이 포함됩니다.
4. 네 번째 항목은 특수 문자와 공백을 포함합니다: @, #, $, %, &, *, ( ), -, _, +, =.

마지막으로, 이 텍스트는 한글뿐만 아니라 영어와 숫자도 포함합니다. 
For example, this sentence is written in English. 
숫자 1234567890도 포함되어 있습니다. 
또한, 특수 문자와 공백이 포함된 문장도 포함되어 있습니다. 
예를 들어, "이 문장은 특수 문자와 공백을 포함하고 있습니다: @, #, $, %, &, *."와 같은 문장이 포함됩니다.

감사합니다. TextSplitter 테스트를 성공적으로 진행하시길 바랍니다!
```

</div>
</details>  


```python
from langchain_text_splitters import CharacterTextSplitter

with open('./text-split-exam.txt') as f:
  file = f.read()


text_splitter = CharacterTextSplitter(
    separator=' ',
    chunk_size=250,
    chunk_overlap=50,
    length_function=len,
    is_separator_regex=False
)

texts = text_splitter.create_documents([file])

print(texts[0])
# page_content='안녕하세요. LangChain의 TextSplitter를 테스트하기 위한 예제 텍스트입니다.
# 
# 이 문서는 다양한 문장과 문단으로 구성되어 있으며, 각 TextSplitter의 동작을 비교하는 데 사용됩니다.
# 예를 들어, 단순히 문장 단위로 나누는 경우와 특정 길이로 나누는 경우의 결과를 확인할 수 있습니다.
# 또한, 이 텍스트는 여러 종류의 데이터를 포함하고 있어 다양한 상황에서 Splitter의 성능을 평가할 수 있습니다.
# 
# 첫 번째 문단은'
print(texts[1])
# page_content='있어 다양한 상황에서 Splitter의 성능을 평가할 수 있습니다.
# 
# 첫 번째 문단은 간단한 문장들로 구성되어 있습니다.
# 예를 들어, "안녕하세요."와 같은 짧은 문장부터, "이 문장은 조금 더 길어서 Splitter가 어떻게 처리하는지 확인할 수 있습니다."와 같은 문장까지 포함됩니다.
# 이 문단은 Splitter가 문장 단위로 나누는 경우와 길이 기반으로 나누는 경우의 차이를 확인하는 데 유용합니다.
# 
# 두 번째 문단은 조금 더 긴 문장과'


# 필요에 따라 아래와 같이 파일별 메타데이터를 추가할 수 있다. 
metadatas = [
    {'file_name' : 'text-split-exam.txt'}
]

documents = text_splitter.create_documents([file], metadatas=metadatas)

documents[0]
# Document(metadata={'file_name': 'text-split-exam.txt'}, page_content='안녕하세요. LangChain의 TextSplitter를 테스트하기 위한 예제 텍스트입니다.\n\n이 문서는 다양한 문장과 문단으로 구성되어 있으며, 각 TextSplitter의 동작을 비교하는 데 사용됩니다. \n예를 들어, 단순히 문장 단위로 나누는 경우와 특정 길이로 나누는 경우의 결과를 확인할 수 있습니다. \n또한, 이 텍스트는 여러 종류의 데이터를 포함하고 있어 다양한 상황에서 Splitter의 성능을 평가할 수 있습니다.')


# 문자열 분할만 필요한 경우
texts = text_splitter.split_text(file)

texts[0]
# 안녕하세요. LangChain의 TextSplitter를 테스트하기 위한 예제 텍스트입니다.
# 
# 이 문서는 다양한 문장과 문단으로 구성되어 있으며, 각 TextSplitter의 동작을 비교하는 데 사용됩니다.
# 예를 들어, 단순히 문장 단위로 나누는 경우와 특정 길이로 나누는 경우의 결과를 확인할 수 있습니다.
# 또한, 이 텍스트는 여러 종류의 데이터를 포함하고 있어 다양한 상황에서 Splitter의 성능을 평가할 수 있습니다.
# 
# 첫 번째 문단은
```  

### RecursiveCharacterTextSplitter
`RecursiveCharacterTextSplitter` 는 여러 구분자를 계측정으로 적용하여 텍스트를 의미있는 청크로 분할한다. 
이는 청크가 충분히 작아질 때까지 주어진 문자 목록 순서대로 텍스트를 분할하려고 시도한다. 
기분 문자 목록은 `["\n\n", "\n", " ", ""]` 로 단락, 문장, 단어 순으로 재귀적으로 분할한다.  

`CharacterTextSplitter` 와 차이점은 다음과 같다. 

- 문맥 보존 향상 : 의미 있는 구조를 더 잘 보존한다. 
- 다중 구분자 : 단일 구분자가 아닌 여러 구분자를 우선순위에 따라 사용한다. 
- 적응형 분할 : 문서 구조에 따라 자동으로 적절한 분할 방식을 선택한다. 
- 지능적 분할 : 단순히 크기만 고려하는 것이 아니라 문서의 논리적 구조를 고려한다. 

그러므로 `RecursiveCharacterTextSplitter` 는 문서의 논리적 구조 보존이 중요하거나, 
다양한 형식의 문서를 처리해야 할 때 유용하다. 
그리고 효과적인 정보 검색이 필요한 `RAG` 시스템 구축 혹은 문맥 유지가 중요한 질의응답 시스템에서 잘 활용할 수 있다.  


`RecursiveCharacterTextSplitter` 예제 또한 `CharacterTextSplitter` 와 동일한 텍스트 파일을 사용한다. 

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 문맥 유지가 여려울 정도로 작은 정크이면서 chunk_overlap 을 허용하는 경우 다음과 같이 중복이 발생할 수 있다. 
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=50,
    chunk_overlap=30,
    length_function=len,
    is_separator_regex=False
)

texts = text_splitter.create_documents([file])

print(texts[0])
# page_content='안녕하세요. LangChain의 TextSplitter를 테스트하기 위한 예제'
print(texts[1])
# page_content='TextSplitter를 테스트하기 위한 예제 텍스트입니다.'
print(texts[2])
# page_content='이 문서는 다양한 문장과 문단으로 구성되어 있으며, 각 TextSplitter의 동작을'
print(texts[3])
# page_content='구성되어 있으며, 각 TextSplitter의 동작을 비교하는 데 사용됩니다.'


# 충분한 청크 크기로 나눈다면 chunk_overlap 을 허용해도 겹치는 경우가 발생하지 않을 수 있다. 
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=250,
    chunk_overlap=50,
    length_function=len,
    is_separator_regex=False
)

texts = text_splitter.create_documents([file])

print(texts[0])
# page_content='안녕하세요. LangChain의 TextSplitter를 테스트하기 위한 예제 텍스트입니다.
# 
# 이 문서는 다양한 문장과 문단으로 구성되어 있으며, 각 TextSplitter의 동작을 비교하는 데 사용됩니다.
# 예를 들어, 단순히 문장 단위로 나누는 경우와 특정 길이로 나누는 경우의 결과를 확인할 수 있습니다.
# 또한, 이 텍스트는 여러 종류의 데이터를 포함하고 있어 다양한 상황에서 Splitter의 성능을 평가할 수 있습니다.'
print(texts[1])
# page_content='첫 번째 문단은 간단한 문장들로 구성되어 있습니다.
# 예를 들어, "안녕하세요."와 같은 짧은 문장부터, "이 문장은 조금 더 길어서 Splitter가 어떻게 처리하는지 확인할 수 있습니다."와 같은 문장까지 포함됩니다.
# 이 문단은 Splitter가 문장 단위로 나누는 경우와 길이 기반으로 나누는 경우의 차이를 확인하는 데 유용합니다.'
print(texts[2])
# page_content='두 번째 문단은 조금 더 긴 문장과 함께, 쉼표(,)와 마침표(.)를 포함합니다.
# 또한, 여러 줄로 구성된 문단입니다.
# 예를 들어, "이 문장은 쉼표를 포함하고 있으며, 여러 문장으로 구성되어 있습니다.
# 이 문장은 Splitter가 쉼표를 어떻게 처리하는지 확인할 수 있도록 설계되었습니다."와 같은 문장이 포함됩니다.
# 이 문단은 Splitter가 문장 내부의 쉼표를 처리하는 방식을 평가하는 데 적합합니다.'
print(texts[3])
# page_content='세 번째 문단은 특수 문자와 공백을 포함합니다: @, #, $, %, &, *, ( ), -, _, +, =, 그리고 공백. 
# 특수 문자는 Splitter가 텍스트를 나눌 때 어떤 영향을 미치는지 확인하는 데 유용합니다.
# 예를 들어, "이 문장은 특수 문자를 포함하고 있습니다: @, #, $, %, &, *."와 같은 문장이 포함됩니다.'
```  
