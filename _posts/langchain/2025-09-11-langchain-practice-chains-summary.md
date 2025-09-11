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

