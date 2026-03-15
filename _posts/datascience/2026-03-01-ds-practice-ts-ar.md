--- 
layout: single
classes: wide
title: "[TimeSeries] AR(AutoRegressive)"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: '자기회귀(AR, Autoregressive) 프로세스 모델링과 예측에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - Data Science
    - Time Series
    - AR
    - AutoRegressive
toc: true
use_math: true
---  

## Autoregressive Process
`Autoregressive Process`(자기회귀 과정, AR)는 시계열 분석에서 널리 사용되는 통계 모델이다. 
현재의 데이터 값이 과거의 자기 자신의 값들의 선형 결합(가중합)으로 표현되는 과정을 의미한다. (현재값이 과거값에 선형적으로 의존)
현재 시점의 값이 바로 이전 시점이나 그보다 더 이전 시점의 값들에 일정한 계수를 곱해 더한 값과, 
우연적 오차(백색잡음)가 더해 만들어지는 시계열 모델이라고 할 수 있다. 
즉, 과거의 값들이 현재 값에 영향을 주는 자기 의존적 구조를 가진 모델이다.  

앞서 살펴본 `MA` 이동평균에서도 `ACF` 도식을 보면 과거 몇개의 오차항에 대해 의존하는 것을 볼 수 있었다. 
여기서 `MA` 는 현재 값이 과저의 오차(백색잡음) 에만 의존하는 것을 의미한다. 
즉 이는 직접적으로 과저의 실제 데이터 값들이 아니라, 당시 발생했던 오차항에만 의존하는 것이다.
이와 비교해 `AR` 자기회귀는 현재 값이 과거의 실제 데이터 값에 직접적으로 의존하는 것을 의미한다.  

자가회귀 모델은 `AR(p)` 로 표기하는데 , 여기서 `p` 는 모델이 의존하는 과거 시점의 수를 나타낸다. 
예를 들어, `AR(1)` 모델은 바로 이전 시점의 값에만 의존하고, 
`AR(2)` 모델은 바로 이전 시점과 그 이전 시점의 값에 의존한다는 것을 의미한다. 


### Order of AR Process
이동평균과 마찬가지로 이동평균에서는 `MA(q)` 에서 `q` 를 식별하는 과정이 매우 중요했듯이, 
자기회귀에서는 `AR(p)` 에서 `p` 를 식별하는 과정이 매우 중요하다. 

이동평균에서 `q` 를 식별하는 과정을 확장해 `AR` 에서 `p` 를 식별하는 과정은 다음과 같다. 

```mermaid
flowchart TD
    data_agg[데이터 수집] --> station[정상적-ADF ?]
    station -->|NO| trans[변환-차분 수행]
    trans --> station
    station -->|YES| acf[ACF 도식]
    acf --> autorelation[자기상관관계 유무 확인]
    autorelation -->|NO| random(확률 보행 프로세스)
    autorelation -->|YES| q_value[ACF에서 자기상관관계가 유의하지 않는 지연 q 확인]
    q_value -->|YES| ma[이동평균 과정]
    q_value -->|NO| pacf[PACF 도식]
    pacf -->|YES| ar[자기회귀 과정]
    pacf -->|NO| no_ar[자기회귀 과정 아님]
```  

이동평균 차수 삭별과정에서 `PACF` 도식이 추가되어 `AR` 자기회귀 과정의 지연 `p` 를 식별하는데 사용한다.  

자기회귀 과정에서는 예제를 위해 소매점의 주간 평균 유동인구를 사용한다. 

```python
df = pd.read_csv('../data/foot_traffic.csv')

df.head()
#   foot_traffic
# 0	500.496714
# 1	500.522366
# 2	501.426876
# 3	503.295990
# 4	504.132695

fig, ax = plt.subplots()

ax.plot(df['foot_traffic'])
ax.set_xlabel('Time')
ax.set_ylabel('Average weekly foot traffic')

plt.xticks(np.arange(0, 1000, 104), np.arange(2000, 2020, 2))

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-1.png)


데이터를 로드했기 때문에 바로 정상성 테스트 `ADF` 검정을 수행한다.  

```python
ADF_result = adfuller(df['foot_traffic'])

print(f'ADF Statistic: {ADF_result[0]}')
# ADF Statistic: -1.1758885999240771
print(f'p-value: {ADF_result[1]}')
# p-value: 0.683880891789618
```  

`ADF` 통계값이 큰 음수가 아니고, p-value 가 0.05 보다 크므로 귀무가설을 기각하지 못해 현재 원본 데이터는 정상적 시계열이 아님을 확인할 수 있다.  
그러므로 변환(차분)을 적용한다.  

```python
foot_traffic_diff = np.diff(df['foot_traffic'], n=1)

fig, ax = plt.subplots()

ax.plot(foot_traffic_diff)
ax.set_xlabel('Time')
ax.set_ylabel('Average weekly foot traffic (differenced)')

plt.xticks(np.arange(0, 1000, 104), np.arange(2000, 2020, 2))

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-2.png)


차분한 데이터를 도식화 했을 때 이전보다 추세가 제거되고 백색소음처럼 보인다. 
정확한 판단을 위해 다시 차분한 데이터에 대해서 `ADF` 검정을 수행한다.  

```python
ADF_result = adfuller(foot_traffic_diff)

print(f'ADF Statistic: {ADF_result[0]}')
# ADF Statistic: -5.268231347422048
print(f'p-value: {ADF_result[1]}')
# p-value: 6.369317654781179e-06
```  

`ADF` 통계값이 충분히 큰 음수이고, p-value 가 0.05 보다 작으므로 귀무가설을 기각해 현재 차분된 데이터는 정상적 시계열임을 확인할 수 있다. 
이제 차분한 데이터를 바탕으로 `ACF` 도식을 그려 특정 지연 후 유의하지 않는 계수가 있는지 확인한다.  

```python
plot_acf(foot_traffic_diff, lags=20);

plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-3.png)


`ACF` 도식을 보면 자기상관 계수는 지연이 증가함에 따라 점차 감소하는 것을 볼 수 있다. 
하지만 특정 지연 후 계수가 갑작스럽게 유의하지 않게되는 특징은 보이지 않는다. 
이는 이동평균 과정이라고는 할 수 없고 데이터에 자기회귀과정이 있을 가능성을 시사한다. 

이때 `AR` 자기회귀 과정의 차수 `p` 를 식별하기 위해 사용하는 방법은 `PACF`(편자기상관함수) 이다. 

> 편자기상관함수(Partial Autocorrelation Function, PACF)는 시계열 데이터에서 특정 시점의 값과 그 이전 시점의 값들 간의 상관관계를 측정하는 통계적 도구이다. 
> 시계열 내에서 상관관계가 있는 지연된 값들 사이의 영향도를 제거할 떄 해당 값들 간의 상관 관계를 측정한다. 
> 그러므로 편자기상관함수를 도식하면 정상화된 `AR(p)` 프로세스의 차수를 결정할 수 있다. 
> `p` 는 특정 지연 `p` 이후 계수가 유의하지 않는 값이다. 

`PACF` 도식은 `statsmodels` 의 `ArmaProcess` 함수를 통해 그릴 수 있다.  

```python
plot_pacf(foot_traffic_diff, lags=20);

plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-4.png)


`PACF` 도식을 보면 지연 3 이후 유의한 계수가 보이지 않는 것을 확인 할 수 있다. 
그러므로 `p` 는 3이 되고, 모델링할 자기회귀과정은 `AR(3)` 이 된다.  


### Forecasting AR Process
이제 자기회귀과정의 차수 `p` 까지 식별해 `AR(3)` 모델을 구축할 수 있다. 
이전 이동평균과정과 동일하게 먼저 훈련 데이터와 테스트 데이터를 분리한다.  

```python
df_diff = pd.DataFrame({'foot_traffic_diff': foot_traffic_diff})

train = df_diff[:-52]
test = df_diff[-52:]

print(len(train))
# 947
print(len(test))
# 52
``` 

훈련 세트와 테스트 세트에 대해서 원본 시계열과 차분한 시계열을 구분해 도식화하면 아래와 같다.  

```python
fig, (ax1, ax2) = plt.subplots(nrows=2, ncols=1, sharex=True, figsize=(10, 8))

ax1.plot(df['foot_traffic'])
ax1.set_xlabel('Time')
ax1.set_ylabel('Avg. weekly foot traffic')
ax1.axvspan(948, 1000, color='#808080', alpha=0.2)

ax2.plot(df_diff['foot_traffic_diff'])
ax2.set_xlabel('Time')
ax2.set_ylabel('Diff. avg. weekly foot traffic')
ax2.axvspan(947, 999, color='#808080', alpha=0.2)

plt.xticks(np.arange(0, 1000, 104), np.arange(2000, 2020, 2))

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-5.png)


이번에도 이동평균의 예측과 동일하게 `rolling forecast` 방식을 사용한다. 
`AR(p)` 모델도 `MA(q)` 모델과 마찬가지로 계수 `p` 만큼의 수를 최대로 예측할 수 있다. 
이번에도 베이스라인 모델로는 평균과 마지막 값을 사용하고 이를 모두 구현한 `rolling_forecast` 함수는 아래와 같다.  

```python
def rolling_forecast(df: pd.DataFrame, train_len: int, horizon: int, window: int, method: str) -> list:
    
    total_len = train_len + horizon
    end_idx = train_len
    
    if method == 'mean':
        pred_mean = []
        
        for i in range(train_len, total_len, window):
            mean = np.mean(df[:i].values)
            pred_mean.extend(mean for _ in range(window))
            
        return pred_mean

    elif method == 'last':
        pred_last_value = []
        
        for i in range(train_len, total_len, window):
            last_value = df[:i].iloc[-1].values[0]
            pred_last_value.extend(last_value for _ in range(window))
            
        return pred_last_value
    
    elif method == 'AR':
        pred_AR = []
        
        for i in range(train_len, total_len, window):
            model = SARIMAX(df[:i], order=(3,0,0))
            res = model.fit(disp=False)
            predictions = res.get_prediction(0, i + window - 1)
            oos_pred = predictions.predicted_mean.iloc[-window:]
            pred_AR.extend(oos_pred)
            
        return pred_AR
```  

이제 베이스라인 모델과 `AR(3)` 모델을 사용해 예측을 수행한다. 
그리고 그결과는 모두 테스트 세트에 추가한다.  

```python
TRAIN_LEN = len(train)
HORIZON = len(test)
WINDOW = 1

pred_mean = rolling_forecast(df_diff, TRAIN_LEN, HORIZON, WINDOW, 'mean')
pred_last_value = rolling_forecast(df_diff, TRAIN_LEN, HORIZON, WINDOW, 'last')
pred_AR = rolling_forecast(df_diff, TRAIN_LEN, HORIZON, WINDOW, 'AR')

test['pred_mean'] = pred_mean
test['pred_last_value'] = pred_last_value
test['pred_AR'] = pred_AR

test.head()
#       foot_traffic_diff	pred_mean	pred_last_value	    pred_AR
# 947	-0.776601	        0.213270	-1.021893	        -0.719714
# 948	-0.574631	        0.212226	-0.776601	        -0.814547
# 949	-0.890697	        0.211397	-0.574631	        -0.664738
# 950	-0.283552	        0.210237	-0.890697	        -0.641469
# 951	-1.830685	        0.209717	-0.283552	        -0.579279
```  

실제 값과 각 예측값을 모두 도식화하면 아래와 같다.  

```python
fig, ax = plt.subplots()

ax.plot(df_diff['foot_traffic_diff'])
ax.plot(test['foot_traffic_diff'], 'b-', label='actual')
ax.plot(test['pred_mean'], 'g:', label='mean')
ax.plot(test['pred_last_value'], 'r-.', label='last')
ax.plot(test['pred_AR'], 'k--', label='AR(3)')

ax.legend(loc=2)

ax.set_xlabel('Time')
ax.set_ylabel('Diff. avg. weekly foot traffic')

ax.axvspan(947, 998, color='#808080', alpha=0.2)

ax.set_xlim(920, 999)

plt.xticks([936, 988],[2018, 2019])

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-6.png)


`AR(3)` 모델이 베이스라인 모델보다 잘 예측한 것을 보이는데, 
더 정확한 성능 검증을 위해 `MSE` 로 각 예측을 평가한다.  

```python
from sklearn.metrics import mean_squared_error

mse_mean = mean_squared_error(test['foot_traffic_diff'], test['pred_mean'])
mse_last = mean_squared_error(test['foot_traffic_diff'], test['pred_last_value'])
mse_AR = mean_squared_error(test['foot_traffic_diff'], test['pred_AR'])

fig, ax = plt.subplots()

x = ['mean', 'last_value', 'AR(3)']
y = [mse_mean, mse_last, mse_AR]

ax.bar(x, y, width=0.4)
ax.set_xlabel('Methods')
ax.set_ylabel('MSE')
ax.set_ylim(0, 5)

for index, value in enumerate(y):
    plt.text(x=index, y=value+0.25, s=str(round(value, 2)), ha='center')

plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-7.png)


`MSE` 결과를 보면 `AR(3)` 모델이 베이스라인 모델보다 성능적으로 가장 우수한 것을 확인했다. 
지금까지 다룬 값은 차분된 값이기 때문에 이를 원본 데이터 규모로 역변환을 수행한다. 
차분을 적용했기 때문에 역변환은 `cumsum` 누적합을 사용한다.  

```python
df['pred_foot_traffic'] = pd.Series()
df['pred_foot_traffic'][948:] = df['foot_traffic'].iloc[948] + test['pred_AR'].cumsum()

fig, ax = plt.subplots()

ax.plot(df['foot_traffic'])
ax.plot(df['foot_traffic'], 'b-', label='actual')
ax.plot(df['pred_foot_traffic'], 'k--', label='AR(3)')

ax.legend(loc=2)

ax.set_xlabel('Time')
ax.set_ylabel('Average weekly foot traffic')

ax.axvspan(948, 1000, color='#808080', alpha=0.2)

ax.set_xlim(920, 1000)
ax.set_ylim(650, 770)

plt.xticks([936, 988],[2018, 2019])

fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/ar-8.png)


`AR(3)` 모델이 예측한 원본 데이터를 보면 실제 데이터의 추세를 잘 따르는 것을 볼 수 있다. 
평균절대오차(`MAE`) 를 측정해 얼만큼의 절대 오차가 있는지 확인한다.  

```python
from sklearn.metrics import mean_absolute_error

mae_AR_undiff = mean_absolute_error(df['foot_traffic'][948:], df['pred_foot_traffic'][948:])

print(mae_AR_undiff)
# 3.4780335607419093
```  

평균절대오차 값으로 보면 평균적으로 예측과 실제 값이 3.4명 정도 주간 유동인구 차이가 발생한 것을 확인할 수 있다.  




---  
## Reference
[TimeSeriesForecastingInPython](https://github.com/marcopeix/TimeSeriesForecastingInPython)  



