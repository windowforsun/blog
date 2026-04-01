--- 
layout: single
classes: wide
title: "[Time] SARIMA"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: '시계열 데이터 예측 및 분석의 ARIMA 에서 계절 자기회귀와 계절 이동평균 그리고 계절 차분을 추가한 SARIMA 에 대해 알아보자.'
author: "window_for_sun"
header-style: text
categories :
  - Data Science
tags:
    - Practice
    - Data Science
    - Time Series
    - AR
    - MA
    - ARMA
    - ARIMA
    - SARIMA
toc: true
use_math: true
---  

## Seaonal ARIMA (SARIMA) Model
`Seasonal ARIMA (SARIMA)` 모델은 시계열 데이터에서 예측 및 분석에서 계절성을 포함하는 
복합적인 패턴을 효과적으로 설명하기 위해 사용되는 확장된 통계 모델이다. 
`SARIMA` 는 `ARIMA` 모델에 계절 자기회귀 `Seasonal AR`, 계절 이동평균 `Seasonal MA` 
그리고 계절 차분 `Seasonal Differencing` 을 추가해, 
데이터 내 단기적 변화뿐만 아니라 일정 주기의 반복적인 변동(계절성)을 동시에 포착할 수 있다. 

일반적으로 `SARIMA(p,d,q)(P,D,Q)m` 로 표기하며, 
앞의 `(p,d,q)` 는 `ARIMA` 의 비계절 차수, 
뒤의 `(P,D,Q)m` 는 계절 `AR`, 계절 차분, 계절 `MA`, `m` 는 계절 주기를(`12개월`, `4개월` 등) 의미한다. 
`SARIMA` 는 비정상성과 계절성 모두를 처리 가능하며, 
연간/월간/주간 등 주기적 특성이 강한 경제, 기후, 생산량 등 더욱 폭 넓은 시계열 데이터 분석에 널리 활용된다.  

`MA`, `AR`, `ARMA`, `ARIMA`, `SARIMA` 모델과의 차이점은 다음 표와 같다.

| 구분    | AR (자기회귀) | MA (이동평균) | ARMA (자기회귀 이동평균) | ARIMA (자기회귀 누적 이동평균) | SARIMA (계절 자기회귀 누적 이동평균) |
|---------|---------------|--------------|--------------------------|-------------------------------|--------------------------------------|
| 모델 구조 | 과거의 값에 의존 | 과거 오차에 의존 | 과거 값 + 과거 오차 | 차분 후 과거 값 + 오차 | 계절 차분 후 과거 값, 오차, 계절성 패턴 |
| 정상성 가정 | 정상 데이터 | 정상 데이터 | 정상 데이터 | 비정상/정상 모두 가능 | 비정상, 계절성 모두 가능 |
| 파라미터 | p | q | p, q | p, d, q | p, d, q, P, D, Q, s |
| 적용 데이터 | 단순 정상 시계열 | 단순 정상 시계열 | 복합 정상 시계열 | 복합 비정상 및 정상 시계열 | 계절적·비계절적 복합 시계열 |
| 예측력 | 보통 | 보통 | 높음 | 매우 높음 | 계절성 데이터에 최고 |
| 특징 | 자기상관이 강한 데이터에 적합 | 오차의 자기상관이 강할 때 적합 | 두 패턴 혼재시 적합 | 정상화 과정 포함, 유연성 최고 | 계절성·비정상성 모두 처리, 복잡한 패턴 분석 가능 |


`SARIMA` 모델에서는 `1949~1960` 년까지 항공사의 월별 총 항공 승객 수 데이터를 사용해
시계열 분석하고 최종적으로 `SARIMA` 모델을 구축하는 과정을 단계별로 설명한다. 
해딩 데이터를 로드하고 도식화하면 아래와 같다.  

```python
df = pd.read_csv('../data/air-passengers.csv')
df.head()
# Month	Passengers
# 0	    1949-01	112
# 1	    1949-02	118
# 2	    1949-03	132
# 3	    1949-04	129
# 4	    1949-05	121

fig, ax = plt.subplots()

ax.plot(df['Month'], df['Passengers'])
ax.set_xlabel('Date')
ax.set_ylabel('Number of air passengers')

plt.xticks(np.arange(0, 145, 12), np.arange(1949, 1962, 1))

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/sarima-1.png)
