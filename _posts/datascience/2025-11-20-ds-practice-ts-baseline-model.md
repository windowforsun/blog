--- 
layout: single
classes: wide
title: "[TimeSeries] Baseline Model"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: '시계열 예측에서 Baseline Model(기본 모델)의 개념과 구현 방법을 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - Data Science
    - Time Series
    - Random Walk
    - Baseline Model
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

### Load Data
베이스라인 모델을 적용하기 위해 `존슨엔드존슨` 의 1960년 ~ 1980년 까지의 분기별 `EPS`(주당순이익) 데이터를 사용한다. 
우리는 이후 이 데이를 사용해서 1980년의 분기별 `EPS` 를 예측하는 베이스라인 모델을 만들 것이다.  

```python
import pandas as pd

df = pd.read_csv('../data/jj.csv')
df.head()

#         date	data
# 0	1960-01-01	0.71
# 1	1960-04-01	0.63
# 2	1960-07-02	0.85
# 3	1960-10-01	0.44
# 4	1961-01-01	0.61

df.tail()

#         date	data
# 79	1979-10-01	9.99
# 80	1980-01-01	16.20
# 81	1980-04-01	14.67
# 82	1980-07-02	16.02
# 83	1980-10-01	11.61
```  

로드한 데이터를 시각적으로 확인하기 위해 차트로 시각화하면 아래와 같다.  

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots()

ax.plot(df['date'], df['data'])
ax.set_xlabel('Date')
ax.set_ylabel('Earnings per share')
ax.axvspan(80, 83, color='gray', alpha=0.2)

plt.xticks(np.arange(0, 81, 8), [1960, 1962, 1964, 1966, 1968, 1970, 1972, 1974, 1976, 1978, 1980])

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/baseline-model-1.png)


### Train/Test Split
모델을 개발하기 전 필수로 수행하는 작업이 `Train/Test Split` 이다. 
이는 모델이 학습한 데이터와 평가에 사용하는 데이터를 분리하는 작업이다. 
시계열 데이터의 경우, 시간의 흐름에 따라 데이터를 분리하는 것이 중요하다. 
즉, 과거 데이터를 학습에 사용하고, 미래 데이터를 평가에 사용하는 방식으로 분리해야 한다.  

이를 위해 데이터 훈련 집합은 1960년 ~ 1979년 데이터로 하고, 
테스트 집합은 1980년 데이터로 분리한다.  

```python
# 1960 ~ 1979년 데이터
train = df[:-4]
# 1980년 데이터
test = df[-4:]

```  

### Mean value
`Mean value` 은 과거 관측값의 평균을 예측값으로 사용하는 방법이다. 
예를 들어, 지난 30일간의 평균 기온을 내일의 기온으로 예측하는 것이다. 
그러므로 우리는 1960년 ~ 1979년 까지의 `EPS` 평균을 계산하고, 이를 1980년의 모든 분기에 대한 예측값으로 사용한다. 


```python
import numpy as npㅈ

historical_mean = np.mean(train['data'])
historical_mean
# 4.3084999875

test.loc[:, 'pred_mean'] = historical_mean
#       date	data	pred_mean
# 80	1980-01-01	16.20	4.3085
# 81	1980-04-01	14.67	4.3085
# 82	1980-07-02	16.02	4.3085
# 83	1980-10-01	11.61	4.3085

```  

전체 평균으로 구한 예측 성능을 측정하기 위해 `MAPE`(Mean Absolute Percentage Error) 를 사용한다. 
이는 예측과 실제의 차이를 백분율로 나타내며, 데이터 단위와 상관없이 해석할 수 있다는 장점이 있다.  

```python
def mape(y_true, y_pred):
    return np.mean(np.abs((y_true - y_pred) / y_true)) * 100

mape_hist_mean = mape(test['data'], test['pred_mean'])
# 70.00752579965118
```  

`MAPE` 계산 결과 `70%` 만큼의 예측 오차가 있음을 알 수 있다. 
이제 평균으로 구한 예측과 실제 데이터를 차트로 그려 시각화하면 아래와 같다.  

```python
fig, ax = plt.subplots()

ax.plot(train['date'], train['data'], 'g-.', label='Train')
ax.plot(test['date'], test['data'], 'b-', label='Test')
ax.plot(test['date'], test['pred_mean'], 'r--', label='Mean')
ax.set_xlabel('Date')
ax.set_ylabel('Earnings per share')
ax.axvspan(80, 83, color='gray', alpha=0.2)
ax.legend(loc=2)

plt.xticks(np.arange(0, 85, 8), [1960, 1962, 1964, 1966, 1968, 1970, 1972, 1974, 1976, 1978, 1980])

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/baseline-model-2.png)



차트를 보면 데이터에 `양의 추세` 가 있음을 알 수 있다. 
그러므로 단순 평균으로는 증가하는 추세를 반영하지 못하므로 예측 오차가 클 수 밖에 없다.  


### Last Year Mean
이전 `Mean value` 는 전체 평균을 사용했기 때문에, 시계열 데이터의 특성을 충분히 반영하지 못했다. 
따라서, 이번에는 `Last Year Mean` 을 사용해 본다. 
이는 데이터에 양의 추세를 반영하기 위해 훈련 집합의 가장 최근 1년의 평균을 예측값으로 사용한다. 

```python
last_year_mean = np.mean(train['data'][-4:])
# 12.96

test.loc[:, 'pred__last_yr_mean'] = last_year_mean
#       date	     data	pred_mean	pred__last_yr_mean
# 80	1980-01-01	16.20	4.3085	    12.96
# 81	1980-04-01	14.67	4.3085	    12.96
# 82	1980-07-02	16.02	4.3085	    12.96
# 83	1980-10-01	11.61	4.3085	    12.96

mape_last_year_mean = mape(test['data'], test['pred__last_yr_mean'])
# 15.5963680725103


fig, ax = plt.subplots()

ax.plot(train['date'], train['data'], 'g-.', label='Train')
ax.plot(test['date'], test['data'], 'b-', label='Test')
ax.plot(test['date'], test['pred__last_yr_mean'], 'r--', label='Last Year Mean')
ax.set_xlabel('Date')
ax.set_ylabel('Earnings per share')
ax.axvspan(80, 83, color='gray', alpha=0.2)
ax.legend(loc=2)

plt.xticks(np.arange(0, 85, 8), [1960, 1962, 1964, 1966, 1968, 1970, 1972, 1974, 1976, 1978, 1980])

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/baseline-model-3.png)


결과를 보면 직전 전체 평균을 사용한 예측과 비교해서 `MAPE` 가 `15.6%` 로 크게 감소했음을 알 수 있다. 
이를 바탕으로 최근 데이터가 미래 예측에 더 유용하다는 점, 
즉 시계열 데이터의 자기상관관계(과거와 미래가 연결 됨)를 확인할 수 있다.  


### Last value
앞선 2가지 방식으로 보았을 때 전체 평균과 최근 1년 평균 중 최신 값을 더 많이 반영할 수록 예측 성능이 올라가는 것을 확인할 수 있었다. 
이러한 추론을 바탕으로 이번에는 훈련 집합의 마지막 값을 예측값으로 사용하는 `Last value` 방식을 적용해 본다. 

```python
last = train['data'].iloc[-1]
# 9.99

test.loc[:, 'pred_last'] = last
#           date	data	pred_mean	pred__last_yr_mean	pred_last
# 80	1980-01-01	16.20	4.3085	    12.96	            9.99
# 81	1980-04-01	14.67	4.3085	    12.96	            9.99
# 82	1980-07-02	16.02	4.3085	    12.96	            9.99
# 83	1980-10-01	11.61	4.3085	    12.96	            9.99

mape_last = mape(test['data'], test['pred_last'])
# 30.457277908606535


fig, ax = plt.subplots()

ax.plot(train['date'], train['data'], 'g-.', label='Train')
ax.plot(test['date'], test['data'], 'b-', label='Test')
ax.plot(test['date'], test['pred_last'], 'r--', label='Last value')
ax.set_xlabel('Date')
ax.set_ylabel('Earnings per share')
ax.axvspan(80, 83, color='gray', alpha=0.2)
ax.legend(loc=2)

plt.xticks(np.arange(0, 85, 8), [1960, 1962, 1964, 1966, 1968, 1970, 1972, 1974, 1976, 1978, 1980])

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/baseline-model-4.png)


하지만 이번에는 `MAPE` 가 `30.4%` 로 오히려 증가해 예측의 정확도가 떨어진 것을 알 수 있다. 
이는 마지막 값만 사용하면 데이터의 계절성(분기별 반복적 변화)을 반영하지 못해 오차가 처진 것으로 볼 수 있다. 

