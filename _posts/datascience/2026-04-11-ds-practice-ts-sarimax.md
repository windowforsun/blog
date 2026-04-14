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



`MA`, `AR`, `ARMA`, `ARIMA`, `SARIMA`, `SARIMAX` 모델과의 차이점은 다음 표와 같다.

| 구분    | AR | MA | ARMA | ARIMA | SARIMA | SARIMAX |
|---------|----|----|------|-------|--------|---------|
| 모델 구조 | 과거 값 | 과거 오차 | 과거 값 + 오차 | 차분 후 과거 값, 오차 | 계절 차분 후 과거 값, 오차, 계절성 | SARIMA 구조 + 외생 변수(X) |
| 정상성 가정 | 정상 | 정상 | 정상 | 정상/비정상 | 정상/비정상/계절성 | 정상/비정상/계절성/외부요인 |
| 파라미터 | p | q | p, q | p, d, q | p, d, q, P, D, Q, s | p, d, q, P, D, Q, s, X |
| 적용 데이터 | 단순 정상 | 단순 정상 | 복합 정상 | 복합 정상/비정상 | 계절성·비계절성 시계열 | 계절성·비계절성·외부 요인 포함 시계열 |
| 예측력 | 보통 | 보통 | 높음 | 매우 높음 | 계절성 데이터에 최고 | 외부요인까지 반영, 예측력 최상 |
| 특징 | 자기상관 데이터 | 오차 자기상관 | 두 패턴 혼재 | 정상화 과정 포함 | 계절성·비정상성 모두 처리, 복잡한 패턴 | 외생 변수 포함, 실무 적용도 최고 |


### Identifying Exogenous Variables
외생 변수(`Exogenous Variables`)는 시계열 데이터에 영향을 미칠 수 있는
외부 요인 또는 변수를 의미한다. 
그러므로 `SARIMAX` 모델을 구축할 때는 시계열에 영향을 미칠 수 있는 외생 변수를 식별하는 것이 중요하다. 

`SARIMAX` 모델에서는 미국 거시경제 데이터를 사용한다. 
먼저 데이터를 불러오고 주요 필드를 확인하면 아래와 같다. 
테스트에 필요한 미국 거시경제 데이터는 `statsmodels` 패키지에서 제공하는 `macrodata` 데이터 세트를 사용한다.  
