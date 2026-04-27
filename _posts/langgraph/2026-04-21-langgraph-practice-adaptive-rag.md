--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph Adaptive RAG"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: '일반 RAG 구조의 한계를 극복하기 위해 외부 지식베이스에서 관련 정보를 탐색까지하는 Adaptive RAG 에 대해 알아보자.'
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
    - RAG
    - VectorStore
    - Adaptive RAG
    - Retrieval
    - Web Search
    - Query Rewrite
toc: true
use_math: true
---  

## Adaptive RAG
`Adaptive RAG` 는 기존 `RAG` 구조의 한계를 극복하기 위해 등장한 접근 방식이다. 
`RAG` 는 `LLM` 이 외부 지식베이스에서 관련 정보를 검색하여 답변 생성에 활용하는 구조인데, 
`Adaptive RAG` 는 `검색` 과 `생성` 과정 모두를 동적으로 조정하여 더욱 효율적이고 정확한 응답을 생성할 수 있도록 한다.  

단순히 검색 걸과를 `LLM` 에 넣는 것이 아니라, 상황에 따라 검색 전략, 문서 수, 검색 방법, `LLM` 파라미터 등을 동적으로 조정한다. 
`적응형` 이라는 말 그대로, 질문의 성격, 난이도, 맥락, 사용자 의도 등에 따라 검색 및 생성 과정을 실시간으로 변화시킨다.  

`Adaptive RAG` 의 주요 특징은 다음과 같다. 

- 동적 검색 증상 전략 : 질문이 간단한 경우 검색 문서 수를 줄이고, 빠른 응딥이 가능하도록 한다. 질문이 복잡한하거나 애매한 경우라면 더 많은 문서, 다양한 검색 기법, 다중 지식 소스를 활용한다. 
- 멀티모달/멀티소스 적용 : 텍스트뿐 아니라 이미지, 표, 코드 등 다양한 데이터를 적응적으로 활용할 수 있다. 복수의 벡터 `DB`, 검색 엔진, `API` 등에서 상황별로 최적의 소스를 선택한다. 
- 생성 컨트롤 : 생성 모델의 온도, 길이, 시스템 프롬프트 등 하이퍼파라미터를 동적으로 변경할 수 있다. 그리고 필요시 생성 결과를 재검색, 재생성할 수 있다. 
- 피드백 기반 적응 : 사용자 피드백, 이전 대화 맥락, 실페 케이스 등을 학습하여 다음 검색/생성 전략에 반영한다. 
- 효율성과 정확성의 균형 : 리소스와 품질을 상황별로 최적화 할 수 있다.  

`Naive RAG` 는 `LLM` 이 외부 지식베이스에서 관련 정보를 검색하여 답변 생성에 활용하는 단순한 구조로
단순한 질의응답, 검색 겨로가를 바로 `LLM` 에 입력해 답변을 생성하는 방법이다.  

```
Query -> Retriever -> Retrieved Docs -> Generator(LLM) -> Answer
```  

`Agentic RAG` 는 `RAG` 를 `Agent` 프레임워크에 통합한 구조로. `RAG` 가 단일 질의-응답을 넘어, 
여러 단계의 행동(검색, 추론, 도구 활용 등)을 스스로 설계/실행하여 목표를 달성하는 방법이다. 
즉 에이전트가 목표 달성을 위해 검색/생성/추론/도구 활용을 여러 단계로 설계 및 실행하는 방법이다.  

```
Query -> Agnet(행동계획) -> 도구(검색, API, 연산) -> Generator(LLM) -> Answer

# Agent 가 여러 번 검색/생성/추론/피드백을 반복
```  

이와 비교해 `Adaptive RAG` 는 기존 `RAG` 의 검색 및 생성 과정을 질문의 복잡성, 상황, 맥락 등에 따라 `동적`으로 조정하는 `RAG` 구조로 상황에 맞게 적응하며 수행된다.  

```
Query -> Adaptive Retriever(동적 검색) -> Adaptive Agumented(동적 증상) -> Adaptive Gnerator(동적 생성) -> Answer
```  

이를 표로 정리하면 다음과 같다.  



| 구분          | 일반 RAG            | Adaptive RAG                   | Agentic RAG                        |
|--------------|---------------------|--------------------------------|-------------------------------------|
| **검색 전략**   | 고정                | 동적(질문, 상황에 따라 변화)        | Agent가 직접 도구를 골라서 실행        |
| **생성 제어**   | 고정                | 동적(하이퍼파라미터, 프롬프트 등)     | Agent가 행동계획에 따라 여러 번 생성    |
| **질의응답 흐름**| 단일 패스           | 상황별 최적화된 단일/다중 패스        | 다중 스텝, 다단계 추론·검색·생성 반복   |
| **피드백 활용**  | 제한적              | 실시간 반영 가능                  | Agent가 결과에 따라 전략 반복/수정      |
| **멀티소스/모달**| 보통(텍스트 위주)    | 가능(멀티모달, 멀티소스 적응)         | Agent가 다양한 도구/소스 활용 가능      |
| **목표설정/계획**| 없음                | 없음                             | Agent가 자체적으로 목표·플랜 설계       |
| **응용 난이도**  | 낮음                | 중간                             | 높음                                 |
| **확장성/유연성**| 보통                | 높음                             | 매우 높음                            |


본 포스팅에서 진행할 예제는 [이전 예제]({{site.baseurl}}{% link _posts/langgraph/2025-03-08-langgraph-practice-agentic-rag.md %})
의 내용을 포함하고 하고 있으므로 이전 내용의 숙지가 필요하다.  


### Query Routing
`Query Routing` 은 사용자의 질문을 분석하여, 어떤 데이터 소스로 정보를 찾으러 갈지 결정하는 과정이다. 
`Adaptive RAG` 에서는 중요한 과정 중 하나로 이유는 다음과 같다. 

- 질문의 주제나 목적에 따라 적합한 소스가 달라질 수 있다. 
- 불필요한 정보 검색을 줄이고, 빠르고 정확한 답변을 제공할 수 있다. 
- 최신 정보는 웹 검색, 특정 리포트/논문은 벡터스토어 등으로 구분하여 검색 품질을 높일 수 있다.  

본 예제에서는 `Query Routging` 을 위해 `LLM` 기반 의가 결정을 사용한다. 
`LLM` 이 질문의 맥락과 키워드를 파악해, 내부 벡터스토어에서 검색을 수행할지, 외부 웹 검색을 할지 자동 판단하도록 한다.  

```python
from typing import Literal

from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate
from pydantic import BaseModel, Field


class RouteQuery(BaseModel):
  """
  Route a use query to the most relevant datasource.
  """

  datasource: Literal["vectorstore", "web_search"] = Field(
      ...,
      description="Given a user question choose th route it to web search or vectorstore"
  )

llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")
structured_llm_router = llm.with_structured_output(RouteQuery)

route_prompt = ChatPromptTemplate.from_messages(
    [
        SystemMessagePromptTemplate.from_template(template="""
        You are an expert at routing a user question to a vectorstore or web search.
        The vectorstore contains documents related to Koea Weather Inforation for 2025 01-06.
        Use the vectorstore for question on these topics. Otherwise, use web-search.
        """),
        HumanMessagePromptTemplate.from_template(template="""
        {question}
        """,)
    ]
)

question_router = route_prompt | structured_llm_router

question_router.invoke({'question':'6월 날씨 요약해줘'})
# RouteQuery(datasource='vectorstore')

question_router.invoke({'question':'미국의 수도는 어디야?'})
# RouteQuery(datasource='web_search')

question_router.invoke({'question':'서울의 현재 날씨 알려줘'})
# RouteQuery(datasource='web_search')
```  

### Retrieval Grader
`Retrieval Grader` 는 검색된 문서들이 실제 답변에 적합한지, 품질과 관련성(`Accuracy & Relevance`)이 높은지 판단하는 과정이다. 
`Adaptive RAG` 에서는 이 단계를 통해 아래와 같은 것들을 가능하게 한다.  

- 검색된 문서가 질문과 정말 관련 있는지 자동 평가
- 관련성/신뢰도 점수가 낮은 문서는 제외하고, 답변 품질 보장
- 여러 문서 중 가장 좋은 것만 `LLM` 에 전달해 최종 생성 품질 극대화

본 예제에서는 사용자의 질의와 검색된 문서를 `LLM` 기반 의사 결정을 사용해 판단한다. 
`LLM` 이 질의와 검색된 개별 문서를 보고 적합한지 판단해 `yes` 또는 `no` 로 응답하도록 한다.  

```python
from pydantic import BaseModel, Field
from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate

class GradeDocuments(BaseModel):
  """
  Binary score or relevance check on retrieved documents.
  """

  binary_score: str = Field(
      description="Documents are relevant to the question. 'yes' or 'no'"
  )

llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")
structured_llm_grader = llm.with_structured_output(GradeDocuments)

grade_prompt = ChatPromptTemplate.from_messages(
    [
        SystemMessagePromptTemplate.from_template(template="""
        You are a grader assessing relevant of a retrieved document to a user question.
        If the document contains keyword(s) or semantic meaning related to the user question,
        grade it a relevant.

        It does not need to be a stringent test.
        The goal is to filter out erronous retrievals.
        Given a binary score 'yes' or 'no' score to indicate whether the docuemtn is relevant to the question.
        """),
        HumanMessagePromptTemplate.from_template(template="""
        Retrieved document:
        {document}

        User question:
        {question}
        """)
    ]
)


retrieval_grader = grade_prompt | structured_llm_grader




def merge_docs(docs):
    return "\n\n".join(
        [
            f'<document><content>{doc.page_content}</content><source>{doc.metadata["source"]}</source><page>{doc.metadata["page"]+1}</page></document>'
            for doc in docs
        ]
    )

docs = pdf_retriever.invoke("6월 기온 정보")
retrieval_grader.invoke({'question' : '6월 기온 정보', 'document' : merge_docs })
# GradeDocuments(binary_score='yes')


retrieval_grader.invoke({'question' : 'youtude 카테고리별 구독자 통계', 'document' : merge_docs })
# GradeDocuments(binary_score='no')
```  

### RAG Chain
질의에 대한 `VectorStore` 검색을 위해 `이전 예제` 에서 구현한 `DemoRetrievalChain` 을 초기화해 준비한다.

```python
from langchain_huggingface.embeddings import HuggingFaceEndpointEmbeddings
from langchain_huggingface.embeddings import HuggingFaceEmbeddings
from langchain_google_genai import ChatGoogleGenerativeAI
import os

os.environ["GOOGLE_API_KEY"] = "api_key"
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

os.environ["HUGGINGFACEHUB_API_TOKEN"] = "api_key"
model_name = "BM-K/KoSimCSE-roberta"
# model_name = "BM-K/KoMiniLM"
hf_endpoint_embeddings = HuggingFaceEndpointEmbeddings(
    model=model_name,
    huggingfacehub_api_token=os.environ["HUGGINGFACEHUB_API_TOKEN"],
)

hf_embeddings = HuggingFaceEmbeddings(
    model_name=model_name,
    encode_kwargs={'normalize_embeddings':True},
)

# retriever 생성
file_list = [
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf',
    '/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf'
]

collection_name='weather-docs-6'
persist_directory = f"/content/drive/MyDrive/Colab Notebooks/data/vectorstore/{collection_name}"

pdf = DemoRetrievalChain(
    source_uri=file_list,
    embeddings=hf_embeddings,
    llm=llm,
    collection_name=collection_name,
    persist_directory=persist_directory
).create_chain()

pdf_retriever = pdf.retriever
rag_chain = pdf.chain
```  

앞서 생성한 `pdf_chain` 을 테스트하면 아래와 같은 결과를 확인 할 수 있다.  

```python
llm_result = rag_chain.invoke({'context': merge_docs(docs), 'question' : '6월 기온 정보 요약', 'chat_history' : []})

print(llm_result)
# 2025년 6월 전국 평균기온은 22.9℃로 평년(21.4℃)보다 +1.5℃ 높았습니다.
# 
# **출처**
# - /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf
```  

### Checker(Hallucination/Relevance)
`Halluncination` 는 `LLM` 이 실제 주어진 정보나 문서에 근거하지 않는 내용을 지어내는 현상을 의미한다. 
그러므로 `RAG` 시스템에서 답변의 신뢰성과 정확성을 높이기 위해, 생성된 답변이 검색된 문서에 근거했는지 평가하는 것이 중요하다. 
`LLM` 은 논리적이고 그럴듯한 답변을 생성할 수 있으나, 실제로 근거 없는 정보(환각)를 말할 때가 있기 때문이다.  

구현할 `Hallucination Checker` 는 `LLM` 이 생성한 답변이 검색된 문서에 근거했는지 평가하는 평가자 역할을 한다. 
그리고 평가의 결과를 `binary_score` 답변으로 `yes/no` 로 응답한다. 
방식은 `LLM` 에게 검색된 문서와 `LLM` 의 답변을 함께 전달해 평가도록 하는 방식이다.  

```python
from pydantic import BaseModel, Field
from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate

class GradeHallucinations(BaseModel):
  """
  Binary score for hallucination present in generation anwser.
  """

  binary_score: str = Field(
      description="Answer is grounded in the facts, 'yes' or 'no'"
  )

llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")
structured_llm_hallu_checker = llm.with_structured_output(GradeHallucinations)

hallu_prompt = ChatPromptTemplate.from_messages(
    [
        SystemMessagePromptTemplate.from_template(template="""
        You are a grader assessing whether an LLM generation is grounded in / supported by as set of retrieved facts.
        Given a binary score 'yes' or 'no'.
        'Yes' means that the answer is grounded in / supported by the set of facts.
        """),
        HumanMessagePromptTemplate.from_template(template="""
        Set of facts:
        {documents}

        LLM generations:
        {generation}
        """)
    ]
)

hallu_grader = hallu_prompt | structured_llm_hallu_checker

hallu_grader.invoke({'documents': merge_docs(docs), 'generation' : llm_result})
# GradeHallucinations(binary_score='yes')

hallu_grader.invoke({'documents': merge_docs(docs), 'generation' : '날씨와 관련된 주식 종목은 아래와 같습니다. - 날씨연구소 - 기후정책기관'})
# GradeHallucinations(binary_score='no')
```  

다음으로 `Relevance` 는 `LLM` 의 답변이 실제로 질문을 해결했는 지에 대한 평가이다. 
이러한 평가를 통해 불충분/부적절한 판변시 추가 검색 및 재생성을 통해 질문에 대한 답변을 개선할 수 있다.  

구현할 `Relevance Checker` 또한 `LLM` 이 생성한 답변이 사용자의 질문과 연관/해결했는지 평가하는 평가자 역할을 한다. 
그리고 평가의 결과물은 `binary_score` 답변으로 `yes/no` 로 응답한다. 
방식은 `LLM` 에게 사용자의 질문과 `LLM` 의 답변을 함께 전달해 평가하도록 하는 방식이다.  

```python

class GradeAnswer(BaseModel):
  """
  Binary scoring to evaluate the appropriateness of answer to question
  """

  binary_score: str = Field(
      description="Indicate 'yes' or 'no' whether the answer solves the question"
  )

llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")
structured_llm_answer_grader = llm.with_structured_output(GradeAnswer)

answer_grader_prompt = ChatPromptTemplate.from_messages(
    [
        SystemMessagePromptTemplate.from_template(template="""
        You are grader asesessing whether an answer address / resolves a question
        Given a binary score 'yes' or 'no'.
        'yes' means that answer resolves the question.
        """),
        HumanMessagePromptTemplate.from_template(template="""
        User question:
        {question}

        LLM generation:
        {generation}
        """)
    ]
)

answer_grader = answer_grader_prompt | structured_llm_answer_grader

answer_grader.invoke({'question' : '6월 기온 요약해줘', 'generation': llm_result})
# GradeAnswer(binary_score='yes')

answer_grader.invoke({'question' : '날씨 기후와 관련된 주식 종목 추천해줘', 'generation': llm_result})
# GradeAnswer(binary_score='no')
```  

### Query Rewriter
`Query Rewriter` 는 사용자가 입력한 원래 질문을 정보 검색에 더 적합하도록 의도와 의미를 명확히 하여 개선된 형태의 질문으로 반환하는 과정이다. 
사용자는 질문을 모호하게, 또는 불필요한 정보를 포함해서 입력할 수 있는데, 
벡터스토어, 검색엔진 등은 질문이 명확하고 키워드가 잘 포함되어 있을 수록 관련성이 높은 결과를 반환한다. 
그러므로 이러한 과정을 통해 질문을 더 명확하고 검색 친화적으로 바꾸어 `RAG` 의 검색 성능을 향상 시킬 수 있다.  

구현할 `Query Rewriter` 는 사용자의 질문을 받아, 검색에 더 적합한 형태로 재작성하는 역할을 한다. 
방식은 시스템 프롬프트에 역할과 개선 방향성을 제시하고 입력값으로 기존 질문을 전달해 개선된 질문을 생성하도록 한다.  

```python
from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

re_write_prompt = ChatPromptTemplate.from_messages(
    [
        SystemMessagePromptTemplate.from_template(template="""
        You a question re-writer that converts an input question to a better version that is optimized
        for vectorstore retrieval. Look at the input and try to reason about the underlying semantic intent/meaning

        Final Output muse be contain only the re-written question.
        """),
        HumanMessagePromptTemplate.from_template(template="""
        Here is teh initial question:
        {question}

        Formulate an improved question.
        """)
    ]
)

question_rewriter = re_write_prompt | llm | StrOutputParser()


question_rewriter.invoke({'question' : '6월 기온 요약해줘'})
# 6월 기온에 대한 요약 정보를 알려줘

question_rewriter.invoke({'question' : '날씨 기후와 관련된 주식 종목 정리해줘'})
# 날씨 및 기후 변화 관련 주식 종목 정보
```  

### Web Search Tool
`웹 검색 도구` 는 `RAG` 시스템에서 최신/실시간 정보, 외부 동적 데이터 등 벡터스토어에 없는 정보를 검색하는 역할을 한다. 
이를 통해 사용자가 벡터스토어에 없는 질문이나 최신 정보를 요구하는 경우나 필요한 경우 해당 도구를 활용해 질문에 적합한 답변을 제공할 수 있다.  

```python
# 웹 검색 도구

from langchain_community.utilities import GoogleSerperAPIWrapper
import json

os.environ["SERPER_API_KEY"] = "api key"
web_search_tool = GoogleSerperAPIWrapper()

search_query = '날씨 기후와 관련된 주식 종목 정리'

search_result = web_search_tool.results(search_query)['organic']

search_result
# [{'title': '여름(폭염) 테마주 관련주 32종목 정리 - 주달',
#   'link': 'https://www.judal.co.kr/?view=stockList&themeIdx=135',
#   'snippet': '여름(폭염) 테마주. 전일비: -1.03%. 3일합산: -1.06%. 52주 상승률: 33.48%. 52주 하락률: -18.91%. 기대 수익률: 23.32%. 소외지수: 53. 3년 테마지수: 58.',
#   'position': 1},
#  {'title': '폭염 관련주 & 테마주 23종목 총정리 [2025년 최신]   - 알파스퀘어',
#   'link': 'https://alphasquare.co.kr/home/theme-factor?theme-id=126',
#   'snippet': '폭염 관련주 - 풍국주정 등 폭염 테마주 23종목 완벽 정리! 실시간 대장주, 테마 사유 등 폭염 테마 투자 전 알아야 할 핵심 정보를 한 번에 확인해보세요.',
#   'position': 2},
#  {'title': '여름 관련주 대장주식 폭염 아이스크림 주류 수혜주 - 네이버 블로그',
#   'link': 'https://m.blog.naver.com/funyggb/223475177960',
#   'snippet': '상승세를 보이는 여름 관련주 대장주식 폭염 아이스크림 주류 수혜주에는 어떤 종목들이지 확인하고, 아래 그림으로 PBR, BPS, EPS, PER 등도 살펴보기 ...',
#   'date': 'Jun 10, 2024',
#   'position': 3},
# 
# ...
#  
#  {'title': 'ESG 경영 관련주 기후변화 관련주 투자하기 괜찮은 종목 정리',
#   'link': 'https://idmeans.tistory.com/34',
#   'snippet': 'ESG 경영 관련주 기후변화 관련주 투자하기 괜찮은 종목 정리 · 1)유한양행 · 2)삼성전기 · 3)LG생활건강 · 4)만도 · 5)삼성에스디에스 · 6)현대글로비스 · 7)LG ...',
#   'date': 'May 2, 2021',
#   'position': 10}]
```  


### Adaptive RAG Graph

먼저 `Graph` 에서 사용할 상태를 정의한다.  

```python
# 그래프 구성 - 상태 정의

from typing import List
from typing_extensions import TypedDict, Annotated

class GraphState(TypedDict):
  question: Annotated[str, 'User question']
  generation: Annotated[str, 'LLM generated answer']
  documents: Annotated[List[str], 'List of documents']
```  

그리고 구현된 체인과 툴을 `Graph` 에서 사용할 수 있도록 노드로 정의한다.  

```python
# 그래프 구성 - 노드 정의

from langchain_core.documents import Document

def retrieve(state: GraphState) -> GraphState:
  print('===== [Retrieve] =====')
  question = state['question']
  docs = pdf_retriever.invoke(question)

  return GraphState(documents=docs, question=question)


def generate(state: GraphState) -> GraphState:
  print('===== [Generate] =====')
  question = state['question']
  documents = state['documents']

  generation = pdf_chain.invoke({
      'context' : documents,
      'question' : question,
      'chat_history' : []
  })

  return GraphState(generation=generation, question=question, documents=documents)

def grade_documents(state: GraphState) -> GraphState:
  print('===== [Grade Documents] =====')
  question = state['question']
  documents = state['documents']

  filtered_docs = []

  for d in documents:
    score = retrieval_grader.invoke({'question' : question, 'document' : d})
    if score.binary_score == 'yes':
      print('----- Document relevant -----')
      filtered_docs.append(d)

  return GraphState(documents=filtered_docs, question=question)

def transform_query(state: GraphState) -> GraphState:
  print('===== [Transform Query] =====')
  question = state['question']
  documents = state['documents']

  rewritten_question = question_rewriter.invoke({'question' : question})

  return GraphState(question=rewritten_question, documents=documents)


def web_search(state: GraphState) -> GraphState:
  print('===== [Web Search] =====')
  question = state['question']

  web_search_tool = GoogleSerperAPIWrapper()

  search_query = '날씨 기후와 관련된 주식 종목 정리'

  search_result = web_search_tool.results(search_query)['organic']
  doc_search_result = [
      Document(
          page_content=result.get('title') + ':' + result.get('snippet'),
          metadata={
              'source' : 'web_search',
              'link' : result.get('link')
          }
      )
      for result in search_result
  ]

  return GraphState(documents=search_result, question=question)

```  
