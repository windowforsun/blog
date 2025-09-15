--- 
layout: single
classes: wide
title: "[LangChain] LangChain Chains for Summary"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain 에서 Chain 을 구성해 문서를 요약하는 다양한 방법론에 대해 알아본다.'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
    - Summary
    - Stuff
    - Map-Reduce
    - Map-Refine
    - Chain of Density
    - Clustering-Map-Refine
toc: true
use_math: true
---  

## Chains for Summary
`LLM` 에서 문서를 요약하는 것은 자연어 처리(NLP)에서 중요한 작업 중 하나이다. 
이번에는 `LangChain` 에서 `Chains` 를 사용하여 문서를 요약하는 방법에 대해 알아본다. 
`LangChain` 의 문서 요약 방법에는 대표적으로 아래와 같은 것들이 있다. 

- `Stuff` : 모든 문서를 한번에 `LLM` 프롬프트에 넣어 요약한다. 
- `Map-Reduce` : 문서를 여러 청크로 나누고, 각 청크를 개별적으로 요약(`Map`)한 뒤, 이 요약본들을 다시 하나로 통합 요약(`Reduce`)한다. 
- `Map-Refine` : 문서를 청크로 나눈 뒤, 첫 번째 청크를 요약하고, 이후 각 청크마다 이전 요약본을 바탕으로 점진적으로 내용을 보완(`Refine`)한다.
- `Chain of Density` : 여러 번 반복적으로 요약을 수행하면서, 매 단계마다 누락된 핵심 엔티티(정보)를 추가로 반영하여 점점 더 졍보 밀도가 높은 요약을 만들어낸다. 
- `Clustering-Map-Refine` : 문서 청크를 의미적으로 `N`개 클러스터로 그룹화한 뒤, 각 클러스터의 중심 청크를 중심으로 `Refine`(점진적 요약) 방식을 적용한다. 


### Stuff
`Stuff` 방식은 문서 전체를 한 번에 요약하는 가장 간단한 방법이다. 
이 방식은 모든 문서를 한 번에 `LLM` 프롬프트에 넣어 요약을 수행한다. 
이 방식은 구현이 가장 쉽고, 소규모 문서에 적합하다. 
대용량 문서의 경우 `LLM` 컨텍스트 제한으로 사용할 수 없을 있어 대용량 문서에는 부적합한 방법이다. 

문서 요약을 위해 아래와 같은 뉴스 기사를 사용한다.  

```text
지식 그래프(Knowledge Graph, KG)라는 대안 DB가 최근 부상하고 있다. Neo4j 같은 KG는 17년 동안 존재했지만, 2012년 구글이 검색 엔진에 부분적으로 KG를 도입하면서 많은 주목을 받았다.\n\nKG는 데이터를 그래프 형식으로 구조화한 DB다. KG의 기본 구성 요소는 \'연결된 노드(node)\'다. 노드는 ‘개체(entity)’를 나타내고, 이들을 연결하는 엣지는 화살표로 두 노드 간 ‘관계(relationship)’를 나타낸다.\n\n방향 있는 ‘아령’같이 생겼다. 많은 경우 로 표현되는 ‘SPO 삼자관계’를 그린다. 예를 들어, ‘히치콕은 “새”를 감독했다’라는 정보를 KG에 저장하자. "히치콕"이라는 노드는 "새"라는 노드를 향해 연결돼 있으며, 엣지는 "감독하다"라는 관계를 의미한다. 또한 "새" 노드를 향해 "로드 테일러"라는 다른 노드가 연결되어 있고, 엣지는 "출연하다"다. 이러한 아령을 많이 겹치게 놓고, 노드와 엣지에 인덱싱을 넣어 그래프 DB를 완성한다.\n\n각 노드와 엣지는 ‘속성’을 지니고 있다. 예로, 히치콕의 노드에는 생년월일이나 국적 등의 속성을 기록한다. 구글 지도의 경우, ‘제일 음식점’이라는 노드에 주소, 영업시간이나 전화번호 같은 속성을 같이 보관하고 필요시 보여준다. KG는 다소 즉흥적인 것처럼 들리지만 경우에 따라 효과적이고 유용하다. 예를 들어, 이 그래프 구조는 구글의 단순한 키워드 기반 검색을 넘어 단어 간 ‘맥락과 관계’를 이해하기 위해 다른 정보끼리를 연결한다.\n\n검색 취지를 더 잘 이해하고, 연계된 의미 있는 답을 낼 수 있다. 예를 들어, “‘새’의 감독이 만든 다른 작품들은 무엇인가?”란 질문에 대해 새-감독-히치콕-감독-현기증의 ‘그래프 줄(multi-hop reasoning)’을 타고 답을 내놓는다. 답이 나온 그래프 줄의 경로도 보여줄 수 있다.\n\nKG의 다른 사례는 하버드 대학교 PrimeKG라는 정밀의료 KG다. 20여개의 의학전문 정보소스를 규합한 KG형 DB로, 질병, 유전자, 단백질, 질병, 표현형, 약물 등 1만7000 노드가, 엣지에는 "연관됨", "상호 작용", "치료 표적", "지시" 및 "부작용"과 같은 4백만 관계가 포함된다. 정밀의료는 개인의 유전, 환경 및 생활패턴을 질병 진단과 치료에 반영하는 의학적 접근 방식이다. 따라서 질병, 약품, 개인 속성의 “관계”에 대한 정보가 핵심이다. 이에 KG가 결정적 역할을 한다. 예를 들면, 약, 질병, 단백질의 관계를 배워 새로운 약을 찾거나 기존 약을 다른 질병에 돌려 적용할 수도 있다. 또, 환자 개인에 맞게 디자인한 처방을 개발할 수도 있다.\n\n최근 새로운 AI 시대를 맞이해, LLM은 KG와 협조 관계로 발전한다. KG는 RAG로 LLM에 연결되어, 이 트리오는 ‘그래프 RAG’를 만든다. 내 회사의 데이터를 KG로 만든 후, RAG로 연결해 LLM과 함께 쓸 수 있는 것이다. 내가 LLM에 자연어로 쿼리를 내면, LLM은 KG 내용을 잡아 자연어로 나에게 답한다. 이를 위해, 사전에 그래프 RAG는 KG의 노드와 엣지를 임베딩하고 벡터DB에 저장해 놓는다. 쿼리가 오면 그를 임베딩한 후 유사치 서치로 벡터DB에서 비슷한 단어들을 축출한다. 여기서 RAG 일이 끝나고 KG에게 바통을 넘긴다. KG는 이 단어들을 기점으로 자기 언어로 KG 안에 관련된 정보를 가져다 LLM에 주면, LLM이 알아서 자연어로 답한다.\n\n이렇게, KG의 구조적으로 정리된 정보, LLM의 언어실력과 이를 연결하는 RAG가 힘을 합쳐 강력한 AI 작품을 만든다. LLM, Neo4j나 CrewAI 같은 제품이 있어 일반 텍스트를 KG로 옮길 수 있다. 게다가 최근 마이크로소프트는 GraphRAG를 개발해 오픈소스로 내놓았으니, KG의 인기는 지속될 것으로 예측된다.\n\n마지막 사례로, 어느 제조업체의 부품에 대한 DB를 생각해 본다. BOM(Bill of Material)은 제품의 구성을 그래프로 표현한다. “제품 A는 부품 A1, A2, A3로 구성되며, 또 A1은 A11과 A12로 구성된다”라는 나무 구조로 돼 있다. 먼저 ‘관계형 DB’에 저장하자. “제품 A에는 무슨 부품이 들어가냐?”라는 질문에 금방 답할 수 있다. 하지만 나무를 거꾸로 들고, “부품 A11은 어느 제품들에 들어가나?”를 물으면 답 얻기가 좀 힘들다. 특히 이 부품이 다른 부품에 껴서 제품 A에 들어가면 아주 힘들다. 즉 ‘부품의 부품’ 같이 손자나 증손자 관계가 맺어지면 관계형 DB는 힘들어 한다.\n\n반면에 ‘KG’라면 그래프 줄을 타고 자연스레 대응한다. 부품 A11 노드에 연결된 모든 엣지를 뒤지고, 그 다음 엣지를 따라 계속 가면 된다. KG는 이런 다단계의 제품-부품 관계뿐 아니라, 제품의 기능, 공장에 대한 정보, 제조사의 여러 공장, 그리고 대체품 등 많은 관계를 저장하고 쉽게 찾아볼 수 있다. 예를 들어, “B 부품 공장이 파업으로 문 닫으면 어떤 제품이 영향을 받으며, 그들의 대체품은 무엇일까?” 혹은 “지진이 자주 일어나는 후쿠시마에는 어떤 1차 혹은 2차 공급자가 있는가?” 라는 질문에 쉽고 빠르게 답을 받을 수 있다. 또한 약간의 코딩으로, 도요타의 RESCUE 시스템처럼, 한 완제품의 BOM과 제조 공장을 나무형으로 그려줄 수도 있고, 공급자들의 공장 들을 전국 지도에 나타낼 수도 있다. 이처럼 ‘관계’가 중요하다면 AI 날개를 단 KG가 효과적인 선택일 수 있다. 하긴 ‘관계’가 중요치 않은 DB가 어디 있을까?
```  

해당 문서를 로드하고 `Stuff` 방식으로 요약하는 코드는 아래와 같다. 

```python
from langchain_community.document_loaders import TextLoader
from langchain_core.prompts import PromptTemplate
from langchain.chains.combine_documents import create_stuff_documents_chain

loader = TextLoader("news-article-llm-rag.txt")
docs = loader.load()

prompt = PromptTemplate.from_template("""
Please summarize the sentence according to the following REQUEST.
REQUEST:
1. Summarize the main points in bullet points.
2. DO NOT translate any technical terms.
3. DO NOT include any unnecessary information.
4. Answer should be written in {language}.

CONTEXT:
{context}

SUMMARY:
""")

stuff_chain = create_stuff_documents_chain(model, prompt)

stuff_summary = stuff_chain.invoke({"context": docs, "language" : "Korean"})
# 요약된 주요 사항은 다음과 같습니다.
# * 지식 그래프(KG)는 데이터를 그래프 형식으로 구조화한 DB입니다.
# * KG의 기본 구성 요소는 연결된 노드(node)와 노드를 연결하는 엣지(edge)입니다.
# * KG는 다소 즉흥적인 것처럼 들리지만 경우에 따라 효과적이고 유용합니다.
# * KG의 예로는 구글 지도가 있으며, KG는 구글의 단순한 키워드 기반 검색을 넘어 단어 간의 맥락과 관계를 이해하기 위해 다른 정보끼리 연결합니다.
# * KG는 정밀의료, 제조업체의 부품 DB 등 다양한 분야에서 활용될 수 있습니다.
# * 최근 새로운 AI 시대를 맞이해, LLM은 KG와 협조 관계로 발전하고 있습니다.
# * KG는 구조적으로 정리된 정보, LLM의 언어실력과 이를 연결하는 RAG가 힘을 합쳐 강력한 AI 작품을 만듭니다.
```  


### Map-Reduce
`Map-Reduce` 방식은 문서를 여러 청크로 나누고, 각 청크를 개별적으로 요약(`Map`)한 뒤, 이 요약본들을 다시 하나로 통합 요약(`Reduce`)하는 방식이다.
대용량 문서나 컨텍스트 윈도우를 초과하는 경우에도 사용 가능하고, 
`Map` 단계가 병렬화되어 처리 속도가 빠르다는 장점이 있다. 
하지만 `Reduce` 단계에서 요약본이 많으면 다시 컨텍스트 한도에 도달할 수 있고, 
각 청크 간의 연관성을 높칠 수 있다.  

예제를 위해 `AI Brief` PDF 문서 중 일부를 사용한다. 

```python
from langchain_community.document_loaders import PyPDFLoader

pdf_loader = PyPDFLoader("./SPRi AI Brief 5월호 산업동향.pdf")
pdf_docs = pdf_loader.load()

pdf_docs_mini = pdf_docs[10:17]
print(len(pdf_docs))
# 7
print(pdf_docs_mini[0])
# Document(metadata={'producer': 'Hancom PDF 1.3.0.505', 'creator': 'Hancom PDF 1.3.0.505', 'creationdate': '2025-05-09T09:07:04+09:00', 'author': 'dj', 'moddate': '2025-05-09T09:07:04+09:00', 'pdfversion': '1.4', 'source': './SPRi AI Brief 5월호 산업동향.pdf', 'total_pages': 28, 'page': 10, 'page_label': '11'}, page_content='정책･법제기업･산업기술･연구인력･교육\n9\n구글, AI 에이전트 간 통신 프로토콜 ‘A2A’ 공개 및 MCP 지원 발표n구글이 에이전트 간 상호운용성을 보장하기 위한 개방형 통신 프로토콜 A2A를 공개했으며, A2A는 에이전트 간 기능 탐색, 작업 관리, 협업, 사용자 경험 협의 등의 다양한 기능을 지원n구글은 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가하기로 했으며, A2A가 MCP보다 상위 계층의 프로토콜로서 MCP를 보완한다고 설명\nKEY Contents\n£A2A, 다중 에이전트 간 협업을 위한 개방형 프로토콜로 설계n구글(Google)이 2025년 4월 9일 50개 이상의 기업*과 협력해 AI 에이전트 간 통신을 위한 개방형 프로토콜 ‘A2A(Agent2Agent)’를 공개* 액센추어(Accenture), 코히어(Cohere), 랭체인(Langchain), 페이팔(Paypal), 세일즈포스(Salesforce) 등∙구글은 다양한 플랫폼과 클라우드 환경에서 다중 AI 에이전트가 서로 통신하고 안전하게 정보를 교환하며 작업을 조정할 수 있도록 A2A 프로토콜을 출시했다고 발표∙구글에 따르면 A2A는 AI 에이전트 간 협업을 위한 표준 방식을 제공하기 위해 HTTP, SSE, JSON-RPC 등 기존 표준을 기반으로 구축되었으며, 기업 환경에서 요구하는 높은 수준의 인증 및 권한 관리 기능을 제공하고 빠른 작업뿐 아니라 장시간 작업 환경에도 적합하며, 텍스트와 오디오, 동영상 스트리밍도 지원nA2A는 작업을 구성하고 전달하는 역할을 하는 클라이언트 에이전트(Client Agent)와 작업을 수행하는 원격 에이전트(Remote Agent) 간 원활한 통신을 위해 다음과 같은 기능을 제공∙(기능 탐색) 각 에이전트가 자신의 기능을 JSON* 형식의 ‘에이전트 카드**’를 통해 공개하면 클라이언트 에이전트는 작업 수행에 가장 적합한 에이전트를 식별해 A2A로 원격 에이전트와 통신* 키-값 쌍으로 이루어진 데이터 객체를 표현하기 위한 텍스트 기반의 개방형 표준 형식** 에이전트의 기능과 스킬, 인증 요구사항 등을 설명하는 공개 메타데이터 파일∙(작업 관리) 클라이언트 에이전트와 원격 에이전트는 최종 사용자의 요청에 대응해 작업 수명주기 전반에서 작업 처리 상태를 지속 동기화하여 처리∙(협업) 각 에이전트는 서로 컨텍스트, 응답, 작업 결과물, 사용자 지시와 같은 메시지를 교환해 협업을 진행∙(사용자 경험 협의) 각 메시지에는 이미지, 동영상, 웹 양식과 같은 특정 콘텐츠 유형이 명시되어 있어, 각 에이전트는 사용자 인터페이스(UI)에 맞게 적절한 콘텐츠 형식을 협의£구글, 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원 발표n한편, 구글 딥마인드(Google Deepmind)의 데미스 하사비스(Demis Hassabis) CEO는 2025년 4월 9일 X를 통해 구글이 앤스로픽의 MCP를 제미나이 모델과 SDK에서 지원하겠다고 발표**  https://x.com/demishassabis/status/1910107859041271977∙구글에 따르면 A2A는 MCP를 보완하는 역할로서, MCP가 LLM을 데이터, 리소스 및 도구와 연결하는 프로토콜이라면 A2A는 에이전트 간 협업을 위한 상위 수준의 프로토콜에 해당 출처 | Google, Announcing the Agent2Agent Protocol (A2A), 2025.04.09.')
```  

`PDF` 문서를 한 장씩 `Map` 단계를 수행한다. 
`Map` 단계를 수행하는 프롬프트와 코드는 아래와 같다.  

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate



map_prompt = ChatPromptTemplate.from_messages([
    ("system", """
    You are a professional main thesis extractor.
    """),
    ("human", """
    Your task is to extract main thesis from given documents. Answer should be in same language as given document. 

    #Format: 
    - thesis 1
    - thesis 2
    - thesis 3
    - ...

    Here is a given document: 
    {doc}

    Write 1~5 sentences.
    #Answer:
    """)
])


map_chain = map_prompt | model | StrOutputParser()


pdf_docs_summaries = map_chain.batch(pdf_docs_mini)

print(pdf_docs_summaries[0])
# - 구글은 AI 에이전트 간 통신 프로토콜 'A2A'를 공개하여 다중 에이전트 간 협업을 위한 개방형 프로토콜을 제공했다.
# - A2A는 에이전트 간 기능 탐색, 작업 관리, 협업, 사용자 경험 협의 등의 다양한 기능을 지원한다.
# - 구글은 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가하기로 했다.
# - A2A는 MCP를 보완하는 역할로서, 에이전트 간 협업을 위한 상위 수준의 프로토콜이다.
# - 구글은 다양한 플랫폼과 클라우드 환경에서 다중 AI 에이전트가 서로 통신하고 안전하게 정보를 교환하며 작업을 조정할 수 있도록 A2A 프로토콜을 출시했다.
```  

모든 페지이가 `Map` 단계를 거쳐 요약된 결과가 만들어 졌으면, 
이를 `Reduce` 단계를 통해 하나의 요약으로 병합한다.  

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

reduce_prompt = ChatPromptTemplate.from_messages([
    ("system", """
    You are a professional summarizer. You are given a list of summaries of documents and you are asked to create a single summary of the documents.
    """),
    ("human", """
    #Instructions: 
    1. Extract main points from a list of summaries of documents
    2. Make final summaries in bullet points format.
    3. Answer should be written in {language}.

    #Format: 
    - summary 1
    - summary 2
    - summary 3
    - ...

    Here is a list of summaries of documents: 
    {doc_summaries}

    #SUMMARY:
    """)
])

reduce_chain = reduce_prompt | model | StrOutputParser()

reduce_answer = reduce_chain.invoke({"doc_summaries": pdf_docs_summaries, "language":"Korean"})
# * 구글은 다중 에이전트 간 협업을 위한 개방형 프로토콜 'A2A'를 공개했다.
# * 메타는 멀티모달 AI 모델 '라마 4' 제품군을 공개했다.
# * 아마존은 웹브라우저 내에서 사용자 대신 다양한 작업을 수행하도록 훈련된 AI 모델 '아마존 노바 액트'를 개발자용 SDK와 함께 공개했다.
# * 오픈AI는 GPT-4.1 제품군을 API로 출시했다.
# * 중국에서 자율주행 보조 기능에 대한 우려가 증대되었다.
# * 포브스는 2025년 50대 AI 기업 목록을 발표했다.
# * 기술 연구의 최근 동향과 발전에 관한 논문이 있다.
```  

위 `Map-Reduce` 를 하나의 체인으로 구성하면 아래와 같다.  

```python
# map-reduce full

from langchain_core.runnables import chain
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

@chain
def map_reduce_chain(docs):
  map_prompt = ChatPromptTemplate.from_messages([
      ("system", """
      You are a professional main thesis extractor.
      """),
      ("human", """
      Your task is to extract main thesis from given documents. Answer should be in same language as given document. 

      #Format: 
      - thesis 1
      - thesis 2
      - thesis 3
      - ...

      Here is a given document: 
      {doc}

      Write 1~5 sentences.
      #Answer:
      """)
  ])
  map_chain = map_prompt | model | StrOutputParser()
  pdf_docs_summaries = map_chain.batch(pdf_docs_mini)

  reduce_prompt = ChatPromptTemplate.from_messages([
      ("system", """
      You are a professional summarizer. You are given a list of summaries of documents and you are asked to create a single summary of the documents.
      """),
      ("human", """
      #Instructions: 
      1. Extract main points from a list of summaries of documents
      2. Make final summaries in bullet points format.
      3. Answer should be written in {language}.
      4. Do not include any content or information other than the document summary in the final answer.

      #Format: 
      - summary 1
      - summary 2
      - summary 3
      - ...

      Here is a list of summaries of documents: 
      {doc_summaries}

      #SUMMARY:
      """)
  ])

  reduce_chain = reduce_prompt | model | StrOutputParser()

  reduce_answer = reduce_chain.invoke({"doc_summaries": pdf_docs_summaries, "language":"Korean"})

  return reduce_answer


answer = map_reduce_chain.invoke({"docs": pdf_docs_mini})
# * 구글은 에이전트 간 상호운용성을 보장하기 위한 개방형 통신 프로토콜 A2A를 공개했다.
# * 메타는 멀티모달 AI 모델 '라마 4' 제품군을 공개했으며, 라마 4는 텍스트, 이미지, 비디오를 함께 처리할 수 있는 멀티모달 기능을 기본 탑재했다.
# * 아마존은 AI 에이전트 구축을 위한 AI 모델 '노바 액트'를 개발자용 SDK와 함께 공개했으며, 노바 액트는 사용자 대신 다양한 작업을 수행하도록 훈련된 AI 모델이다.
# * 오픈AI는 GPT-4.1 API를 출시했으며, 최대 100만 개 토큰의 컨텍스트 창을 지원한다.
# * 자율주행 보조 기능의 안전성 및 과도한 마케팅에 대한 우려가 증폭했다.
# * 포브스는 2025년 50대 AI 기업 목록을 발표했으며, 이 목록에는 오픈AI, 앤스로픽, 싱킹머신랩, 월드랩스 등이 포함되었다.
```  


### Map-Refine
`Map-Refine` 방식은 문서를 청크로 나눈 뒤, 첫 번째 청크를 요약하고, 이후 각 청크마다 이전 요약본을 바탕으로 점진적으로 내용을 보완(`Refine`)하는 방식이다. 
이전 요약을 계속 참고하므로, 문서의 흐름이나 맥락을 더 잘 반영할 수 있고, 
순차적으로 정보가 누적되어, 전체 맥락을 유지한 요약이 가능하다. 
하지만 벙렬 처리가 불가하고, 청크 수가 많으면 시간이 오래 걸릴 수 있다.  

우선 `Map` 단계에서 각 청크를 요약하고 중요한 정보를 추출한다.  

```python
# map 
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

map_summary_prompt = ChatPromptTemplate.from_messages([
    ("system", """
    You are an expert summarizer. Your task is to summarize the following document in {language}.
    """),
    ("human", """
    Extract most important main thesis from the documents, then summarize in bullet points.

    #Forma
    - summary 1
    - summary 2
    - summary 3
    -...

    Here is a given document: 
    {documents}

    Write 1~5 sentences. Think step by step.
    #Summary:
    """)
])

map_summary_chain = map_summary_prompt | model | StrOutputParser()

prompt_datas = [{"documents" : doc, "language" :"Korean"} for doc in pdf_docs_mini]

map_summary_answer = map_summary_chain.batch(prompt_datas)

print(map_summary_answer[0])
# - 구글은 AI 에이전트 간 통신 프로토콜 'A2A(Agent2Agent)'를 공개했다.
# - A2A는 에이전트 간 기능 탐색, 작업 관리, 협업, 사용자 경험 협의 등의 다양한 기능을 지원한다.
# - 구글은 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가하기로 했다.
# - A2A는 MCP를 보완하는 역할로서, 에이전트 간 협업을 위한 상위 수준의 프로토콜이다.
# - A2A 프로토콜은 다중 에이전트 간 협업을 위한 개방형 프로토콜로 설계되어, 다양한 플랫폼과 클라우드 환경에서 사용할 수 있다.
```  

이후 `Refine` 단계에서는 이전 청크까지의 요약과 현재 청크 요약정보를 바탕으로 점진적으로 보완한다. 
최종적으로는 전체 문서에 대한 요약을 생성할 수 있다.  

```python
# refine
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

refine_prompt = ChatPromptTemplate.from_messages([
    ("system", """
    You are an expert summarizer.
    """),
    ("human", """
    Your job is to produce a final summary

    We have provided an existing summary up to a certain point:
    previous summary: {previous_summary}

    Update the summary including the above with the summary below.
    ------------
    {current_summary}
    ------------
    Given the new context, refine the original summary in {language}.
    If the context isn't useful, return the original summary.
    """)
])

refine_chain = refine_prompt | model | StrOutputParser()

refine_answer = refine_chain.invoke({"previous_summary": map_summary_answer[0], 
                                     "current_summary": map_summary_answer[1], 
                                     "language":"Korean"})
# 구글은 AI 에이전트 간 통신 프로토콜 'A2A(Agent2Agent)'를 공개했으며, A2A는 에이전트 간 협업을 위한 표준 방식을 제공하기 위해 구축되었다. A2A 프로토콜은 다양한 플랫폼과 클라우드 환경에서 다중 AI 에이전트가 서로 통신하고 안전하게 정보를 교환하며 작업을 조정할 수 있도록 설계되었다. A2A는 작업을 구성하고 전달하는 역할을 하는 클라이언트 에이전트와 작업을 수행하는 원격 에이전트 간 원활한 통신을 위해 기능 탐색, 작업 관리, 협업, 사용자 경험 협의 등의 기능을 제공한다. 또한, 구글은 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가하기로 했으며, A2A가 MCP보다 상위 계층의 프로토콜로서 MCP를 보완한다고 설명했다. A2A 프로토콜은 기업 환경에서 요구하는 높은 수준의 인증 및 권한 관리 기능을 제공하고 빠른 작업뿐 아니라 장시간 작업 환경에도 적합하며, 텍스트와 오디오, 동영상 스트리밍도 지원한다.
# 
# 한편, 메타는 멀티모달 AI 모델 '라마 4' 제품군을 공개하였으며, 라마 4 스카우트와 라마 4 매버릭을 출시하고 라마 4 베히모스는 프리뷰로 공개되었습니다. 라마 4는 라마 시리즈 최초의 전문가혼합 모델로 설계되었으며, 텍스트뿐 아니라 이미지, 비디오를 함께 처리할 수 있는 멀티모달 기능을 기본 탑재하였습니다. 라마 4 스카우트는 활성 매개변수 170억 개에 16개의 전문가 모델로 구성되어 있으며, 주요 벤치마크 평가에서 유사 크기의 경쟁 모델보다 우수한 성능을 기록하였습니다. 메타는 라마 4의 성능 평가에 사용한 버전과 실제 개발자에게 제공되는 버전 간의 성능 차이로 인해 테스트셋으로 모델을 학습시켰다는 의혹을 부인했습니다. 메타의 아흐마드 알달레 사장은 이러한 의혹은 전혀 사실이 아니라고 반박하며, 모델이 준비되자마자 공개했기 때문에 기능 안정화에는 시간이 필요하다고 해명했습니다. 이러한 새로운 기술 개발과 공개는 AI 분야의 발전에 기여할 것으로 예상됩니다.
```  

위 `Map-Refine` 를 하나의 체인으로 구성하면 아래와 같다.

```python
# map-refine full

from langchain_core.runnables import chain
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

@chain
def map_refine_chain(docs):
  map_summary_prompt = ChatPromptTemplate.from_messages([
      ("system", """
      You are an expert summarizer. Your task is to summarize the following document in {language}.
      """),
      ("human", """
      Extract most important main thesis from the documents, then summarize in bullet points.

      #Format:
      - summary 1
      - summary 2
      - summary 3
      -...

      Here is a given document: 
      {documents}

      Write 1~5 sentences. Think step by step.
      #Summary:
      """)
  ])

  map_summary_chain = map_summary_prompt | model | StrOutputParser()

  prompt_datas = [{"documents" : doc, "language" :"Korean"} for doc in pdf_docs_mini]

  map_summary_answer = map_summary_chain.batch(prompt_datas)

  refine_prompt = ChatPromptTemplate.from_messages([
      ("system", """
      You are an expert summarizer.
      """),
      ("human", """
      Your job is to produce a final summary

      We have provided an existing summary up to a certain point:
      previous summary: {previous_summary}

      Update the summary including the above with the summary below.
      ------------
      {current_summary}
      ------------
      Given the new context, refine the original summary in {language}.
      If the context isn't useful, return the original summary.
      """)
  ])

  refine_chain = refine_prompt | model | StrOutputParser()

  previous_summary = map_summary_answer[0]

  for current_summary in map_summary_answer[1:]:
    previous_summary = refine_chain.invoke({
        "previous_summary": previous_summary, 
        "current_summary": current_summary, 
        "language":"Korean"
    })

    print(previous_summary)
    print("==================================================")

  return previous_summary


answer = map_refine_chain.invoke(pdf_docs_mini)
# 구글은 AI 에이전트 간 상호운용성을 보장하기 위한 개방형 통신 프로토콜 'A2A(Agent2Agent)'를 공개했다.
# - A2A는 에이전트 간 기능 탐색, 작업 관리, 협업, 사용자 경험 협의 등의 다양한 기능을 지원한다.
# - A2A 프로토콜은 다중 에이전트 간 협업을 위한 표준 방식을 제공하기 위해 설계되었다.
# - 구글은 제미나이 모델과 SDK에서 앤στρο픽의 MCP 지원을 추가하기로 했으며, A2A가 MCP보다 상위 계층의 프로토콜로서 MCP를 보완한다고 설명했다.
# - A2A 프로토콜은 다양한 플랫폼과 클라우드 환경에서 다중 AI 에이전트가 서로 통신하고 안전하게 정보를 교환하며 작업을 조정할 수 있도록 설계되었다.
# 한편, 메타는 최근 멀티모달 AI 모델 '라마 4' 제품군을 공개했으며, 이 모델은 텍스트, 이미지, 비디오를 함께 처리할 수 있는 멀티모달 기능을 기본 탑재한 전문가혼합 모델입니다. 라마 4는 이미 일부 제품이 출시되었으며, 다양한 벤치마크 평가에서 우수한 성능을 기록했습니다. 그러나 일부 AI 연구자들은 메타가 성능을 조작했다는 의혹을 제기했지만, 메타는 이를 부인했습니다.
# 또한, 아마존은 웹브라우저 내에서 사용자 대신 다양한 작업을 수행하도록 훈련된 AI 모델 '아마존 노바 액트'를 개발자용 SDK와 함께 공개했다.
# - 노바 액트는 벤치마크 평가에서 앤스로픽의 클로드 3.7 소네트와 오픈AI CUA를 능가했으며, 아마존의 AI 기반 음성 비서 '알렉사 플러스'에도 적용되었다.
# - 노바 액트는 복잡한 작업을 세부적인 작업 단위로 나누어 처리할 수 있도록 설계되어 사용자 대신 작업을 수행하는 안정성을 강화한다.
# - 아마존은 노바 액트가 반복 작업을 자동 실행하도록 설정 가능하며, 예를 들어 매주 같은 시간에 샐러드를 자동으로 주문하는 등 다양한 작업을 수행할 수 있다.
# - 노바 액트는 화면 없이 백그라운드에서 작동하게 하거나 AI 에이전트를 API 형태로 만들어 제품에 통합할 수 있다.
# さらに, 오픈AI는 GPT-4.1 API를 출시했으며, 최대 100만 개 토큰의 컨텍스트 창을 지원하고, 전반적인 성능이 GPT-4o와 GPT-4o 미니를 능가한다.
# - GPT-4.1 기본 모델은 코딩, 지시이행, 장문 컨텍스트 이해 능력에서 GPT-4o를 능가한다.
# - 오픈AI는 또한 추론 모델 o3와 o4-미니를 출시했으며, o3는 멀티모달과 코딩 관련 벤치마크 평가에서 최고 점수를 기록했다.
# - 그러나 o3와 o4-미니의 환각 현상은 o1보다 심해졌으며, 오픈AI는 환각 증가의 원인을 규명하기 위해 더 많은 연구가 필요하다고 밝혔다.
# - GPT-4.1 미니와 나노 모델은 각각 비슷하거나 더 뛰어난 지능을 발휘하면서 지연 시간과 비용을 줄였다.
# 최근 중국에서 자율주행 보조 기능을 탑재한 샤오미 전기차가 충돌사고로 3명이 사망하면서 자율주행 기능의 안전성 및 과도한 마케팅에 대한 우려가 증폭되고 있다. 이러한 기술 개발은 AI 에이전트의 협업과 상호운용성을 향상시키며, 사용자에게 더 편리하고 효율적인 서비스를 제공할 수 있을 것으로 기대된다. 그러나 안전성과 마케팅에 대한 우려도 함께 증가하고 있어, 기술 개발과 함께 안전성 및 책임성에 대한 고려가 필요할 것으로 보인다.
# 또한, 포브스는 2025년 50대 AI 기업 목록을 발표했으며, 이 목록에는 오픈AI와 앤스로픽을 비롯한 여러 기업이 포함되어 있습니다.
# - 이 목록은 AI 에이전트로의 전환 추세를 반영하며, 단순히 명령에 응답하는 AI를 넘어 문제를 해결하고 전체 작업을 완료할 수 있는 AI 에이전트가 주류로 부상할 것으로 예상됩니다.
# - 포브스는 2026년에는 사용자 대신 모든 업무를 처리하는 AI 에이전트가 주류가 될 것으로 전망합니다.
# - 목록에 포함된 기업 중에는 법률 AI 스타트업 하비와 애니스피어 등이 있으며, 이 기업들은 업무용 AI 도구를 제공하고 있습니다.
# - 포브스의 2025년 AI 50 목록은 AI 기술의 발전과 미래의 전망을 보여주는 중요한 지표입니다.
# 이러한 기술 개발과 기업 동향은 AI 분야의 빠른 발전과 사용자에게 제공되는 서비스의 다양화와 개선을 의미하며, 앞으로의 기술 발전과 함께 책임성과 안전성에 대한 관심도 함께 증가할 것으로 예상된다.
```  

### Chain of Density
반복적으로 요약을 수행하며, 매 단계마다 누락된 핵심 정보를 추가해 점점 더 정보 밀도가 높은 요약을 만들어내는 방식이다. 
정보가 누락되지 않도록 반복적으로 보완해 매우 압축적이면서도 풍부한 요약을 생성하고, 
복잡한 긴 문서에서 핵심 정보를 최대한 많이 담아낼 수 있다. 
하지만 반복적으로 `LLM` 을 호출하므로 비용과 시간이 증가하고, 
프롬프트 설계가 다소 복잡해 질 수 있다.  

- 점진적 개선 : 초기에는 간단한 요약을 생성한 뒤, 단계적으로 중요한 개체들을 추가하며 개선한다. 과정이 반복되더라도 요약 길이는 유지되면서 정보의 밀도는 높아져 읽기 슆고 풍부한 요약이 만들어진다. 
- 정보 밀도와 가독성 균형 : 요약의 정보 밀도를 계속 조절하며 정보성과 가독성 사이의 최적 균형점을 찾는다. 
- 추상화 및 정보 융합 : 추상적이고 융합성이 뛰어나다. 원문의 앞부분에 치우치는 경향이 덜하다. 

`CoD` 를 구현하는 프롬프트와 코드는 아래와 같다. 
먼저 `Cod` 의 프롬프틑 아래와 같은 입력 파라미터를 사용한다.  

- `content_category` : 요약할 문서의 종류 (기사, 논문, 블로그 등)
- `content` : 요약할 컨텐츠
- `entity_range` : 컨텐츠에서 선택하여 요약에 추가할 엔티티 수의 범위
- `max_words` : 1번 요약을 수행할 때 포함할 최대 단어 수
- `iterations` : 엔티티 고도화 라운드 수

```python
import textwrap
from langchain_core.output_parsers import SimpleJsonOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough

cod_chain_input = {
    "content" : lambda d : d.get("content"),
    "content_category" : lambda d: d.get("content_category", "Article"),
    "entity_range" : lambda d: d.get("entity_range", "1-3"),
    "max_words" : lambda d: int(d.get("max_words", 80)),
    "iterations" : lambda d: int(d.get("iterations", 5)),
    "language" : lambda d: d.get("language", "Korean")
}

cod_prompt = ChatPromptTemplate.from_messages([
    ("system", """
    As an expert copy-writer, you will write increasingly concise, entity-dense summaries of the user provided {content_category}. The initial summary should be under {max_words} words and contain {entity_range} informative Descriptive Entities from the {content_category}.

    A Descriptive Entity is:
    - Relevant: to the main story.
    - Specific: descriptive yet concise (5 words or fewer).
    - Faithful: present in the {content_category}.
    - Anywhere: located anywhere in the {content_category}.

    # Your Summarization Process
    - Read through the {content_category} and the all the below sections to get an understanding of the task.
    - Pick {entity_range} informative Descriptive Entities from the {content_category} (";" delimited, do not add spaces).
    - In your output JSON list of dictionaries, write an initial summary of max {max_words} words containing the Entities.
    - You now have `[{{"missing_entities": "...", "denser_summary": "..."}}]`

    Then, repeat the below 2 steps {iterations} times:

    - Step 1. In a new dict in the same list, identify {entity_range} new informative Descriptive Entities from the {content_category} which are missing from the previously generated summary.
    - Step 2. Write a new, denser summary of identical length which covers every Entity and detail from the previous summary plus the new Missing Entities.

    A Missing Entity is:
    - An informative Descriptive Entity from the {content_category} as defined above.
    - Novel: not in the previous summary.

    # Guidelines
    - The first summary should be long (max {max_words} words) yet highly non-specific, containing little information beyond the Entities marked as missing. Use overly verbose language and fillers (e.g., "this {content_category} discusses") to reach ~{max_words} words.
    - Make every word count: re-write the previous summary to improve flow and make space for additional entities.
    - Make space with fusion, compression, and removal of uninformative phrases like "the {content_category} discusses".
    - The summaries should become highly dense and concise yet self-contained, e.g., easily understood without the {content_category}.
    - Missing entities can appear anywhere in the new summary.
    - Never drop entities from the previous summary. If space cannot be made, add fewer new entities.
    - You're finished when your JSON list has 1+{iterations} dictionaries of increasing density.

    # IMPORTANT
    - Remember, to keep each summary to max {max_words} words.
    - Never remove Entities or details. Only add more from the {content_category}.
    - Do not discuss the {content_category} itself, focus on the content: informative Descriptive Entities, and details.
    - Remember, if you're overusing filler phrases in later summaries, or discussing the {content_category} itself, not its contents, choose more informative Descriptive Entities and include more details from the {content_category}.
    - Answer with a minified JSON list of dictionaries with keys "missing_entities" and "denser_summary".
    - "denser_summary" should be written in the same language as the "content".
    - Answer should be written in {language}.

    ## Example output
    [{{"missing_entities": "ent1;ent2", "denser_summary": "<vague initial summary with entities 'ent1','ent2'>"}}, {{"missing_entities": "ent3", "denser_summary": "denser summary with 'ent1','ent2','ent3'"}}, ...]
    """),
    ("human", """
    {content_category}:
    {content}
    """)
])

cod_chain = (
        cod_chain_input
        | cod_prompt
        | model
        | SimpleJsonOutputParser()
)

cod_summary = cod_chain.invoke({
    "content" : pdf_docs_mini[0],
    "content_category" : "Article",
    "iterations" : 5
})


# [{'missing_entities': 'A2A;에이전트',
#   'denser_summary': '구글이 에이전트 간 통신 프로토콜 A2A를 공개했으며, 에이전트 간 상호운용성을 보장하기 위한 개방형 프로토콜입니다.'},
#  {'missing_entities': 'MCP',
#   'denser_summary': '구글이 에이전트 간 통신 프로토콜 A2A를 공개했으며, 에이전트 간 상호운용성을 보장하기 위한 개방형 프로토콜이고, A2A는 MCP를 보완하는 역할입니다.'},
#  {'missing_entities': '제미나이',
#   'denser_summary': '구글이 에이전트 간 통신 프로토콜 A2A를 공개했으며, 에이전트 간 상호운용성을 보장하기 위한 개방형 프로토콜이고, A2A는 MCP를 보완하는 역할이며, 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가합니다.'},
#  {'missing_entities': 'HTTP;SSE;JSON-RPC',
#   'denser_summary': '구글이 에이전트 간 통신 프로토콜 A2A를 공개했으며, 에이전트 간 상호운용성을 보장하기 위한 개방형 프로토콜이고, A2A는 MCP를 보완하는 역할이며, 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가하고, HTTP, SSE, JSON-RPC 등 기존 표준을 기반으로 구축되었습니다.'},
#  {'missing_entities': '클라이언트 에이전트;원격 에이전트',
#   'denser_summary': '구글이 에이전트 간 통신 프로토콜 A2A를 공개했으며, 에이전트 간 상호운용성을 보장하기 위한 개방형 프로토콜이고, A2A는 MCP를 보완하는 역할이며, 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가하고, HTTP, SSE, JSON-RPC 등 기존 표준을 기반으로 구축되었으며, 클라이언트 에이전트와 원격 에이전트 간 원활한 통신을 제공합니다.'},
#  {'missing_entities': '에이전트 카드',
#   'denser_summary': '구글이 에이전트 간 통신 프로토콜 A2A를 공개했으며, 에이전트 간 상호운용성을 보장하기 위한 개방형 프로토콜이고, A2A는 MCP를 보완하는 역할이며, 제미나이 모델과 SDK에서 앤스로픽의 MCP 지원을 추가하고, HTTP, SSE, JSON-RPC 등 기존 표준을 기반으로 구축되었으며, 클라이언트 에이전트와 원격 에이전트 간 원활한 통신을 제공하고, 에이전트 카드를 통해 에이전트의 기능을 공개합니다.'}]
```  

이후 `Refine` 등 작업을 사용해 `cod_summary` 의 마지막 요약들을 합쳐 전체 요약본을 생성할 수 있다.  
