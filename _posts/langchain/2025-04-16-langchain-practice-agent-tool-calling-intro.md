--- 
layout: single
classes: wide
title: "[LangChain] LangChain Introduction"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
toc: true
use_math: true
---  

## LangChain Agent
`LangChain Agent` 는 `LLM` 기반 애플리케이션에서 중요한 역할을 하는 컴포넌트로, 
`AI` 시스템이 더욱 자율적이고 목표 지향적으로 작업을 수행할 수 있게 해주는 컴포넌트이다. 
`Agent` 는 주어진 목표를 달성하기 위해 상호작용하며 의사 결정을 내리고 해동을 취하는 지능형 시스템이다.  

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
