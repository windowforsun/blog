--- 
layout: single
classes: wide
title: "[LangGraph] LangGraph RAG Structure"
header:
  overlay_image: /img/langgraph-img-2.jpeg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - LangGraph
tags:
    - Practice
    - LangChain
    - LangGraph
toc: true
use_math: true
---  


## Web Search
앞서 구현한 [LangGraph RAG]({{site.baseurl}}{% link _posts/langgraph/2026-02-01-langgraph-practice-naive-rag-relevant.md %})
구성에서 관련성 검증에 실패한 경우, 
`Web Search` 를 통해 추가적인 문서를 검색하고, 이를 다시 `llm` 에 던져 답변을 생성하는 흐름을 구현해 본다. 

웹 검색으로는 `GoogleSerperAPIWrapper` 를 사용한다. 
웹 검색 노드 함수를 구현하면 아래와 같다.  

```python
# 웹 검색 노드 추가
from langchain_community.utilities import GoogleSerperAPIWrapper
import json

os.environ["SERPER_API_KEY"] = "api key"

def web_search(state: GraphState) -> GraphState:
  web_search_tool = GoogleSerperAPIWrapper()

  search_query = state['question']

  search_result = web_search_tool.results(search_query)['organic']
  print(search_result)

  return GraphState(context="\n".join(json.dumps(search_result)))
```  

이제 이전에 정의했던 내용들에 웹 검색 노드를 추가하여 전체 그래프를 구성한다. 

```python
# 그래프 구성
from langgraph.graph import START, END, StateGraph
from langgraph.checkpoint.memory import MemorySaver
from IPython.display import Image, display

graph_builder = StateGraph(GraphState)

graph_builder.add_node('retrieve', retrieve_document)
graph_builder.add_node('relevance_check', relevance_check)
graph_builder.add_node('llm_answer', llm_anwser)
graph_builder.add_node('web_search', web_search)

graph_builder.add_edge('retrieve', 'relevance_check')
graph_builder.add_conditional_edges(
    'relevance_check',
    is_relevant,
    {
        'yes': 'llm_answer',
        'no' : 'web_search'
    }
)

graph_builder.add_edge('web_search', 'llm_answer')
graph_builder.set_entry_point('retrieve')

memory = MemorySaver()

graph = graph_builder.compile(memory)



# 그래프 시각화
try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    pass
```  

![그림 1]({{site.baseurl}}/img/langgraph/web-search-query-rewrite-1.png)


`RAG` 에 사용한 문서와 관련된 질의의 경우 `Retrieve` 결과를 바탕으로 최종 답변을 생성한다.  

```python
# 관련성 있는 쿼리로 그래프 실행

from langchain_core.runnables import RunnableConfig

config = RunnableConfig(recursion_limit=20, configurable={'thread_id' : '3'})

inputs = GraphState(question='지금까지 각 월의 평균 기온에 대해 정리해줘')

execute_graph(graph, config, inputs)
# retrieve
# {'context': '<document><content>하거나\x01적었습니다.\n평균기온(℃) 강수량(㎜)\n낮음 비슷 높음 적음 비슷 많음\n※\x01네모\x01박스\x01위:\x01월\x01평균값(℃)/편차(℃),\x01아래:\x01평년(1991~2020년)\x01비슷범위(℃) ※\x01네모\x01박스\x01위:\x01월\x01누적값(㎜),\x01아래:\x01평년(1991~2020년)\x01비슷범위(㎜)\n평균기온\x01확률밀도분포\n▶\x01채색:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(4월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온\x01분포\n▶\x01점선:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(4월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>5</page></document>\n<document><content>평년과\x01비슷하거나\x01적었습니다.\n평균기온(℃) 강수량(㎜)\n낮음 비슷 높음 적음 비슷 많음\n※\x01네모\x01박스\x01위:\x01월\x01평균값(℃)/편차(℃),\x01아래:\x01평년(1991~2020년)\x01비슷범위(℃) ※\x01네모\x01박스\x01위:\x01월\x01누적값(㎜),\x01아래:\x01평년(1991~2020년)\x01비슷범위(㎜)\n평균기온\x01확률밀도분포\n▶\x01채색:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2023년(3월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온\x01분포\n▶\x01점선:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2023년(3월\x01평균기온\x011위),\x01(초록)평년\x01월평균기온</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>5</page></document>\n<document><content>이\x01평년과\x01비슷하거나\x01많았습니다.\n평균기온(℃) 강수량(㎜)\n낮음 비슷 높음 적음 비슷 많음\n※\x01네모\x01박스\x01위:\x01월\x01평균값(℃)/편차(℃),\x01아래:\x01평년(1991~2020년)\x01비슷범위(℃) ※\x01네모\x01박스\x01위:\x01월\x01누적값(㎜),\x01아래:\x01평년(1991~2020년)\x01비슷범위(㎜)\n평균기온\x01확률밀도분포\n▶\x01채색:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(6월\x01평균기온\x012위),\x01(초록)평년\x01월평균기온\x01분포\n▶\x01점선:\x01우리나라\x0166개\x01지점\x01(빨강)2025년,\x01(보라)2024년(6월\x01평균기온\x012위),\x01(초록)평년\x01월평균기온</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>5</page></document>\n<document><content>년보다\x01낮은\x01기온이\x01지속되다가,\x01이후에는\x01대체로\x01평년\x01수준으로\x01회복되었으나,\x0120~21일에는\x01기온이\x01일시적으로\x01크게\x01올랐습니다.\x01\n※\x01평년\x01비슷범위:\x0117.0~17.6℃\n2025년\x015월\x01평균기온/평균\x01최고기온/평균\x01최저기온\x01(1973년\x01이후\x01전국평균)\n2025년\x015월\n구분\n평균값\x01(℃) 평년값\x01(℃) 평년편차\x01(℃) 순위(상위)\n평균기온 16.8 17.3 -0.5 33위\n평균\x01최고기온 22.4 23.5 -1.1 45위\n평균\x01최저기온 11.5 11.6 -0.1 23위</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf</source><page>1</page></document>\n<document><content>2025년\n2025.\x017.\x014.\x01발간\n6월호\n6월\x01기후\x01동향\n기온\n6월\x01기온\x01시계열\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01막대:\x012025년\x016월\x01전국\x0166개\x01지점의\x01일별\x01(빨강)최고기온\x01범위,\x01(파랑)최저기온\x01범위\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01실선:\x012025년\x016월\x01전국\x0166개\x01지점\x01평균\x01일별\x01(초록)평균기온,\x01(빨강)최고기온,\x01(파랑)최저기온\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01점:\x011973~2025년\x016월\x01전국\x0166개\x01지점\x01기준\x01일별\x01(빨강)최고기온\x01극값,\x01(파랑)최저기온\x01극값</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>1</page></document>\n<document><content>평균기온 22.9 21.4 +1.5 1위\n평균\x01최고기온 28.2 26.7 +1.5 2위\n평균\x01최저기온 18.2 16.8 +1.4 3위\n※\x01전국평균:\x011973년\x01이후부터\x01연속적으로\x01관측한\x01전국\x0162개\x01지점의\x01관측자료를\x01활용((1973~1989년)\x0156개\x01지점,\x01(1990~2025년)\x0162개\x01지점)\n※\x01평년값:\x011991~2020년\x01적용\n1</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>1</page></document>\n<document><content>2025년\x014월\n구분\n평균값\x01(℃) 평년값\x01(℃) 평년편차\x01(℃) 순위(상위)\n평균기온 13.1 12.1 1.0 10위\n평균\x01최고기온 19.7 18.6 1.1 9위\n평균\x01최저기온 6.6 6.0 0.6 18위\n※\x01전국평균:\x011973년\x01이후부터\x01연속적으로\x01관측한\x01전국\x0162개\x01지점의\x01관측자료를\x01활용((1973~1989년)\x0156개\x01지점,\x01(1990~2025년)\x0162개\x01지점)\n※\x01평년값:\x011991~2020년\x01적용\n1</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>1</page></document>\n<document><content>2025년\n2025.\x014.\x013.\x01발간\n3월호\n3월\x01기후\x01동향\n기온\n3월\x01기온\x01시계열\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01막대:\x012025년\x013월\x01전국\x0166개\x01지점의\x01일별\x01(빨강)최고기온\x01범위,\x01(파랑)최저기온\x01범위\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01실선:\x012025년\x013월\x01전국\x0166개\x01지점\x01평균\x01일별\x01(초록)평균기온,\x01(빨강)최고기온,\x01(파랑)최저기온\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01점:\x011973~2025년\x013월\x01전국\x0166개\x01지점\x01기준\x01일별\x01(빨강)최고기온\x01극값,\x01(파랑)최저기온\x01극값</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>1</page></document>\n<document><content>관측자료\n4월\x01해수면\x01온도\x01시계열(℃)\n4월\x01전\x01해상\x01일별\x01평균해수면\x01온도\x01(파랑)\x01최근\x0110년\x01(2015~2024년),\x01(빨강)\x012025년\n2015년\x01이후부터\x01연속적으로\x01관측한\x01해양기상부이\x0111개\x01지점의\x01관측자료를\x01활용\n2025년\x014월\x01평균\x01해수면온도\x01분포 최근\x0110년\x014월\x01평균\x01해수면온도\x01분포\n재분석자료(OISST)\n2025년\x014월\x01평균\x01해수면온도\x01분포 평년(1991~2020년)\x01편차</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>8</page></document>\n<document><content>2025년\n2025.\x015.\x017.\x01발간\n4월호\n4월\x01기후\x01동향\n기온\n4월\x01기온\x01시계열\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01막대:\x012025년\x014월\x01전국\x0166개\x01지점의\x01일별\x01(빨강)최고기온\x01범위,\x01(파랑)최저기온\x01범위\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01실선:\x012025년\x014월\x01전국\x0166개\x01지점\x01평균\x01일별\x01(초록)평균기온,\x01(빨강)최고기온,\x01(파랑)최저기온\n\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01\x01▶\x01점:\x011973~2025년\x014월\x01전국\x0166개\x01지점\x01기준\x01일별\x01(빨강)최고기온\x01극값,\x01(파랑)최저기온\x01극값</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>1</page></document>'}
# ===== relevant check =====
# yes
# relevance_check
# {'relevance': 'yes'}
# llm_answer
# {'answer': '제공된 문맥에 따르면 2025년 3월, 4월, 5월, 6월의 평균 기온은 다음과 같습니다. 3월 평균 기온 데이터는 제공되지 않습니다.\n\n*   **4월:** 평균 기온 13.1℃\n*   **5월:** 평균 기온 16.8℃\n*   **6월:** 평균 기온 22.9℃\n\n**출처:**\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf (페이지 1)', 'messages': [('user', '지금까지 각 월의 평균 기온에 대해 정리해줘'), ('assistant', '제공된 문맥에 따르면 2025년 3월, 4월, 5월, 6월의 평균 기온은 다음과 같습니다. 3월 평균 기온 데이터는 제공되지 않습니다.\n\n*   **4월:** 평균 기온 13.1℃\n*   **5월:** 평균 기온 16.8℃\n*   **6월:** 평균 기온 22.9℃\n\n**출처:**\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf (페이지 1)\n*   /content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf (페이지 1)')]}
```  

문서와 관련성이 없는 질의의 경우 `web_search` 노드를 통해 추가 정보를 검색하고 최종 답변이 생성된다.  

```python
# 관련성 없는 쿼리로 그래프 실행

from langchain_core.runnables import RunnableConfig

config = RunnableConfig(recursion_limit=5, configurable={'thread_id' : '6'})

inputs = GraphState(question='비트코인 현재 가격')

execute_graph(graph, config, inputs)
# retrieve
# {'context': '<document><content>※\x01평년값:\x011991~2020년\x01적용\n2</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>2</page></document>\n<document><content>년/월 기준\n4월 5월 6월 7월 8월 9월 10월 11월 12월 1월 2월 3월\n편차(℃) 1.28 1.19 1.22 1.21 1.27 1.26 1.34 1.34 1.28 1.36 1.26 1.31 1901\x01~\x012000년\n순위(상위) 1 1 1 1 1 2 2 2 2 1 3 3 1880\x01~\x012025년</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>11</page></document>\n<document><content>전\x01지구\x01월별\x01기온\x01편차와\x01순위\x01(2024년\x013월\x01~\x012025년\x012월)\n2024년 2025년\n년/월 기준\n3월 4월 5월 6월 7월 8월 9월 10월 11월 12월 1월 2월\n편차(℃) 1.34 1.27 1.19 1.22 1.22 1.27 1.25 1.33 1.32 1.29 1.33 1.26 1901\x01~\x012000년\n순위(상위) 1 1 1 1 1 1 2 2 2 2 1 3 1880\x01~\x012025년</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>11</page></document>\n<document><content>전\x01지구\x01월별\x01기온\x01편차와\x01순위\x01(2024년\x015월\x01~\x012025년\x014월)\n2024년 2025년\n년/월 기준\n5월 6월 7월 8월 9월 10월 11월 12월 1월 2월 3월 4월\n편차(℃) 1.18 1.22 1.22 1.27 1.25 1.34 1.32 1.29 1.34 1.25 1.32 1.22 1901\x01~\x012000년\n순위(상위) 1 1 1 1 2 2 2 2 1 3 3 2 1880\x01~\x012025년</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf</source><page>11</page></document>\n<document><content>\x01\x01\x01\x01\x01\x01\x01((1973~1989년)\x0156개\x01지점,\x01(1990~2025년)\x0162개\x01지점)\n우리나라\x01월별\x01평균기온\x01평년편차와\x01순위\x01(2024년\x016월\x01~\x012025년\x015월)\n2024년 2025년\n년/월 기준\n6월 7월 8월 9월 10월 11월 12월 1월 2월 3월 4월 5월\n월평균(℃) 22.7 26.2 27.9 24.7 16.1 9.7 1.8 -0.2 -0.5 7.6 13.1 16.8</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf</source><page>5</page></document>\n<document><content>전\x01지구\x01월별\x01기온\x01편차와\x01순위\x01(2024년\x016월\x01~\x012025년\x015월)\n2024년 2025년\n년/월 기준\n6월 7월 8월 9월 10월 11월 12월 1월 2월 3월 4월 5월\n편차(℃) 1.22 1.22 1.27 1.25 1.34 1.34 1.30 1.34 1.27 1.34 1.21 1.10 1901\x01~\x012000년\n순위(상위) 1 1 1 2 2 2 2 1 3 1 2 2 1880\x01~\x012025년</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>11</page></document>\n<document><content>10</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_03.pdf</source><page>10</page></document>\n<document><content>10</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_04.pdf</source><page>10</page></document>\n<document><content>10</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_05.pdf</source><page>10</page></document>\n<document><content>10</content><source>/content/drive/MyDrive/Colab Notebooks/data/rag/weather-docs/ellinonewsletter_2025_06.pdf</source><page>10</page></document>'}
# ===== relevant check =====
# no
# relevance_check
# {'relevance': 'no'}
# [{'title': 'Bitcoin - 비트코인 가격 (BTC 암호화폐) - Investing.com - 인베스팅닷컴', 'link': 'https://kr.investing.com/crypto/bitcoin', 'snippet': '비트코인, 사상 첫 12만달러 돌파… 이더리움도 3000달러 기록. 비트코인이 12만달러(약 1억6545만원)대를 돌파했다. 14일 오후 1시 현재 글로벌 코인 시황 중계사이트인.', 'sitelinks': [{'title': 'BTC USD', 'link': 'https://kr.investing.com/crypto/bitcoin/btc-usd'}, {'title': '차트', 'link': 'https://kr.investing.com/crypto/bitcoin/chart'}, {'title': '포럼', 'link': 'https://kr.investing.com/crypto/bitcoin/chat'}, {'title': 'BTC KRW', 'link': 'https://kr.investing.com/crypto/bitcoin/btc-krw'}], 'position': 1}, {'title': '업비트 upbit 비트코인 거래소', 'link': 'https://upbit.com/exchange/CRIX.UPBIT.KRW-BTC', 'snippet': '비트코인 시세, 비트코인 가격, 비트코인 거래, 비트코인 정보 등 비트코인의 모든 것, 가장 신뢰받는 디지털 자산 거래소 업비트에서 확인해보세요.', 'position': 2}, {'title': '비트코인 시세: BTC 가격 차트, 시가총액 & 오늘의 뉴스 | CoinGecko', 'link': 'https://www.coingecko.com/ko/%EC%BD%94%EC%9D%B8/%EB%B9%84%ED%8A%B8%EC%BD%94%EC%9D%B8', 'snippet': '비트코인(BTC)의 가격은 현재 US$122,297, 24시간 거래량은 US$47,512,190,251입니다. 최근 24시간 내 가격은 3.71% 상승했으며, 지난 7일 내에는 12.40% 상승했습니다.', 'position': 3}, {'title': 'BTC USD — 비트코인 가격 및 차트 - 트레이딩뷰', 'link': 'https://kr.tradingview.com/symbols/BTCUSD/', 'snippet': '비트코인(BTC) 의 현재 시가총액은 \u202a2.42 T\u202c USD 입니다. 이 숫자를 맥락에서 확인하려면 시가총액별로 순위가 매겨진 크립토 코인 리스트를 체크하거나 크립토 마켓 캡 ...', 'sitelinks': [{'title': 'Btcusd 뉴스', 'link': 'https://kr.tradingview.com/symbols/BTCUSD/news/'}, {'title': 'BTCUSD Perpetual Contract', 'link': 'https://kr.tradingview.com/symbols/BTCUSD.P/'}, {'title': 'Btcusd 트레이딩 아이디어', 'link': 'https://kr.tradingview.com/symbols/BTCUSD/ideas/'}, {'title': '토론', 'link': 'https://kr.tradingview.com/symbols/BTCUSD/minds/'}], 'position': 4}, {'title': '비트코인 시세 - 네이버 증권 - NAVER', 'link': 'https://m.stock.naver.com/crypto/UPBIT/BTC', 'snippet': 'No information is available for this page. · Learn why', 'position': 5}, {'title': '비트코인-BTC/BTC/KRW | 시세 확인 및 거래 | 거래소 | 빗썸 - Bithumb', 'link': 'https://www.bithumb.com/react/trade/order/BTC-KRW', 'snippet': 'BTC/KRW-비트코인-BTC/BTC/KRW 시세, 가격, 거래 정보 및 상세 내역을 빗썸에서 확인하고 거래해보세요.', 'sitelinks': [{'title': '비트코인', 'link': 'https://m.bithumb.com/react/trade/chart/BTC-KRW'}, {'title': '엑스알피 [리플]', 'link': 'https://www.bithumb.com/react/trade/order/XRP-KRW'}, {'title': '이더리움(ETH)', 'link': 'https://www.bithumb.com/react/trade/order/ETH-KRW'}, {'title': '테더-USDT/KRW', 'link': 'https://www.bithumb.com/react/trade/order/USDT-KRW'}], 'position': 6}, {'title': '실시간 암호화폐 및 코인 시세 - Investing.com - 인베스팅닷컴', 'link': 'https://kr.investing.com/crypto', 'snippet': '가장 거래가 활발한 암호화폐쌍 ; 비트코인. 21:29:41 |BTC/USD. 121,770.5. +2.85 ; 리플. 21:30:09 |XRP/USD. 2.9596. +4.97 ; 이더리움. 21:29:53 |ETH/USD. 3,043.94. + ...', 'sitelinks': [{'title': 'Bitcoin - 비트코인 가격 (BTC...', 'link': 'https://kr.investing.com/crypto/bitcoin'}, {'title': '모든 암호화페', 'link': 'https://kr.investing.com/crypto/currencies'}, {'title': '이더리움', 'link': 'https://kr.investing.com/crypto/ethereum'}, {'title': '뉴스', 'link': 'https://kr.investing.com/news/cryptocurrency-news'}], 'position': 7}, {'title': '비트코인(BTC) - 코인원(Coinone) 시세 확인 및 거래하기', 'link': 'https://coinone.co.kr/exchange', 'snippet': '거래소 · 고가: 0.0000 · 저가: 0.0000 · 전일가: 0.0000 · 거래량: 0.0000BTC · 거래대금: 0KRW.', 'sitelinks': [{'title': '공지사항', 'link': 'https://coinone.co.kr/info/notice'}, {'title': '엑스알피(리플)(XRP)', 'link': 'https://coinone.co.kr/exchange/trade/xrp/krw'}, {'title': '보유자산', 'link': 'https://coinone.co.kr/balance/investment'}, {'title': '도지코인(DOGE)', 'link': 'https://coinone.co.kr/exchange/trade/doge/krw'}], 'position': 8}, {'title': 'Bitcoin (BTC) 가격, 차트, 시가총액 | 코인마켓캡 - CoinMarketCap', 'link': 'https://coinmarketcap.com/ko/currencies/bitcoin/', 'snippet': '오늘의 Bitcoin 실시간 가격은 ₩168335458.64 KRW이며 24시간 거래량은 ₩149611784330593.19 KRW입니다.BTC 대 KRW 가격을 실시간으로 업데이트합니다.', 'sitelinks': [{'title': '과거 데이터 보기', 'link': 'https://coinmarketcap.com/ko/currencies/bitcoin/historical-data/'}, {'title': 'BTC USD', 'link': 'https://coinmarketcap.com/ko/currencies/bitcoin/btc/usd/'}, {'title': 'BTC BTC', 'link': 'https://coinmarketcap.com/ko/currencies/bitcoin/btc/btc/'}], 'rating': 4.4, 'ratingCount': 3, 'position': 9}]
# web_search
# {'context': '[\n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\ni\nt\nc\no\ni\nn\n \n-\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n \n(\nB\nT\nC\n \n\\\nu\nc\n5\n5\n4\n\\\nu\nd\n6\n3\n8\n\\\nu\nd\n6\n5\n4\n\\\nu\nd\n3\nd\n0\n)\n \n-\n \nI\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n \n-\n \n\\\nu\nc\n7\n7\n8\n\\\nu\nb\nc\na\n0\n\\\nu\nc\n2\na\n4\n\\\nu\nd\n3\n0\n5\n\\\nu\nb\n2\nf\n7\n\\\nu\nc\ne\nf\n4\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nb\ni\nt\nc\no\ni\nn\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n,\n \n\\\nu\nc\n0\na\nc\n\\\nu\nc\n0\nc\n1\n \n\\\nu\nc\nc\na\nb\n \n1\n2\n\\\nu\nb\n9\nc\nc\n\\\nu\nb\n2\ne\nc\n\\\nu\nb\n7\ne\nc\n \n\\\nu\nb\n3\nc\nc\n\\\nu\nd\n3\n0\nc\n\\\nu\n2\n0\n2\n6\n \n\\\nu\nc\n7\n7\n4\n\\\nu\nb\n3\n5\n4\n\\\nu\nb\n9\na\nc\n\\\nu\nc\n6\nc\n0\n\\\nu\nb\n3\nc\n4\n \n3\n0\n0\n0\n\\\nu\nb\n2\ne\nc\n\\\nu\nb\n7\ne\nc\n \n\\\nu\na\ne\n3\n0\n\\\nu\nb\n8\n5\nd\n.\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n\\\nu\nc\n7\n7\n4\n \n1\n2\n\\\nu\nb\n9\nc\nc\n\\\nu\nb\n2\ne\nc\n\\\nu\nb\n7\ne\nc\n(\n\\\nu\nc\n5\n7\nd\n \n1\n\\\nu\nc\n5\nb\n5\n6\n5\n4\n5\n\\\nu\nb\n9\nc\nc\n\\\nu\nc\n6\nd\n0\n)\n\\\nu\nb\n3\n0\n0\n\\\nu\nb\n9\n7\nc\n \n\\\nu\nb\n3\nc\nc\n\\\nu\nd\n3\n0\nc\n\\\nu\nd\n5\n8\n8\n\\\nu\nb\n2\ne\n4\n.\n \n1\n4\n\\\nu\nc\n7\n7\nc\n \n\\\nu\nc\n6\n2\n4\n\\\nu\nd\n6\nc\n4\n \n1\n\\\nu\nc\n2\nd\nc\n \n\\\nu\nd\n6\n0\n4\n\\\nu\nc\n7\na\nc\n \n\\\nu\na\ne\n0\n0\n\\\nu\nb\n8\n5\nc\n\\\nu\nb\nc\n8\nc\n \n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nd\n6\n6\n9\n \n\\\nu\nc\n9\n1\n1\n\\\nu\na\nc\nc\n4\n\\\nu\nc\n0\na\nc\n\\\nu\nc\n7\n7\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\n7\n7\n8\n.\n"\n,\n \n"\ns\ni\nt\ne\nl\ni\nn\nk\ns\n"\n:\n \n[\n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nT\nC\n \nU\nS\nD\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nb\ni\nt\nc\no\ni\nn\n/\nb\nt\nc\n-\nu\ns\nd\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nc\nc\n2\n8\n\\\nu\nd\n2\nb\n8\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nb\ni\nt\nc\no\ni\nn\n/\nc\nh\na\nr\nt\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nd\n3\ne\nc\n\\\nu\nb\n7\nf\nc\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nb\ni\nt\nc\no\ni\nn\n/\nc\nh\na\nt\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nT\nC\n \nK\nR\nW\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nb\ni\nt\nc\no\ni\nn\n/\nb\nt\nc\n-\nk\nr\nw\n"\n}\n]\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n1\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nc\n5\nc\n5\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n \nu\np\nb\ni\nt\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nc\n1\n8\nc\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nu\np\nb\ni\nt\n.\nc\no\nm\n/\ne\nx\nc\nh\na\nn\ng\ne\n/\nC\nR\nI\nX\n.\nU\nP\nB\nI\nT\n.\nK\nR\nW\n-\nB\nT\nC\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nc\n1\n3\n8\n,\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n,\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n,\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nc\n8\n1\n5\n\\\nu\nb\nc\nf\n4\n \n\\\nu\nb\n4\nf\n1\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n\\\nu\nc\n7\n5\n8\n \n\\\nu\nb\na\na\n8\n\\\nu\nb\n4\ne\n0\n \n\\\nu\na\nc\n8\n3\n,\n \n\\\nu\na\nc\n0\n0\n\\\nu\nc\n7\na\n5\n \n\\\nu\nc\n2\ne\n0\n\\\nu\nb\n8\nb\n0\n\\\nu\nb\nc\n1\nb\n\\\nu\nb\n2\n9\n4\n \n\\\nu\nb\n5\n1\n4\n\\\nu\nc\n9\nc\n0\n\\\nu\nd\n1\n3\n8\n \n\\\nu\nc\n7\n9\n0\n\\\nu\nc\n0\nb\n0\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nc\n1\n8\nc\n \n\\\nu\nc\n5\nc\n5\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\n5\nd\n0\n\\\nu\nc\n1\n1\nc\n \n\\\nu\nd\n6\n5\n5\n\\\nu\nc\n7\n7\n8\n\\\nu\nd\n5\n7\n4\n\\\nu\nb\nc\nf\n4\n\\\nu\nc\n1\n3\n8\n\\\nu\nc\n6\n9\n4\n.\n"\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n2\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nc\n1\n3\n8\n:\n \nB\nT\nC\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n \n\\\nu\nc\nc\n2\n8\n\\\nu\nd\n2\nb\n8\n,\n \n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n0\n\\\nu\nc\nd\n1\nd\n\\\nu\nc\n5\n6\n1\n \n&\n \n\\\nu\nc\n6\n2\n4\n\\\nu\nb\n2\n9\n8\n\\\nu\nc\n7\n5\n8\n \n\\\nu\nb\n2\n7\n4\n\\\nu\nc\n2\na\n4\n \n|\n \nC\no\ni\nn\nG\ne\nc\nk\no\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nw\nw\nw\n.\nc\no\ni\nn\ng\ne\nc\nk\no\n.\nc\no\nm\n/\nk\no\n/\n%\nE\nC\n%\nB\nD\n%\n9\n4\n%\nE\nC\n%\n9\nD\n%\nB\n8\n/\n%\nE\nB\n%\nB\n9\n%\n8\n4\n%\nE\nD\n%\n8\nA\n%\nB\n8\n%\nE\nC\n%\nB\nD\n%\n9\n4\n%\nE\nC\n%\n9\nD\n%\nB\n8\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n(\nB\nT\nC\n)\n\\\nu\nc\n7\n5\n8\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n\\\nu\nc\n7\n4\n0\n \n\\\nu\nd\n6\n0\n4\n\\\nu\nc\n7\na\nc\n \nU\nS\n$\n1\n2\n2\n,\n2\n9\n7\n,\n \n2\n4\n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n4\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nb\n7\nc\n9\n\\\nu\nc\n7\n4\n0\n \nU\nS\n$\n4\n7\n,\n5\n1\n2\n,\n1\n9\n0\n,\n2\n5\n1\n\\\nu\nc\n7\n8\n5\n\\\nu\nb\n2\nc\n8\n\\\nu\nb\n2\ne\n4\n.\n \n\\\nu\nc\nd\n5\nc\n\\\nu\na\nd\nf\nc\n \n2\n4\n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n4\n \n\\\nu\nb\n0\nb\n4\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n\\\nu\nc\n7\n4\n0\n \n3\n.\n7\n1\n%\n \n\\\nu\nc\n0\nc\n1\n\\\nu\nc\n2\nb\n9\n\\\nu\nd\n5\n8\n8\n\\\nu\nc\n7\n3\nc\n\\\nu\nb\na\n7\n0\n,\n \n\\\nu\nc\n9\nc\n0\n\\\nu\nb\n0\n9\nc\n \n7\n\\\nu\nc\n7\n7\nc\n \n\\\nu\nb\n0\nb\n4\n\\\nu\nc\n5\nd\n0\n\\\nu\nb\n2\n9\n4\n \n1\n2\n.\n4\n0\n%\n \n\\\nu\nc\n0\nc\n1\n\\\nu\nc\n2\nb\n9\n\\\nu\nd\n5\n8\n8\n\\\nu\nc\n2\nb\n5\n\\\nu\nb\n2\nc\n8\n\\\nu\nb\n2\ne\n4\n.\n"\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n3\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nT\nC\n \nU\nS\nD\n \n\\\nu\n2\n0\n1\n4\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n \n\\\nu\nb\nc\n0\nf\n \n\\\nu\nc\nc\n2\n8\n\\\nu\nd\n2\nb\n8\n \n-\n \n\\\nu\nd\n2\nb\n8\n\\\nu\nb\n8\n0\n8\n\\\nu\nc\n7\n7\n4\n\\\nu\nb\n5\n2\n9\n\\\nu\nb\nd\nf\n0\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\nt\nr\na\nd\ni\nn\ng\nv\ni\ne\nw\n.\nc\no\nm\n/\ns\ny\nm\nb\no\nl\ns\n/\nB\nT\nC\nU\nS\nD\n/\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n(\nB\nT\nC\n)\n \n\\\nu\nc\n7\n5\n8\n \n\\\nu\nd\n6\n0\n4\n\\\nu\nc\n7\na\nc\n \n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n0\n\\\nu\nc\nd\n1\nd\n\\\nu\nc\n5\n6\n1\n\\\nu\nc\n7\n4\n0\n \n\\\nu\n2\n0\n2\na\n2\n.\n4\n2\n \nT\n\\\nu\n2\n0\n2\nc\n \nU\nS\nD\n \n\\\nu\nc\n7\n8\n5\n\\\nu\nb\n2\nc\n8\n\\\nu\nb\n2\ne\n4\n.\n \n\\\nu\nc\n7\n7\n4\n \n\\\nu\nc\n2\n2\nb\n\\\nu\nc\n7\n9\n0\n\\\nu\nb\n9\n7\nc\n \n\\\nu\nb\n9\ne\n5\n\\\nu\nb\n7\n7\nd\n\\\nu\nc\n5\nd\n0\n\\\nu\nc\n1\n1\nc\n \n\\\nu\nd\n6\n5\n5\n\\\nu\nc\n7\n7\n8\n\\\nu\nd\n5\n5\n8\n\\\nu\nb\n8\n2\n4\n\\\nu\nb\na\n7\n4\n \n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n0\n\\\nu\nc\nd\n1\nd\n\\\nu\nc\n5\n6\n1\n\\\nu\nb\nc\nc\n4\n\\\nu\nb\n8\n5\nc\n \n\\\nu\nc\n2\n1\nc\n\\\nu\nc\n7\n0\n4\n\\\nu\na\nc\n0\n0\n \n\\\nu\nb\n9\ne\n4\n\\\nu\na\nc\na\n8\n\\\nu\nc\n9\nc\n4\n \n\\\nu\nd\n0\n6\nc\n\\\nu\nb\n9\nb\nd\n\\\nu\nd\n1\na\n0\n \n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nb\n9\na\nc\n\\\nu\nc\n2\na\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nb\n9\n7\nc\n \n\\\nu\nc\nc\nb\n4\n\\\nu\nd\n0\n6\nc\n\\\nu\nd\n5\n5\n8\n\\\nu\na\nc\n7\n0\n\\\nu\nb\n0\n9\n8\n \n\\\nu\nd\n0\n6\nc\n\\\nu\nb\n9\nb\nd\n\\\nu\nd\n1\na\n0\n \n\\\nu\nb\n9\nc\n8\n\\\nu\nc\nf\n1\n3\n \n\\\nu\nc\ne\na\n1\n \n.\n.\n.\n"\n,\n \n"\ns\ni\nt\ne\nl\ni\nn\nk\ns\n"\n:\n \n[\n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nt\nc\nu\ns\nd\n \n\\\nu\nb\n2\n7\n4\n\\\nu\nc\n2\na\n4\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\nt\nr\na\nd\ni\nn\ng\nv\ni\ne\nw\n.\nc\no\nm\n/\ns\ny\nm\nb\no\nl\ns\n/\nB\nT\nC\nU\nS\nD\n/\nn\ne\nw\ns\n/\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nT\nC\nU\nS\nD\n \nP\ne\nr\np\ne\nt\nu\na\nl\n \nC\no\nn\nt\nr\na\nc\nt\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\nt\nr\na\nd\ni\nn\ng\nv\ni\ne\nw\n.\nc\no\nm\n/\ns\ny\nm\nb\no\nl\ns\n/\nB\nT\nC\nU\nS\nD\n.\nP\n/\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nt\nc\nu\ns\nd\n \n\\\nu\nd\n2\nb\n8\n\\\nu\nb\n8\n0\n8\n\\\nu\nc\n7\n7\n4\n\\\nu\nb\n5\n2\n9\n \n\\\nu\nc\n5\n4\n4\n\\\nu\nc\n7\n7\n4\n\\\nu\nb\n5\n1\n4\n\\\nu\nc\n5\nb\n4\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\nt\nr\na\nd\ni\nn\ng\nv\ni\ne\nw\n.\nc\no\nm\n/\ns\ny\nm\nb\no\nl\ns\n/\nB\nT\nC\nU\nS\nD\n/\ni\nd\ne\na\ns\n/\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nd\n1\na\n0\n\\\nu\nb\n8\n6\n0\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\nt\nr\na\nd\ni\nn\ng\nv\ni\ne\nw\n.\nc\no\nm\n/\ns\ny\nm\nb\no\nl\ns\n/\nB\nT\nC\nU\nS\nD\n/\nm\ni\nn\nd\ns\n/\n"\n}\n]\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n4\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nc\n1\n3\n8\n \n-\n \n\\\nu\nb\n1\n2\n4\n\\\nu\nc\n7\n7\n4\n\\\nu\nb\nc\n8\n4\n \n\\\nu\nc\n9\n9\nd\n\\\nu\na\nd\n8\nc\n \n-\n \nN\nA\nV\nE\nR\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nm\n.\ns\nt\no\nc\nk\n.\nn\na\nv\ne\nr\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nU\nP\nB\nI\nT\n/\nB\nT\nC\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\nN\no\n \ni\nn\nf\no\nr\nm\na\nt\ni\no\nn\n \ni\ns\n \na\nv\na\ni\nl\na\nb\nl\ne\n \nf\no\nr\n \nt\nh\ni\ns\n \np\na\ng\ne\n.\n \n\\\nu\n0\n0\nb\n7\n \nL\ne\na\nr\nn\n \nw\nh\ny\n"\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n5\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n-\nB\nT\nC\n/\nB\nT\nC\n/\nK\nR\nW\n \n|\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nc\n1\n3\n8\n \n\\\nu\nd\n6\n5\n5\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nb\nc\n0\nf\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n \n|\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nc\n1\n8\nc\n \n|\n \n\\\nu\nb\ne\n5\n7\n\\\nu\nc\n3\n7\n8\n \n-\n \nB\ni\nt\nh\nu\nm\nb\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nw\nw\nw\n.\nb\ni\nt\nh\nu\nm\nb\n.\nc\no\nm\n/\nr\ne\na\nc\nt\n/\nt\nr\na\nd\ne\n/\no\nr\nd\ne\nr\n/\nB\nT\nC\n-\nK\nR\nW\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\nB\nT\nC\n/\nK\nR\nW\n-\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n-\nB\nT\nC\n/\nB\nT\nC\n/\nK\nR\nW\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nc\n1\n3\n8\n,\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n,\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n \n\\\nu\nc\n8\n1\n5\n\\\nu\nb\nc\nf\n4\n \n\\\nu\nb\nc\n0\nf\n \n\\\nu\nc\n0\nc\n1\n\\\nu\nc\n1\n3\n8\n \n\\\nu\nb\n0\nb\n4\n\\\nu\nc\n5\ne\nd\n\\\nu\nc\n7\n4\n4\n \n\\\nu\nb\ne\n5\n7\n\\\nu\nc\n3\n7\n8\n\\\nu\nc\n5\nd\n0\n\\\nu\nc\n1\n1\nc\n \n\\\nu\nd\n6\n5\n5\n\\\nu\nc\n7\n7\n8\n\\\nu\nd\n5\n5\n8\n\\\nu\na\nc\ne\n0\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nd\n5\n7\n4\n\\\nu\nb\nc\nf\n4\n\\\nu\nc\n1\n3\n8\n\\\nu\nc\n6\n9\n4\n.\n"\n,\n \n"\ns\ni\nt\ne\nl\ni\nn\nk\ns\n"\n:\n \n[\n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nm\n.\nb\ni\nt\nh\nu\nm\nb\n.\nc\no\nm\n/\nr\ne\na\nc\nt\n/\nt\nr\na\nd\ne\n/\nc\nh\na\nr\nt\n/\nB\nT\nC\n-\nK\nR\nW\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nc\n5\nd\n1\n\\\nu\nc\n2\na\n4\n\\\nu\nc\n5\n4\nc\n\\\nu\nd\n5\n3\nc\n \n[\n\\\nu\nb\n9\na\nc\n\\\nu\nd\n5\n0\nc\n]\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nw\nw\nw\n.\nb\ni\nt\nh\nu\nm\nb\n.\nc\no\nm\n/\nr\ne\na\nc\nt\n/\nt\nr\na\nd\ne\n/\no\nr\nd\ne\nr\n/\nX\nR\nP\n-\nK\nR\nW\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nc\n7\n7\n4\n\\\nu\nb\n3\n5\n4\n\\\nu\nb\n9\na\nc\n\\\nu\nc\n6\nc\n0\n(\nE\nT\nH\n)\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nw\nw\nw\n.\nb\ni\nt\nh\nu\nm\nb\n.\nc\no\nm\n/\nr\ne\na\nc\nt\n/\nt\nr\na\nd\ne\n/\no\nr\nd\ne\nr\n/\nE\nT\nH\n-\nK\nR\nW\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nd\n1\n4\nc\n\\\nu\nb\n3\n5\n4\n-\nU\nS\nD\nT\n/\nK\nR\nW\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nw\nw\nw\n.\nb\ni\nt\nh\nu\nm\nb\n.\nc\no\nm\n/\nr\ne\na\nc\nt\n/\nt\nr\na\nd\ne\n/\no\nr\nd\ne\nr\n/\nU\nS\nD\nT\n-\nK\nR\nW\n"\n}\n]\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n6\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nc\n2\ne\n4\n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n4\n \n\\\nu\nc\n5\n5\n4\n\\\nu\nd\n6\n3\n8\n\\\nu\nd\n6\n5\n4\n\\\nu\nd\n3\nd\n0\n \n\\\nu\nb\nc\n0\nf\n \n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nc\n1\n3\n8\n \n-\n \nI\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n \n-\n \n\\\nu\nc\n7\n7\n8\n\\\nu\nb\nc\na\n0\n\\\nu\nc\n2\na\n4\n\\\nu\nd\n3\n0\n5\n\\\nu\nb\n2\nf\n7\n\\\nu\nc\ne\nf\n4\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\n\\\nu\na\nc\n0\n0\n\\\nu\nc\n7\na\n5\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\na\nc\n0\n0\n \n\\\nu\nd\n6\n5\nc\n\\\nu\nb\nc\n1\nc\n\\\nu\nd\n5\n5\nc\n \n\\\nu\nc\n5\n5\n4\n\\\nu\nd\n6\n3\n8\n\\\nu\nd\n6\n5\n4\n\\\nu\nd\n3\nd\n0\n\\\nu\nc\n3\n0\nd\n \n;\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n.\n \n2\n1\n:\n2\n9\n:\n4\n1\n \n|\nB\nT\nC\n/\nU\nS\nD\n.\n \n1\n2\n1\n,\n7\n7\n0\n.\n5\n.\n \n+\n2\n.\n8\n5\n \n;\n \n\\\nu\nb\n9\na\nc\n\\\nu\nd\n5\n0\nc\n.\n \n2\n1\n:\n3\n0\n:\n0\n9\n \n|\nX\nR\nP\n/\nU\nS\nD\n.\n \n2\n.\n9\n5\n9\n6\n.\n \n+\n4\n.\n9\n7\n \n;\n \n\\\nu\nc\n7\n7\n4\n\\\nu\nb\n3\n5\n4\n\\\nu\nb\n9\na\nc\n\\\nu\nc\n6\nc\n0\n.\n \n2\n1\n:\n2\n9\n:\n5\n3\n \n|\nE\nT\nH\n/\nU\nS\nD\n.\n \n3\n,\n0\n4\n3\n.\n9\n4\n.\n \n+\n \n.\n.\n.\n"\n,\n \n"\ns\ni\nt\ne\nl\ni\nn\nk\ns\n"\n:\n \n[\n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\ni\nt\nc\no\ni\nn\n \n-\n \n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n \n(\nB\nT\nC\n.\n.\n.\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nb\ni\nt\nc\no\ni\nn\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\na\na\n8\n\\\nu\nb\n4\ne\n0\n \n\\\nu\nc\n5\n5\n4\n\\\nu\nd\n6\n3\n8\n\\\nu\nd\n6\n5\n4\n\\\nu\nd\n3\n9\n8\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\nc\nu\nr\nr\ne\nn\nc\ni\ne\ns\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nc\n7\n7\n4\n\\\nu\nb\n3\n5\n4\n\\\nu\nb\n9\na\nc\n\\\nu\nc\n6\nc\n0\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nc\nr\ny\np\nt\no\n/\ne\nt\nh\ne\nr\ne\nu\nm\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\n2\n7\n4\n\\\nu\nc\n2\na\n4\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nk\nr\n.\ni\nn\nv\ne\ns\nt\ni\nn\ng\n.\nc\no\nm\n/\nn\ne\nw\ns\n/\nc\nr\ny\np\nt\no\nc\nu\nr\nr\ne\nn\nc\ny\n-\nn\ne\nw\ns\n"\n}\n]\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n7\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\ne\n4\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n(\nB\nT\nC\n)\n \n-\n \n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n\\\nu\nc\n6\nd\n0\n(\nC\no\ni\nn\no\nn\ne\n)\n \n\\\nu\nc\n2\nd\nc\n\\\nu\nc\n1\n3\n8\n \n\\\nu\nd\n6\n5\n5\n\\\nu\nc\n7\n7\n8\n \n\\\nu\nb\nc\n0\nf\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nd\n5\n5\n8\n\\\nu\na\ne\n3\n0\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\no\nn\ne\n.\nc\no\n.\nk\nr\n/\ne\nx\nc\nh\na\nn\ng\ne\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nc\n1\n8\nc\n \n\\\nu\n0\n0\nb\n7\n \n\\\nu\na\nc\ne\n0\n\\\nu\na\nc\n0\n0\n:\n \n0\n.\n0\n0\n0\n0\n \n\\\nu\n0\n0\nb\n7\n \n\\\nu\nc\n8\n0\n0\n\\\nu\na\nc\n0\n0\n:\n \n0\n.\n0\n0\n0\n0\n \n\\\nu\n0\n0\nb\n7\n \n\\\nu\nc\n8\n0\n4\n\\\nu\nc\n7\n7\nc\n\\\nu\na\nc\n0\n0\n:\n \n0\n.\n0\n0\n0\n0\n \n\\\nu\n0\n0\nb\n7\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nb\n7\nc\n9\n:\n \n0\n.\n0\n0\n0\n0\nB\nT\nC\n \n\\\nu\n0\n0\nb\n7\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nb\n3\n0\n0\n\\\nu\na\ne\n0\n8\n:\n \n0\nK\nR\nW\n.\n"\n,\n \n"\ns\ni\nt\ne\nl\ni\nn\nk\ns\n"\n:\n \n[\n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\na\nc\nf\n5\n\\\nu\nc\n9\nc\n0\n\\\nu\nc\n0\na\nc\n\\\nu\nd\n5\n6\nd\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\no\nn\ne\n.\nc\no\n.\nk\nr\n/\ni\nn\nf\no\n/\nn\no\nt\ni\nc\ne\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nc\n5\nd\n1\n\\\nu\nc\n2\na\n4\n\\\nu\nc\n5\n4\nc\n\\\nu\nd\n5\n3\nc\n(\n\\\nu\nb\n9\na\nc\n\\\nu\nd\n5\n0\nc\n)\n(\nX\nR\nP\n)\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\no\nn\ne\n.\nc\no\n.\nk\nr\n/\ne\nx\nc\nh\na\nn\ng\ne\n/\nt\nr\na\nd\ne\n/\nx\nr\np\n/\nk\nr\nw\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\nc\nf\n4\n\\\nu\nc\n7\n2\n0\n\\\nu\nc\n7\n9\n0\n\\\nu\nc\n0\nb\n0\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\no\nn\ne\n.\nc\no\n.\nk\nr\n/\nb\na\nl\na\nn\nc\ne\n/\ni\nn\nv\ne\ns\nt\nm\ne\nn\nt\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\nb\n3\nc\n4\n\\\nu\nc\n9\nc\n0\n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n(\nD\nO\nG\nE\n)\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\no\nn\ne\n.\nc\no\n.\nk\nr\n/\ne\nx\nc\nh\na\nn\ng\ne\n/\nt\nr\na\nd\ne\n/\nd\no\ng\ne\n/\nk\nr\nw\n"\n}\n]\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n8\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\ni\nt\nc\no\ni\nn\n \n(\nB\nT\nC\n)\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n,\n \n\\\nu\nc\nc\n2\n8\n\\\nu\nd\n2\nb\n8\n,\n \n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n0\n\\\nu\nc\nd\n1\nd\n\\\nu\nc\n5\n6\n1\n \n|\n \n\\\nu\nc\nf\n5\n4\n\\\nu\nc\n7\n7\n8\n\\\nu\nb\n9\nc\n8\n\\\nu\nc\nf\n1\n3\n\\\nu\nc\ne\na\n1\n \n-\n \nC\no\ni\nn\nM\na\nr\nk\ne\nt\nC\na\np\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\nm\na\nr\nk\ne\nt\nc\na\np\n.\nc\no\nm\n/\nk\no\n/\nc\nu\nr\nr\ne\nn\nc\ni\ne\ns\n/\nb\ni\nt\nc\no\ni\nn\n/\n"\n,\n \n"\ns\nn\ni\np\np\ne\nt\n"\n:\n \n"\n\\\nu\nc\n6\n2\n4\n\\\nu\nb\n2\n9\n8\n\\\nu\nc\n7\n5\n8\n \nB\ni\nt\nc\no\ni\nn\n \n\\\nu\nc\n2\ne\n4\n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n4\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n\\\nu\nc\n7\n4\n0\n \n\\\nu\n2\n0\na\n9\n1\n6\n8\n3\n3\n5\n4\n5\n8\n.\n6\n4\n \nK\nR\nW\n\\\nu\nc\n7\n7\n4\n\\\nu\nb\na\n7\n0\n \n2\n4\n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n4\n \n\\\nu\na\nc\n7\n0\n\\\nu\nb\n7\n9\n8\n\\\nu\nb\n7\nc\n9\n\\\nu\nc\n7\n4\n0\n \n\\\nu\n2\n0\na\n9\n1\n4\n9\n6\n1\n1\n7\n8\n4\n3\n3\n0\n5\n9\n3\n.\n1\n9\n \nK\nR\nW\n\\\nu\nc\n7\n8\n5\n\\\nu\nb\n2\nc\n8\n\\\nu\nb\n2\ne\n4\n.\nB\nT\nC\n \n\\\nu\nb\n3\n0\n0\n \nK\nR\nW\n \n\\\nu\na\nc\n0\n0\n\\\nu\na\nc\na\n9\n\\\nu\nc\n7\n4\n4\n \n\\\nu\nc\n2\ne\n4\n\\\nu\nc\n2\nd\nc\n\\\nu\na\nc\n0\n4\n\\\nu\nc\n7\n3\nc\n\\\nu\nb\n8\n5\nc\n \n\\\nu\nc\n5\nc\n5\n\\\nu\nb\n3\n7\n0\n\\\nu\nc\n7\n7\n4\n\\\nu\nd\n2\nb\n8\n\\\nu\nd\n5\n6\n9\n\\\nu\nb\n2\nc\n8\n\\\nu\nb\n2\ne\n4\n.\n"\n,\n \n"\ns\ni\nt\ne\nl\ni\nn\nk\ns\n"\n:\n \n[\n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\n\\\nu\na\nc\nf\nc\n\\\nu\na\nc\n7\n0\n \n\\\nu\nb\n3\n7\n0\n\\\nu\nc\n7\n7\n4\n\\\nu\nd\n1\n3\n0\n \n\\\nu\nb\nc\nf\n4\n\\\nu\na\ne\n3\n0\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\nm\na\nr\nk\ne\nt\nc\na\np\n.\nc\no\nm\n/\nk\no\n/\nc\nu\nr\nr\ne\nn\nc\ni\ne\ns\n/\nb\ni\nt\nc\no\ni\nn\n/\nh\ni\ns\nt\no\nr\ni\nc\na\nl\n-\nd\na\nt\na\n/\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nT\nC\n \nU\nS\nD\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\nm\na\nr\nk\ne\nt\nc\na\np\n.\nc\no\nm\n/\nk\no\n/\nc\nu\nr\nr\ne\nn\nc\ni\ne\ns\n/\nb\ni\nt\nc\no\ni\nn\n/\nb\nt\nc\n/\nu\ns\nd\n/\n"\n}\n,\n \n{\n"\nt\ni\nt\nl\ne\n"\n:\n \n"\nB\nT\nC\n \nB\nT\nC\n"\n,\n \n"\nl\ni\nn\nk\n"\n:\n \n"\nh\nt\nt\np\ns\n:\n/\n/\nc\no\ni\nn\nm\na\nr\nk\ne\nt\nc\na\np\n.\nc\no\nm\n/\nk\no\n/\nc\nu\nr\nr\ne\nn\nc\ni\ne\ns\n/\nb\ni\nt\nc\no\ni\nn\n/\nb\nt\nc\n/\nb\nt\nc\n/\n"\n}\n]\n,\n \n"\nr\na\nt\ni\nn\ng\n"\n:\n \n4\n.\n4\n,\n \n"\nr\na\nt\ni\nn\ng\nC\no\nu\nn\nt\n"\n:\n \n3\n,\n \n"\np\no\ns\ni\nt\ni\no\nn\n"\n:\n \n9\n}\n]'}
# llm_answer
# {'answer': '비트코인(Bitcoin, BTC)의 현재 가격은 US$47,512.190입니다.\n\n**출처:**\n* https://www.coingecko.com/ko/%EC%BD%94%EC%9D%B8/%EB%B9%84%ED%8A%B8%EC%BD%94%EC%9D%B8', 'messages': [('user', '비트코인 현재 가격'), ('assistant', '비트코인(Bitcoin, BTC)의 현재 가격은 US$47,512.190입니다.\n\n**출처:**\n* https://www.coingecko.com/ko/%EC%BD%94%EC%9D%B8/%EB%B9%84%ED%8A%B8%EC%BD%94%EC%9D%B8')]}
```  


## Query Rewriter
[이전 포스팅]({{site.baseurl}}{% link _posts/langgraph/2026-01-20-langgraph-practice-graph-rag-structure.md %})
과 앞서 구현한 `web_search` 에 추가적으로 `query rewriter` 를 추가하여, 
`retrieval` 를 수행하기 전에 `query` 를 재작성하는 구조를 구현해 본다. 
이를 통해 사용자의 질의가 `retrieval` 에 적합한 형태로 변환되어, 
더 나은 검색 결과를 얻을 수 있도록 한다.  

물론 `query rewriter` 는 단순히 `retrieval` 에 적합한 형태로 변환하는 것 뿐만 아니라, 
사용자의 의도를 파악하고, 검색 결과의 품질을 향상시키는 데에도 중요한 역할을 한다. 
또한 최초 수행 시점에도 사용할 수 있지만, 관련성 검증이 실패한 경우에도 좀 더 관련성이 높은 결과가 나올 수 있도록 쿼리를 수정한다는 등 
다양한 측면에서 활용될 수 있다.  

쿼리 재작성에 대한 내용을 `QueryRewrite` 클래스로 구현하면 알애ㅘ 같다.  

```python
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser

class QueryRewrite:
  def __init__(self, llm):
    self.llm = llm

  def __call__(self, params: dict):
    prompt = PromptTemplate(
        template="""
        Reformulate the given question to enhance ist effectiveness for vectorstore retrieval.

        - Analyze the initial question to identity areas for improvment such as specificity, clarity, and relevance.
        - Consider the context and potential keywords that would optimize retrieval.
        - Maintain the intent of the original question while enhancing its structure and vocabulary.

        # Steps
        1. **Understand the Original Question**: Identify the core intent and any keyword.
        2. **Enhance Clarity**: Simplify language and ensure the question is direct and to the point.
        3. **Optimize of Retrieval**: Add or rearrange keywords for better alignment with vectorstore indexing.
        4. **Review**: Ensure the improved question accurately reflects the original intent and is free of ambiguity

        # Output Format
        - Provide a single, improved question.
        - Do not include any introductory for explanatory text; only the reformulate question.

        # Examples
        **Input**:
        "What are the benefits of using renewable energy sources over fossil fuels?"
        *Output**:
        "How do renewable energy sources compare to fossil fuels in terms of benefits?"
        **Input**:
        "How does climate change impact polar bear populations?"
        **Output**:
        "What effects does climate change have on polar bear populations?"

        # Notes
        - Ensure the improved question is concise and contextually relevant.
        - Avoid altering the fundamental intent or meaning of the original question.

        [REMEMBER] Re-written question should be in the same langugage as the original question.

        # Ehere is the original question that needs to be rewritten:
        {question}
        """,
        input_variables=["question"]
    )
    chain = prompt | self.llm | StrOutputParser()
    return chain.invoke(params)
```  

구현한 `QueryRewrite` 클래스가 어떤 결과를 보이는지 테스트해 보면아래와 같다.  

```python
from langchain_google_genai import ChatGoogleGenerativeAI
import os

os.environ["GOOGLE_API_KEY"] = "api key"
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash")

rewrite = QueryRewrite(llm)
rewrite({'question': '사람이 배가고프면 어떻게돼?' })
# 배고픔이 사람에게 미치는 영향은 무엇인가?
```  

이제 구현된 클래스를 사용해 `query_rewrite` 노드 함수를 정의한다.  

```python
# Query Rewrite 함수 노드 정의
def query_rewrite(state: GraphState) -> GraphState:
  latest_question = state['question']
  question_rewritten = QueryRewrite(llm)({'question': latest_question })
  return GraphState(question=question_rewritten)
```  
