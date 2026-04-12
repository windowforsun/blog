--- 
layout: single
classes: wide
title: "[Time] SARIMAX"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: '계절성과 비정상성을 처리하면서 외부 변수까지 활용할 수 있는 시계열 예측 통계적 모델 SARIMAX 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - Data Science
    - Time Series
    - AR
    - MA
    - ARMA
    - ARIMA
    - SARIMA
    - SARIMAX
toc: true
use_math: true
---  

## Seasonal ARIMAX (SARIMAX) Model
`SARIMAX`(Seasonal AutoRegressive Integrated Moving Average with eXogenous regressors) 모델은 
시계열 데이터의 예측에 사용되는 통계적 모델이다. 
이 모델은 계절성(`Seasonality`)과 비정상성(`Non-stationarity`)을 처리할 수 있으며, 외부 요인/변수(`Exogenous Variables`)이 
시계열에 미치는 영향을 함께 분석할 수 있도록 확장한 모델이다. 
`SARIMAX` 는 `SARIMA` 모델 구조에 외생 변수(`X`)를 추가하여, 계절적/비계절적 패턴뿐 아니라 기상, 경제 지표, 이벤트 등 
외부 변수의 영향까지 통합적으로 반영할 수 있다. 
보통 `SARIMAX(p,d,q)(P,D,Q)m + X` 형태로 표현하고, 
`(p,d,q)`는 비계절적 부분, `(P,D,Q)m`는 계절적 부분을 나타낸다. 
이는 `SARIMA` 와 동일하게 `AR`, `차분`, `MA`, `계절 AR`, `계절 차분`, `계절 MA`, `계절 주기` 를 의미하고, 
`X` 는 외생 변수의 수 또는 종류를 나타낸다. 
