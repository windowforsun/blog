--- 
layout: single
classes: wide
title: "[TimeSeries] Baseline Model"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - Data Science
    - Time Series
    - Random Walk
toc: true
use_math: true
---  

## Baseline Model
`Baseline Model` 은 시계열에서 문제 해결을 위한 가장 단순하거나 기본적인 예측 방법을 의미한다. 
이는 복잡한 모델을 적용하기 전에 성능을 평가하고 비교하는 기준점으로 사용되는 만큼 비교 대상이 되는 `기본 모델`을 지칭한다.  

우리가 만든 복합한 모델이 실제로 쓸만한지 확인하려면 `아무 생각 없이 예측해도 이 정도는 나온다` 라는 기준이 필요하다. 
이러한 기준을 제공하는 것이 바로 `Baseline Model` 이다. 

대표적인 `Baseline Model` 로는 다음과 같은 것들이 있다.

- Last Value: 가장 최근의 관측값을 그대로 예측값으로 사용하는 방법이다. 예를 들어, 내일의 주가를 오늘의 주가로 예측하는 것이다.
- Seasonal Naive: 계절성이 있는 데이터에 대해, 이전 계절의 동일한 시점의 값을 예측값으로 사용하는 방법이다. 예를 들어, 작년 같은 날의 기온을 올해 같은 날의 기온으로 예측하는 것이다
- Mean Value: 과거 관측값의 평균을 예측값으로 사용하는 방법이다. 예를 들어, 지난 30일간의 평균 기온을 내일의 기온으로 예측하는 것이다.
- Drift Method: 과거 관측값의 추세를 고려하여 예측값을 계산하는 방법이다. 예를 들어, 지난 30일간의 기온 변화량을 기반으로 내일의 기온을 예측하는 것이다.


이번 포스팅에서 설명하는 예측 모델은 말그대로 `Baseline Model` 이다. 
아주 단순하고 기본적으로 접근할 수 있는 모델을 의미하기 때문에 복잡한 예측에 대한 내용은 포함돼있지 않다. 
이러한 베이스라인 모델이 필요한 이유는 앞서 언급한 것과 같이 `복잡한 모델을 적용하기 전에 성능을 평가하고 비교하는 기준점으로 사용` 하기 위함이다. 
따라서, 이 모델을 통해 얻은 결과를 바탕으로 더 복잡한 모델을 개발하고 평가할 수 있음을 명심해야 한다.  
