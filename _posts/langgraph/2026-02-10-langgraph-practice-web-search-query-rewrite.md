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
