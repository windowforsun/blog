--- 
layout: single
classes: wide
title: "[Time] ARMA"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: '자기회귀와 이동평균 두 가지 구성요소를 결합한 모델인 ARMA 에 대해 알아보자'
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
toc: true
use_math: true
---  

## Autoregressive Moving Average (ARMA) Model
`ARMA`(Autoregressive Moving Average, 자기회귀 이동평균) 모델은 시계열 데이터 예측 및 분석에 널리 사용되는 통계적 모델 중 하나로, 
자기회귀(`AR`)와 이동평균(`MA`) 두 가지 구성 요소를 결합한 모델이다. 
시계열의 현재 값이 과거의 값들과 과거의 오차항의 선형 결합으로 표현된다는 점에서,
`ARMA` 모델은 시간에 따라 상호 관련성이 존재하는 데이터를 효과적으로 설명할 수 있다. 

`ARMA` 모델은 `ACF` 도식이나 `PACF` 도식에서 차수를 추정할 수 없는 경우에 사용할 수 있다. 
두 도식 모두 천천히 감쇄하는 패턴이나 사인 곡선 패턴을 나타내는 경우이다. 

`MA`, `AR`m `ARMA` 모델을 비교하면 아래와 같다.  

| 구분    | AR (자기회귀) | MA (이동평균) | ARMA (자기회귀 이동평균) |
|---------|---------------|--------------|--------------------------|
| 모델 구조 | 과거의 값에 의존 | 과거 오차에 의존 | 과거 값 + 과거 오차에 모두 의존 |
| 파라미터 | AR 계수(p개) | MA 계수(q개) | AR 계수(p개) + MA 계수(q개) |
| 데이터 패턴 | 자기상관이 강한 데이터 | 오차의 자기상관이 강한 데이터 | 두 패턴이 모두 존재할 때 적합 |
| 예측력 | 단순 패턴에 적합 | 단순 패턴에 적합 | 복합 패턴에 적합, 예측력 우수 |

`ARMA` 모델링을 위해서는 `AIC`(Akaike Information Criterion, 아카이케 정보 기준) 을 사용한다. 
이를 통해 시게열에 대한 최적으 `p`, `q` 값을 결정하고, 
모델 잔차의 상관관계도 `Q-Q` 도식과 밀도 도식을 사용하는 `잔자 분석` 을 통해 잔차가 백섹소음과 유사한지 평가한다. 
`잔자 분석`을 통해 유효한 것으로 판정되면 `ARMA` 모델을 통해 시게열 예측을 수행해 볼 수 있다.  


`ARMA` 모델을 사용해서 데이터 세넡의 대역폭 사용량을 예측해 본다.
먼저 2019년 1월 부터 특정된 시간당 대역폭 데이터를 사용한다.
대역폭은 초당 메가비트(Mbps) 단위로 측정되었다.
원본 데이터를 도식화하면 아래와 같다.

```python
import matplotlib.pyplot as plt
import pandas as pd

df = pd.read_csv('../data/bandwidth.csv')

df.head()

fig, ax = plt.subplots()

ax.plot(df['hourly_bandwidth'])
ax.set_xlabel('Time')
ax.set_ylabel('Hourly bandwith usage (MBps)')

plt.xticks(
    np.arange(0, 10000, 730), 
    ['Jan 2019', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec', 'Jan 2020', 'Feb'])

fig.autofmt_xdate()
plt.tight_layout()
```   

![그림 1]({{site.baseurl}}/img/datascience/arma-1.png)


데이터를 보면 장기적인 추세가 있으므로 정상적은 아닌 것으로 보인다. 
그리고 주기적인 형태는 없어 계절성은 존재하지 않는 것으로 보인다. 
이후 `ARMA` 모델의 모델링 절차를 살펴보고 이게 맞춰 진행해 본다.  

### AIC
`AIC`(Akaike Information Criterion, 아카이케 정보 기준)는 통계 모델의 품질을 평가하는 데 사용되는 지표이다. 
모델의 품질을 다른 모델들과 비교하여 상대적으로 정량화한다. 
데이터에 피팅할 때 일부 정보는 손실되는데 `AIC` 는 모델에 의해 손실되는 정보의 양을 상대적으로 정량화하고, 
모델들을 정량적으로 비교할 수 있다. 
`AIC` 값이 낮을 수록 손실된 정보가 적다는 의미이므로 더 우수한 모델로 판단할 수 있다.  


### Order of ARMA Process
`ARMA` 모델에서도 `ARMA(p, q)` 에서 `p`, `q` 를 식별하는 과정이 매우 중요하다. 
`p` 는 `AR` 자기회귀 차수를 나타내고, `q` 는 `MA` 이동평균 차수를 나타낸다. 
`ARMA` 모델에서 `p`, `q` 를 식별하는 모델링 과정은 다음과 같다.  

```mermaid
flowchart TD
    data_agg[데이터 수집] --> station[정상적-ADF ?]
    station -->|NO| trans[변환-차분 수행]
    trans --> station
    station -->|YES| p-q[p, q 목록 생성]
    p-q --> ARMA-fit[ARMA에 모든 p, q 조합 피팅]
    ARMA-fit --> AIC[AIC 가 가장 낮은 최적 p, q 선택]
    AIC --> residual[잔차 분석]
    residual --> Q-Q[Q-Q 도식이 직선?]
    residual --> ljungbox[융박스-잔차가 백색잡음?]
    Q-Q -->|YES| predict[시계열 예측]
    ljungbox -->|NO| p-q
    Q-Q -->|NO| p-q
    ljungbox -->|YES| predict
```  

`ADF` 를 사용해 현재 시계열 데이터가 정상적인지 확인한다.  

```python
ADF_result = adfuller(df['hourly_bandwidth'])

print(f'ADF Statistic: {ADF_result[0]}')
# ADF Statistic: -0.8714653199452314
print(f'p-value: {ADF_result[1]}')
# p-value: 0.7972240255014685
```  

`ADF` 통계값이 큰 음수가 아니고, `p-value` 가 `0.05` 보다 크므로 시계열은 정상적이지 않다.
데이터를 정상적으로 만들기 위해 변환(차분)을 수행해 다시 `ADF` 검정을 수행한다.  

```python
bandwidth_diff = np.diff(df.hourly_bandwidth, n=1)

fig, ax = plt.subplots()

ax.plot(bandwidth_diff)
ax.set_xlabel('Time')
ax.set_ylabel('Hourly bandwith usage - diff (MBps)')

plt.xticks(
    np.arange(0, 10000, 730),
    ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec', '2020', 'Feb'])

fig.autofmt_xdate()
plt.tight_layout()

ADF_result = adfuller(bandwidth_diff)

print(f'ADF Statistic: {ADF_result[0]}')
# ADF Statistic: -20.69485386378902
print(f'p-value: {ADF_result[1]}')
# p-value: 0.0
```  
