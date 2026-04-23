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
