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

```python
import statsmodels.api as sm

macro_econ_data = sm.datasets.macrodata.load_pandas().data
macro_econ_data
#       year	quarter	    realgdp	    realcons	realinv	    realgovt	realdpi	    cpi 	    m1	        tbilrate	unemp	pop	        infl	realint
# 0	    1959.0	1.0	        2710.349	1707.4	    286.898	    470.045	    1886.9	    28.980	    139.7	    2.82	    5.8	    177.146	    0.00	0.00
# 1	    1959.0	2.0	        2778.801	1733.7	    310.859	    481.301	    1919.7	    29.150	    141.7	    3.08	    5.1	    177.830	    2.34	0.74
# 2	    1959.0	3.0	        2775.488	1751.8	    289.226	    491.260	    1916.4	    29.350	    140.5	    3.82	    5.3	    178.657	    2.74	1.09
# 3	    1959.0	4.0	        2785.204	1753.7	    299.356	    484.052	    1931.3	    29.370	    140.0	    4.33	    5.6	    179.386	    0.27	4.06
# 4	    1960.0	1.0	        2847.699	1770.5	    331.722	    462.199	    1955.5	    29.540	    139.6	    3.50	    5.2	    180.007	    2.31	1.19
# ...	...	...	...	...	    ...	...	...	...	...	    ...	...	...	...	...                 
# 198	2008.0	3.0	        13324.600	9267.7	    1990.693	991.551	    9838.3	    216.889	    1474.7	    1.17	    6.0	    305.270	    -3.16	4.33
# 199	2008.0	4.0	        13141.920	9195.3	    1857.661	1007.273	9920.4	    212.174	    1576.5	    0.12	    6.9	    305.952	    -8.79	8.91
# 200	2009.0	1.0	        12925.410	9209.2	    1558.494	996.287	    9926.4	    212.671	    1592.8	    0.22	    8.1	    306.547	    0.94	-0.71
# 201	2009.0	2.0	        12901.504	9189.0	    1456.678	1023.528	10077.5	    214.469	    1653.6	    0.18	    9.2	    307.226	    3.37	-3.19
# 202	2009.0	3.0	        12990.341	9256.0	    1486.398	1044.088	10040.6	    216.385	    1673.9	    0.12	    9.6	    308.013	    3.56	-3.44
# 203 rows × 14 columns
```  

각 변수에 대한 설명은 아래와 같다.  

| 변수       | 설명                                               |
|------------|--------------------------------------------------|
|realgdp    | 실질 국내총생산 (Real Gross Domestic Product), 목표/내생 변수 |
|realcons   | 실질 소비 (Real Consumption)                         |
|realinv    | 실질 투자 (Real Investment)                          |
|realgovt   | 실질 정부 지출 (Real Government Spending)              |
|realdpi    | 실질 가처분 소득 (Real Disposable Personal Income)      |
|cpi        | 소비자 물가 지수 (Consumer Price Index)                 |
|m1         | 통화 공급량 M1 (Money Supply M1)                      |
|tbilrate   | 단기 국채 금리 (3-Month Treasury Bill Rate)            |
|unemp      | 실업률 (Unemployment Rate)                          |
|pop        | 인구 (Population)                                  |
|infl       | 인플레이션율 (Inflation Rate)                          |
|realint    | 실질 이자율 (Real Interest Rate)                      |

테스트에서는 목표 변수 `realgdp`, `realcons`, `realinv`, `realgovt`, `realdpi`, `cpi` 5개 변수를 외생 변수를 사용해 총 6개 변수를 사용한다. 
모든 변수들이 시간에 따라 어떠한 패턴을 보이는지 시각화해보면 아래와 같다. 

```python
fig, axes = plt.subplots(nrows=3, ncols=2, dpi=300, figsize=(11,6))

for i, ax in enumerate(axes.flatten()[:6]):
    data = macro_econ_data[macro_econ_data.columns[i+2]]
    
    ax.plot(data, color='black', linewidth=1)
    ax.set_title(macro_econ_data.columns[i+2])
    ax.xaxis.set_ticks_position('none')
    ax.yaxis.set_ticks_position('none')
    ax.spines['top'].set_alpha(0)
    ax.tick_params(labelsize=6)

plt.setp(axes, xticks=np.arange(0, 208, 8), xticklabels=np.arange(1959, 2010, 2))
fig.autofmt_xdate()
plt.tight_layout()
```  

![그림 1]({{site.baseurl}}/img/datascience/sarimax-1.png)
