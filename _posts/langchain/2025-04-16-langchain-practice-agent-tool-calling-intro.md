--- 
layout: single
classes: wide
title: "[LangChain] LangChain Agent Tool Calling Intro"
header:
  overlay_image: /img/langchain-bg-2.png
excerpt: 'LangChain 에서 AI 시스템이 더 자율적이고 목표 지향적으로 상호작용을 하며 행동을 취하는 지능형 시스템인 Agent 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - Agent
    - Tool Calling
    - Function Calling
toc: true
use_math: true
---  

## LangChain Agent
`LangChain Agent` 는 `LLM` 기반 애플리케이션에서 중요한 역할을 하는 컴포넌트로, 
`AI` 시스템이 더욱 자율적이고 목표 지향적으로 작업을 수행할 수 있게 해주는 컴포넌트이다. 
`Agent` 는 주어진 목표를 달성하기 위해 상호작용하며 의사 결정을 내리고 행동을 취하는 지능형 시스템이다.  

`LLM` 이 다양한 `Tool` 을 활용해 동적인 의사결정을 할 수 있도록 구성된 컴포넌트를 의미한다.
사용자의 입력에 따라 적절한 도구를 선택하고 실행하여 최적의 답변을 생성하는 방식으로 동작하게 된다.  

### Agent Type
`Agent` 는 아래와 같은 다양한 유형을 목적에 맞게 사용할 수 있다. 

- `ZERO_SHOT_REACT_DESCRIPTION` : 사전 학습 없이 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 간단한 질문에 대해 빠르게 답변해야 하는 경우 사용하기 좋다. 
- `REACT_DOCSTORE` : 문서 저장소에 접근하여 관련 정보를 검색하고 질문에 답하는 에이전트이다. 대량의 문서나 데이터베이스에서 종보를 검색해야 하는 경우 사용하기 좋다. 
- `SELF_ASK_WITH_SEARCH` : 복잡한 질문을 더 간단한 질문으로 나누고, 검색 도구를 사용해 답을 찾는 에이전트이다. 복잡한 문제를 단계별로 해결해야 하는 경우 사용하기 좋다. 
- `CONVERSATIONAL_REACT_DESCRIPTION` : 대화형으로 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 사용자와의 대화에서 자연스럽게 정보를 제공해야 하는 경우 사용하기 좋다. 
- `CHAT_ZERO_SHOT_REACT_DESCRIPTION` : 대화형 모델에 최적화된 사전 학습 없이 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 대화형 AI 챗봇 구현에 사용하기 좋다. 
- `CHAT_CONVERSATIONAL_REACT_DESCRIPTION` : 대화형으로 주어진 설명을 바탕으로 추론 단계를 거쳐 행동하는 에이전트이다. 사용자와의 대화에서 자연스럽게 정보를 제공해야 하는 경우 사용하기 좋다. 
- `STRUCTURED_CHAT_ZERO_SHOT_REACT_DESCRIPTION` : 여러 입력을 가진 도구를 호출할 수 있는 대화형 모델에 최적화된 에이전트이다. 복잡한 입력을 처리해야 하는 대화형 AI 시스템에 사용하기 좋다. 
- `OPENAI_FUNCTIONS` : `OpenAI` 함수 사용에 최적화된 에이전트이다. `OpenAI` 의 다양한 기능을 화룡ㅇ해야 하는 경우 사용하기 좋다. 
- `OPENAI_MULTI_FUNCTIONS` : 여러 `OpenAI` 함수를 사용하는 에이전트이다. 여러 `OpenAI` 기능을 동시에 활용해야 하는 경우 사용하기 좋다. 


## LangChain Tool calling
`LangChain Tool calling` 은 `LLM` 이 특정 기능을 실행하도록 설게된 도구를 호출하는 방식을 의미한다. 
사용자가 입력한 질문에 따라 `LLM` 이 적절한 `Tool` 을 선택하고 실행해 최적의 결과를 제공한다. 
`LLM` 이 단순히 답변을 생성하는 것이 아니라 `API` 호출, 데이터베이스, 계산기 등 활용 가능한 다양한 `Tool` 을 직접 실행해 결과를 생성하는 것을 의미한다.  

`Tool Calling` 을 활용하면 아래와 같은 장점을 얻을 수 있다. 

- 기능 확장 : `LLM` 의 능력을 크게 확장할 수 있다. 모델이 자체적으로 가지고 있지 않은 정보나 기능에 접근하도록 할 수 있다. 
- 실시간 정보 : 웹 검색 등을 통해 모델이 학습하지 못한 최신 정보를 얻을 수 있어, 답변의 최신성을 제공할 수 있다. 
- 적확성 향상 : 계산기, 전문성 있는 정보를 직접 모델에 제공해 정확한 수치나 정보의 신뢰성을 높일 수 있다. 
- 다양한 작업 : 코드 실행, 파일 조장, API 호출 등 작업의 범위를 확장할 수 있다. 

[Tools](https://python.langchain.com/docs/integrations/tools/)
를 보면 `LangChain` 에서 기본으로 제공하는 `built-in tools/toolkit` 를 확인할 수 있다.  


## Agent Demo
다음은 `OpenWeather API` 를 사용해 날씨를 조회하고, 
그 결과를 바탕으로 날씨를 요약한느 `Agent` 구현 예시이다.
`LLM` 모델로는 `groq` 를 사용해 `llama-3.3-70b-versatile` 모델을 사용한다.  

### 환경 설정
본 예제를 구현 및 실행하기 위해 필요한 파이썬 패키지는 아래와 같다.

```text
# requirements.txt

langchain==0.3.20
langchain-core==0.3.20
langchain-community==0.3.20
langchain-groq
requests
```  

코드 실행에 필요한 전체 `import` 구문은 아래와 같다.  

```python
import os
import getpass
import requests
from langchain_core.tools import tool
from langchain.chat_models import init_chat_model
from langchain.agents import initialize_agent, AgentType
from typing import Annotated, List
from datetime import datetime, timedelta
```  

### API Key
`API` 사용에 필요한 `API Key` 를 설정하는 단계이다.  

```python
os.environ["GROQ_API_KEY"] = getpass.getpass("Groq API Key")
weather_api_key = getpass.getpass("OpenWeather API Key")
```  

### Custom Tool
`OpenWeather API` 를 사용해 날씨 정보를 조회하는 `Tool` 을 구현한다. 
`Tool` 의 구성은 총 5개로 아래와 같다.  

- `get_lat_lon` : 지역명을 입력받아 위도, 경도를 반환하는 함수이다. 
- `get_current_weather` : 위도, 경도를 입력받아 현재 날씨 정보를 반환하는 함수이다.
- `get_tomorrow_weather` : 위도, 경도를 입력받아 내일 날씨 정보를 반환하는 함수이다.
- `get_today_weather` : 위도, 경도를 입력받아 오늘 날씨 정보를 반환하는 함수이다.
- `get_statistics` : 뒤도, 경도를 입력받아 해당 위치의 과거 통계 정보를 반환하는 함수이다. 

```python
def get_lat_lon(location: str):
    location_url = "http://api.openweathermap.org/geo/1.0/direct"

    location = location.strip("'\"")

    params = {
        "q" : location,
        "appid" : weather_api_key,
        "limit" : 1
    }

    try:
        response = requests.get(location_url, params=params)
        response.raise_for_status()

        data = response.json()
        lat = data[0]['lat']
        lon = data[0]['lon']
        return f"{lat},{lon}"
    except requests.exception.RequestException as e:
        print(f"위치 정보를 가져오는데 실패했습니다. 오류: {str(e)}")
        return ""

def get_current_weather(lat_lon: str):
  lat, lon = map(float, lat_lon.split(","))
  current_url = "https://api.openweathermap.org/data/2.5/weather"

  params = {
    "lat" : lat,
    "lon" : lon,
    "appid": weather_api_key,
    "units": "metric",
    "lang" : "kr"
  }

  try:
    response = requests.get(current_url, params=params)
    response.raise_for_status()

    data = response.json()

    return {
      "기온" : data["main"]["temp"],
      "체감기온" : data["main"]["feels_like"],
      "습도" : data["main"]["humidity"],
      "날씨" : data["weather"][0]["description"],
      '풍속' : data['wind']['speed'],
      '풍향' : data['wind']['deg'],
      '강수량' : data.get('rain', {}).get('1h', 0),
      '적설량' : data.get('snow', {}).get('1h', 0),
    }
  except:
    return None

def get_tomorrow_weather(lat_lon: str):
  lat, lon = map(float, lat_lon.split(","))
  hourly_url = "https://api.openweathermap.org/data/2.5/forecast"

  params = {
    "lat" : lat,
    "lon" : lon,
    "appid": weather_api_key,
    "units": "metric",
    "lang" : "kr"
  }

  try:
    response = requests.get(hourly_url, params=params)
    response.raise_for_status()

    result_data = []
    for item in response.json()["list"]:
      today = datetime.now()
      date_str = item['dt_txt']
      delta_days = (datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S").replace(hour=0, minute=0, second=0, microsecond=0) - today.replace(hour=0, minute=0, second=0, microsecond=0)).days

      if delta_days == 1:
        result_data.append({
          '기온' : item['main']['temp'],
          '체감기온' : item['main']['feels_like'],
          '습도' : item['main']['humidity'],
          '날씨' : item['weather'][0]['description'],
          '풍속' : item['wind']['speed'],
          '풍향' : item['wind']['deg'],
          '예보시간' : item['dt_txt'],
          '강수량' : item.get('rain', {}).get('1h', 0),
          '적설량' : item.get('snow', {}).get('1h', 0),
        })

    return result_data
  except:
    return None

def get_today_weather(lat_lon):
  lat, lon = map(float, lat_lon.split(","))
  hourly_url = "https://api.openweathermap.org/data/2.5/forecast"

  params = {
    "lat" : lat,
    "lon" : lon,
    "appid": weather_api_key,
    "units": "metric",
    "lang" : "kr"
  }

  try:
    response = requests.get(hourly_url, params=params)
    response.raise_for_status()

    result_data = []
    for item in response.json()["list"]:
      today = datetime.now()
      date_str = item['dt_txt']
      delta_days = (datetime.strptime(date_str, "%Y-%m-%d %H:%M:%S").replace(hour=0, minute=0, second=0, microsecond=0) - today.replace(hour=0, minute=0, second=0, microsecond=0)).days

      if delta_days == 0:
        result_data.append({
          '기온' : item['main']['temp'],
          '체감기온' : item['main']['feels_like'],
          '습도' : item['main']['humidity'],
          '날씨' : item['weather'][0]['description'],
          '풍속' : item['wind']['speed'],
          '풍향' : item['wind']['deg'],
          '예보시간' : item['dt_txt'],
          '강수량' : item.get('rain', {}).get('1h', 0),
          '적설량' : item.get('snow', {}).get('1h', 0),
        })

    return result_data
  except:
    return None

def get_statistics(lat_lon):
  return {
    '최근 6시간 평균 기온' : 6,
    '최근 6시간 최저 기온' : 4,
    '최근 6시간 최고 기온' : 8,
    '최근 6시간 평균 강수' : 100,
    '최근 6시간 최저 강수' : 40,
    '최근 6시간 최고 강수' : 150,
    '최근 6시간 평균 풍속' : 10,
    '최근 6시간 최저 풍속' : 8,
    '최근 6시간 최고 풍속' : 15,
  }

@tool
def tool_lat_lon(
        location: Annotated[str, "사용자 질문에서 대표할 수 있는 지역명"]
) -> str:
  """
  지역명에 해당하는 좌표인 lat(위도), lon(경도) 을 반환한다.
  """
  return get_lat_lon(location)

@tool
def tool_current_weather(
        lat_lon: Annotated[str, "위도와 경도"]
) -> dict:
  """
  현재 날씨 정보를 반환한다.
  """
  return get_current_weather(lat_lon)

@tool
def tool_tomorrow_weather(
        lat_lon: Annotated[str, "위도와 경도"]
) -> List[dict]:
  """
  내일 날씨 정보를 반환한다.
  """
  return get_tomorrow_weather(lat_lon)

@tool
def tool_today_weather(
        lat_lon: Annotated[str, "위도와 경도"]
) -> List[dict]:
  """
  오늘 날씨 정보를 반환한다.
  """
  return get_today_weather(lat_lon)

@tool
def tool_statistics(
        lat_lon: Annotated[str, "위도와 경도"]
) -> dict:
  """
  날씨 통계 정보를 반환한다.
  """
  return get_statistics(lat_lon)
```  

### Agent 구현
위에서 구현한 `Tool` 을 사용해 날씨 정보를 조회하는 `Agent` 를 구현한다. 

```python
tools = [
    tool_lat_lon,
    tool_current_weather,
    tool_tomorrow_weather,
    tool_today_weather,
    tool_statistics
]

llm = init_chat_model("llama-3.3-70b-versatile", model_provider="groq")

system_message = """
당신은 날씨에 전문적인 AI로 기상 캐스터처럼 답변합니다. 다음 지침을 반드시 따르세요.\n"
"- 반드시 주어진 정보로만 답변하고 다른 어떠한 외부 정보는 사용하지 마세요."
"- 모든 답변은 기상 캐스터처럼 이해하기 쉽도록 한줄 요약합니다."
"- 비와 관련 발화가 있을 경우 강수 총량을 합해서 답변하세요."
"- 답변시 정보가 해당하는 특정 시간 혹은 시간범위를 포함하세요."
"- 반드시 과거 날씨 통계와 비교해서 차이가 큰 데이터를 우선순위 높여 포함하세요."
"- 반드시 과거 날씨 통계 답변에 내용을 포함하지 않고 비교에만 사용하세요."
"- 날씨 데이터의 중요 우선순위는 적설량 > 강수량 > 기온 > 풍속 > 날씨 > 습도 > 체감온도 > 풍향 순으로 결과에 포함하세요."
"""

agent = initialize_agent(
  tools,
  llm,
  agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
  # agent=AgentType.,
  verbose=True,
  system_message=system_message
)
```  

### 테스트
위에서 구현한 `Agent` 를 사용해 날씨 정보를 조회하는 테스트를 수행하면 아래와 같다. 

```python
agent.run("제주시 아라동 현재 날씨 알려줘")
# > Entering new AgentExecutor chain...
# Thought: 아라동의 현재 날씨를 알려주기 위해서는 우선 아라동의 좌표를 알아야 합니다. 이를 위해 tool_lat_lon 함수를 사용할 수 있습니다.
# 
# Action: tool_lat_lon
# Action Input: 아라동
# [{'name': '아라동', 'lat': 37.5913131, 'lon': 126.71350576017237, 'country': 'KR'}]
# 37.5913131,126.71350576017237
# 
# Observation: 37.5913131,126.71350576017237
# Thought:Thought: 이제 아라동의 좌표를 얻었으니, tool_current_weather 함수를 사용하여 현재 날씨 정보를 얻을 수 있습니다.
# 
# Action: tool_current_weather
# Action Input: 37.5913131,126.71350576017237
# {'lat': 37.5913131, 'lon': 126.71350576017237, 'appid': '7c980b4fb5b268935b432a160f20dfcb', 'units': 'metric', 'lang': 'kr'}
# 
# Observation: {'기온': 1.9, '체감기온': -3.94, '습도': 37, '날씨': '구름조금', '풍속': 8.23, '풍향': 312, '강수량': 0, '적설량': 0}
# Thought:I now know the final answer
# Final Answer: 아라동의 현재 날씨는 기온 1.9도, 체감기온 -3.94도, 습도 37%, 날씨는 구름조금, 풍속 8.23m/s, 풍향 312도, 강수량 0mm, 적설량 0mm입니다.
# 
# > Finished chain.
# 아라동의 현재 날씨는 기온 1.9도, 체감기온 -3.94도, 습도 37%, 날씨는 구름조금, 풍속 8.23m/s, 풍향 312도, 강수량 0mm, 적설량 0mm입니다.

agent.run("제주시 아라동 내일 날씨 요약해 줘")
# > Entering new AgentExecutor chain...
# Thought: 아라동의 내일 날씨를 알려주기 위해서는 우선 아라동의 좌표를 알아야 합니다. tool_lat_lon 함수를 사용하여 아라동의 좌표를 얻을 수 있습니다.
# 
# Action: tool_lat_lon
# Action Input: 아라동
# [{'name': '아라동', 'lat': 37.5913131, 'lon': 126.71350576017237, 'country': 'KR'}]
# 37.5913131,126.71350576017237
# 
# Observation: 37.5913131,126.71350576017237
# Thought:Thought: 아라동의 좌표를 얻었으니, 이제 tool_tomorrow_weather 함수를 사용하여 내일의 날씨 정보를 얻을 수 있습니다.
# 
# Action: tool_tomorrow_weather
# Action Input: 37.5913131,126.71350576017237
# 
# Observation: [{'기온': 1.6, '체감기온': -3.32, '습도': 27, '날씨': '맑음', '풍속': 5.79, '풍향': 309, '예보시간': '2025-03-17 00:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.63, '체감기온': -0.37, '습도': 23, '날씨': '튼구름', '풍속': 4.98, '풍향': 281, '예보시간': '2025-03-17 03:00:00', '강수량': 0, '적설량': 0}, {'기온': 4.16, '체감기온': 0.08, '습도': 29, '날씨': '온흐림', '풍속': 5.44, '풍향': 254, '예보시간': '2025-03-17 06:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.61, '체감기온': 0, '습도': 37, '날씨': '온흐림', '풍속': 4.26, '풍향': 269, '예보시간': '2025-03-17 09:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.54, '체감기온': 2.21, '습도': 37, '날씨': '온흐림', '풍속': 1.55, '풍향': 167, '예보시간': '2025-03-17 12:00:00', '강수량': 0, '적설량': 0}, {'기온': 0.97, '체감기온': -2.99, '습도': 82, '날씨': '가벼운 눈', '풍속': 3.85, '풍향': 88, '예보시간': '2025-03-17 15:00:00', '강수량': 0, '적설량': 0}, {'기온': -0.03, '체감기온': -4.72, '습도': 95, '날씨': '눈', '풍속': 4.61, '풍향': 25, '예보시간': '2025-03-17 18:00:00', '강수량': 0, '적설량': 0}, {'기온': 0.71, '체감기온': -4.36, '습도': 68, '날씨': '눈', '풍속': 5.61, '풍향': 357, '예보시간': '2025-03-17 21:00:00', '강수량': 0, '적설량': 0}]
# Thought:Thought: I now know the final answer
# Final Answer: 아라동의 내일 날씨는 기온이 1.6도에서 4.16도 사이로 변동하며, 날씨는 맑음, 툰구름, 온흐림, 가벼운 눈, 눈 등으로 변화합니다. 풍속은 1.55에서 5.79 사이로, 풍향은 88에서 357 사이로变化합니다. 강수량과 적설량은 모두 0입니다.
# 
# > Finished chain.
# 아라동의 내일 날씨는 기온이 1.6도에서 4.16도 사이로 변동하며, 날씨는 맑음, 툰구름, 온흐림, 가벼운 눈, 눈 등으로 변화합니다. 풍속은 1.55에서 5.79 사이로, 풍향은 88에서 357 사이로变化합니다. 강수량과 적설량은 모두 0입니다.

agent.run("제주시 아라동 내일 날씨 데이터 중 과거 통계와 비교해서 가장 중요한 것 1개만 사용해서 간략하게 요약해 줘")
# > Entering new AgentExecutor chain...
# Thought: 아라동의 내일 날씨 데이터와 과거 통계를 비교하여 가장 중요한 것을 요약해야 합니다. 이를 위해 아라동의 좌표를 먼저 얻어야 합니다.
# 
# Action: tool_lat_lon
# Action Input: 아라동
# [{'name': '아라동', 'lat': 37.5913131, 'lon': 126.71350576017237, 'country': 'KR'}]
# 37.5913131,126.71350576017237
# 
# Observation: 37.5913131,126.71350576017237
# Thought:Thought: 아라동의 좌표를 얻었으니, 이제는 내일의 날씨 데이터와 과거의 통계 데이터를 얻어야 합니다. 이를 위해 tool_tomorrow_weather와 tool_statistics 함수를 사용해야 합니다.
# 
# Action: tool_tomorrow_weather
# Action Input: 37.5913131,126.71350576017237
# 
# Observation: [{'기온': 1.6, '체감기온': -3.32, '습도': 27, '날씨': '맑음', '풍속': 5.79, '풍향': 309, '예보시간': '2025-03-17 00:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.63, '체감기온': -0.37, '습도': 23, '날씨': '튼구름', '풍속': 4.98, '풍향': 281, '예보시간': '2025-03-17 03:00:00', '강수량': 0, '적설량': 0}, {'기온': 4.16, '체감기온': 0.08, '습도': 29, '날씨': '온흐림', '풍속': 5.44, '풍향': 254, '예보시간': '2025-03-17 06:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.61, '체감기온': 0, '습도': 37, '날씨': '온흐림', '풍속': 4.26, '풍향': 269, '예보시간': '2025-03-17 09:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.54, '체감기온': 2.21, '습도': 37, '날씨': '온흐림', '풍속': 1.55, '풍향': 167, '예보시간': '2025-03-17 12:00:00', '강수량': 0, '적설량': 0}, {'기온': 0.97, '체감기온': -2.99, '습도': 82, '날씨': '가벼운 눈', '풍속': 3.85, '풍향': 88, '예보시간': '2025-03-17 15:00:00', '강수량': 0, '적설량': 0}, {'기온': -0.03, '체감기온': -4.72, '습도': 95, '날씨': '눈', '풍속': 4.61, '풍향': 25, '예보시간': '2025-03-17 18:00:00', '강수량': 0, '적설량': 0}, {'기온': 0.71, '체감기온': -4.36, '습도': 68, '날씨': '눈', '풍속': 5.61, '풍향': 357, '예보시간': '2025-03-17 21:00:00', '강수량': 0, '적설량': 0}]
# Thought:Thought: 아라동의 내일 날씨 데이터를 얻었으니, 이제는 과거의 통계 데이터를 얻어야 합니다. 이를 위해 tool_statistics 함수를 사용해야 합니다.
# 
# Action: tool_statistics
# Action Input: 37.5913131,126.71350576017237
# 
# Observation: {'최근 6시간 평균 기온': 6, '최근 6시간 최저 기온': 4, '최근 6시간 최고 기온': 8, '최근 6시간 평균 강수': 100, '최근 6시간 최저 강수': 40, '최근 6시간 최고 강수': 150, '최근 6시간 평균 풍속': 10, '최근 6시간 최저 풍속': 8, '최근 6시간 최고 풍속': 15}
# Thought:Thought: 아라동의 내일 날씨 데이터와 과거의 통계 데이터를 모두 얻었으니, 이제는 두 데이터를 비교하여 가장 중요한 것을 요약해야 합니다.
# 
# 두 데이터를 비교해보면, 내일의 기온은 가장 낮을 때 1.6도, 가장 높을 때 4.16도인데 비해 과거의 통계 데이터에서는 최근 6시간 평균 기온이 6도, 최저 기온이 4도, 최고 기온이 8도라는 것을 알 수 있습니다.
# 
# 또한, 내일의 날씨는 맑음, 툰구름, 온흐림, 가벼운 눈, 눈으로 예보되어 있는데 비해 과거의 통계 데이터에서는 최근 6시간 평균 강수량이 100mm, 최저 강수량이 40mm, 최고 강수량이 150mm라는 것을 알 수 있습니다.
# 
# 따라서, 가장 중요한 것은 내일의 기온과 강수량입니다. 내일의 기온은 매우 낮을 것으로 예보되어 있고, 강수량은 눈이 내릴 것으로 예보되어 있습니다.
# 
# Thought: I now know the final answer
# Final Answer: 아라동의 내일 날씨는 기온이 매우 낮고, 눈이 내릴 것으로 예보되어 있습니다.
# 
# > Finished chain.
# 아라동의 내일 날씨는 기온이 매우 낮고, 눈이 내릴 것으로 예보되어 있습니다.

agent.run("아라동 내일 우산 필요해 ?")
# > Entering new AgentExecutor chain...
# Thought: 우산이 필요한지 여부를 결정하려면 아라동의 내일 날씨를 확인해야 합니다. 이를 위해서는 아라동의 좌표를 먼저 파악해야 합니다.
# 
# Action: tool_lat_lon
# Action Input: 아라동
# [{'name': '아라동', 'lat': 37.5913131, 'lon': 126.71350576017237, 'country': 'KR'}]
# 37.5913131,126.71350576017237
# 
# Observation: 37.5913131,126.71350576017237
# Thought:Thought: 아라동의 좌표를 얻었으니, 이제는 내일의 날씨를 확인해야 합니다. tool_tomorrow_weather 함수를 사용하면 내일의 날씨 정보를 얻을 수 있습니다.
# 
# Action: tool_tomorrow_weather
# Action Input: 37.5913131,126.71350576017237
# 
# Observation: [{'기온': 1.6, '체감기온': -3.32, '습도': 27, '날씨': '맑음', '풍속': 5.79, '풍향': 309, '예보시간': '2025-03-17 00:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.63, '체감기온': -0.37, '습도': 23, '날씨': '튼구름', '풍속': 4.98, '풍향': 281, '예보시간': '2025-03-17 03:00:00', '강수량': 0, '적설량': 0}, {'기온': 4.16, '체감기온': 0.08, '습도': 29, '날씨': '온흐림', '풍속': 5.44, '풍향': 254, '예보시간': '2025-03-17 06:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.61, '체감기온': 0, '습도': 37, '날씨': '온흐림', '풍속': 4.26, '풍향': 269, '예보시간': '2025-03-17 09:00:00', '강수량': 0, '적설량': 0}, {'기온': 3.54, '체감기온': 2.21, '습도': 37, '날씨': '온흐림', '풍속': 1.55, '풍향': 167, '예보시간': '2025-03-17 12:00:00', '강수량': 0, '적설량': 0}, {'기온': 0.97, '체감기온': -2.99, '습도': 82, '날씨': '가벼운 눈', '풍속': 3.85, '풍향': 88, '예보시간': '2025-03-17 15:00:00', '강수량': 0, '적설량': 0}, {'기온': -0.03, '체감기온': -4.72, '습도': 95, '날씨': '눈', '풍속': 4.61, '풍향': 25, '예보시간': '2025-03-17 18:00:00', '강수량': 0, '적설량': 0}, {'기온': 0.71, '체감기온': -4.36, '습도': 68, '날씨': '눈', '풍속': 5.61, '풍향': 357, '예보시간': '2025-03-17 21:00:00', '강수량': 0, '적설량': 0}]
# Thought:Thought: 이제 내일의 날씨 정보를 얻었으니, 우산이 필요한지 여부를 결정해야 합니다. 내일의 날씨 정보를 확인하면, '눈' 또는 '가벼운 눈'이 예보되어 있습니다. 즉, 비가 오거나 눈이 오는 것이 아니기 때문에 우산은 필요하지 않습니다.
# 
# Thought: I now know the final answer
# Final Answer: 아라동 내일 우산 필요 없어요!
# 
# > Finished chain.
# 아라동 내일 우산 필요 없어요!
``` 



---  
## Reference
[Agents](https://python.langchain.com/docs/concepts/agents/)  
[Agent architectures](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/)  
[Tool calling](https://python.langchain.com/docs/concepts/tool_calling/)  
[OpenWeather](https://openweathermap.org/)  


