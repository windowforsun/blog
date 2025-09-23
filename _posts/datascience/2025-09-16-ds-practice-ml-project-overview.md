--- 
layout: single
classes: wide
title: "[ML] ML Project Overview"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: 'ML 프로젝트를 진행할 때 필요한 전체적인 흐름과 절차 및 구성요소에 대해 알아본다.'
author: "window_for_sun"
header-style: text
categories :
  - Data Science
tags:
    - Practice
    - ML
    - Python
toc: true
use_math: true
---  

## ML Project Overview
`ML` 프로젝트를 진행할 때 필요한 전체적인 흐름과 절차 및 구성요소에 대해 알아본다. 
그 절차와 구성을 간략화 하면 아래와 같다.  

1. 문제 분석
2. 데이터 수집
3. 데이터 탐색 및 시각화
4. `ML` 을 위한 데이터 준비
5. 모델 선택 및 훈련
6. 모델 튜닝
7. 모델 배포

위 전체 흐름에 대해 알아보고 데모 프로젝트를 진행해 보기위해 
주택 가격 데이터 셋을 사용해 특성이 주어졌을 때 주택 가격을 예측하는 `ML` 프로젝트를 진행해 본다.  


### 문제 분석
주택 가격 데이터 셋을 사용해 특성이 주어졌을 때 주택 가격을 예측하는 문제를 해결해야 한다. 
주요 특성으로는 인구, 중간 소득, 중간 주택 가격 등이 있기 때문에 이러한 특성을 사용해 최종적으로 중간 주택 가격 모델을 학습하고 예측해야 한다.  

이 프로젝트의 주된 목적은 기존에 주택 전문가가 직접 데이터를 보고 하나씩 예측하던 시간이 많이 소요되는 프로세스를 개선하는 것이다. 
그리고 이를 통해 예측의 정확도를 높이고, 비용을 절감하고, 시간을 단축하는 것이다.  

이를 구현하기 위해 먼저 모델 훈련을 어떤 지도 학습 방식을 시용해야하는지 결정해야 한다. 
학습 방식에는 지도 학습, 비지도 학습, 준지도 학습, 자기 지도 학습, 강화 학습이 있다. 
그리고 분류와 회귀인지 배치 학습인지 온라인 학습인지 결정해야 한다.  

인구, 중간 소득 등과 같은 특성 데이터가 있고, 레이블된 중간 주택 가격으로 구성된 훈련 데이터 셋을 사용한다고 하면 이는 `지도 학습`이다. 
그리고 모델이 값을 예측해야 하기 때문에 `회귀 문제`이다. 
좀 더 자세히 말하면 특성 여러개를 사용해야 하기 대문에 `다중 회귀` 문제이다. 
또한 예측할 값이 한개이므로 `단변량 회귀` 문제이다. 
그리고 주어진 데이터 셋을 사용해서만 학습하고 빠르게 변화하는 데이터를 실시간 반영할 필요가 없기 때문에 배치 학습을 사용한다.  

`회귀 문제` 의 성능 측정 지표는 일반적으로 `평균제곱근 오차(RMSE)` 이므로 이를 사용한다. 
`RMSE` 는 오차가 커질수록 해당 값도 커지게 된다. 
그러므로 모델 훈련 후 해당 값이 얼마나 큰지 확인해 오차를 측정할 수 있다. 
그리고 필요에 따라 `평균 절대 오차(MAE)` 를 사용할 수도 있다.  

### 데이터 수집
주택 가격 데이터는 `SatatLib` 에서 제공하는 `California Housing Prices` 데이터 셋을 사용한다. 
데이터는 아래 코드를 통해 다운 받을 수 있다.  

```python
def load_housing_data():
    housing_file_path = Path("datasets/housing.tgz")
    if not housing_file_path.is_file():
        Path("datasets").mkdir(parents=True, exist_ok=True)
        url = "https://github.com/ageron/data/raw/main/housing.tgz"
        urllib.request.urlretrieve(url, housing_file_path)
        with tarfile.open(housing_file_path) as housing_file:
            housing_file.extractall(path="datasets")

    return pd.read_csv(Path("datasets/housing/housing.csv"))

housing_origin = load_housing_data();
```

데이터를 수집했으면 해당 데이터가 어떠한 필드(특성)를 가지고 있는지 구조가 어떤지 확인해야 한다.  

```python
housing_origin.head()

'''
   longitude  latitude  housing_median_age  total_rooms  total_bedrooms
0    -122.23     37.88                41.0        880.0           129.0
1    -122.22     37.86                21.0       7099.0          1106.0
2    -122.24     37.85                52.0       1467.0           190.0
3    -122.25     37.85                52.0       1274.0           235.0
4    -122.25     37.85                52.0       1627.0           280.0

   population  households  median_income  median_house_value ocean_proximity
0       322.0       126.0         8.3252            452600.0        NEAR BAY
1      2401.0      1138.0         8.3014            358500.0        NEAR BAY
2       496.0       177.0         7.2574            352100.0        NEAR BAY
3       558.0       219.0         5.6431            341300.0        NEAR BAY
4       565.0       259.0         3.8462            342200.0        NEAR BAY
'''
```  

총 10개의 특성이 있으며, `ocean_proximity` 는 범주형 특성이다.
숫자가 아닌 범주형 특성을 다루는 것은 이후에 다시 알아본다.
`info()` 메서드를 사용해 데이터의 간략한 정보를 확인할 수 있다.  

```python
housing_origin.info()

'''
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 20640 entries, 0 to 20639
Data columns (total 10 columns):
#   Column              Non-Null Count  Dtype  
---  ------              --------------  -----
0   longitude           20640 non-null  float64
1   latitude            20640 non-null  float64
2   housing_median_age  20640 non-null  float64
3   total_rooms         20640 non-null  float64
4   total_bedrooms      20433 non-null  float64
5   population          20640 non-null  float64
6   households          20640 non-null  float64
7   median_income       20640 non-null  float64
8   median_house_value  20640 non-null  float64
9   ocean_proximity     20640 non-null  object
dtypes: float64(9), object(1)
memory usage: 1.6+ MB
'''
```  

위 결과를 통해 `total_bedrooms` 에는 누락된 값이 몇 있는 것을 확인할 수 있다. 
`ML` 에서는 값이 누락된 경우를 처리해야 하기 때문에 이를 처리하는 방법은 이후에 알아본다.  
숫자형이 아닌 `ocean_proximity` 는 범주형 특성이므로 어떤 카테고리가 있고 각 카테고리에 얼마나 많은 구역이 있는지 확인하면 아래와 같다. 

```python
housing_origin["ocean_proximity"].value_counts()

'''
ocean_proximity
<1H OCEAN     9136
INLAND        6551
NEAR OCEAN    2658
NEAR BAY      2290
ISLAND           5
Name: count, dtype: int64
'''
```  

그리고 `describe()` 를 사용해 숫자형 특성의 요약 정보를 확인할 수 있다.  

```python
housing_origin.describe()

'''
          longitude      latitude  housing_median_age   total_rooms  \
count  20640.000000  20640.000000        20640.000000  20640.000000   
mean    -119.569704     35.631861           28.639486   2635.763081   
std        2.003532      2.135952           12.585558   2181.615252   
min     -124.350000     32.540000            1.000000      2.000000   
25%     -121.800000     33.930000           18.000000   1447.750000   
50%     -118.490000     34.260000           29.000000   2127.000000   
75%     -118.010000     37.710000           37.000000   3148.000000   
max     -114.310000     41.950000           52.000000  39320.000000   

       total_bedrooms    population    households  median_income  \
count    20433.000000  20640.000000  20640.000000   20640.000000   
mean       537.870553   1425.476744    499.539680       3.870671   
std        421.385070   1132.462122    382.329753       1.899822   
min          1.000000      3.000000      1.000000       0.499900   
25%        296.000000    787.000000    280.000000       2.563400   
50%        435.000000   1166.000000    409.000000       3.534800   
75%        647.000000   1725.000000    605.000000       4.743250   
max       6445.000000  35682.000000   6082.000000      15.000100   

       median_house_value  
count        20640.000000  
mean        206855.816909  
std         115395.615874  
min          14999.000000  
25%         119600.000000  
50%         179700.000000  
75%         264725.000000  
max         500001.000000  
'''
```  

`std` 는 값이 퍼져있는 정도를 의미하는 표준 편차이다. 
`std` 가 크다는 것은 데이터가 평균값으로부터 퍼져있고, 데이터 간의 차이가 크다는 것을 의미한다. 
`25%`, `50%`, `75%` 는 백분위수를 의미하며, 백분위수는 주어진 값들 중에서 주어진 백분율이하의 값이 되는 위치의 값을 의미한다.  

데이터 형태를 빠르게 확인하는 방법중 하나는 숫자형 특성을 히스토그램으로 그려보는 것이다. 
수평축은 해당 특성 값의 범위를 의미하고, 수직축은 해당 범위에 속한 수를 의미한다.  

```python
housing_origin.hist(bins=50, figsize=(12, 8))
plt.show()
```  

![ml-project-overview-1.png]({{site.baseurl}}/img/datascience/ml-project-overview-1.png)

데이터 수집에서 마지막으로 남은 절차는 테스트 세트를 만드는 것이다. 
이는 전체 데이터 셋에서 일부 데이터를 떼어내어 모델을 훈련시킨 후 테스트하는 데 사용한다. 
만약 전체 데이터를 훈련에 그대로 사용한다면 모델이 훈련 데이터에 과적합되어 새로운 데이터에 대해 잘 일반화되지 않을 수 있다. 
이 상태 그대로 모델을 훈련하고 일반화 오차를 추정하면 낙관적인 추정만 될 뿐 실제 모델의 성능을 잘 반영하지 않을 수 있다. 
이러한 현상을 `데이터 스누핑(data snooping) 편향` 이라고 한다.  

테스트 세트를 구성하는 것은 간단하다. 
랜덤으로 어떤 샘플을 선택해서 데이터 셋의 20% 정도를 테스트 세트로 사용하면 된다.
이를 위해 `train_test_split` 함수를 사용해 데이터 셋을 무작위로 나누어 훈련 세트와 테스트 세트로 만든다.

```python
# 랜덤 샘플링
train_set, test_set = train_test_split(housing_origin, test_size=0.2, random_state=42)
```  

위 방식은 전형적인 랜덤 샘플링 방식이다. 
데이터셋이 충분히 큰 경우에는 일반적으론 해당 방법으로도 괜찮지만, 
데이터셋이 크지 않다면 샘플링 편향이 발생할 수 있다.  

샘플링 편향이란 테스트 세트가 전체 데이터셋을 대표하지 못하는 경우를 의미한다. 
이를 방지하기 위해 `계층적 샘플링`을 사용할 수 있다. 
이는 전체 데이터셋을 여러 계층으로 나누고, 각 계층에서 올바른 수의 샘플을 추출하는 것이다. 
주택 가격의 경우 중간 소득이 중간 주택 가격을 예측하는 데 매우 중요한 특성이라고 가정해보자. 
중간 소득이 이를 카테고리형 특성으로 변환하고 계층을 나눠 각 계층 비율에 맞는 테스트 세트를 추출한다.  

```python

housing_with_income_cat = housing_origin.copy()
housing_with_income_cat["income_cat"] = pd.cut(housing_with_income_cat["median_income"],
                                 bins=[0., 1.5, 3.0, 4.5, 6., np.inf],
                                 labels=[1, 2, 3, 4, 5])
print('data set income category ratio')
print(housing_with_income_cat["income_cat"].value_counts() / len(housing_with_income_cat))

'''
income_cat
3    0.350581
2    0.318847
4    0.176308
5    0.114438
1    0.039826
Name: count, dtype: float64
'''

strat_train_set, strat_test_set = train_test_split(housing_with_income_cat,
                                                   test_size=0.2,
                                                   stratify=housing_with_income_cat["income_cat"],
                                                   random_state=42)

print('test set income category ratio')
print(strat_test_set["income_cat"].value_counts() / len(strat_test_set))

'''
income_cat
3    0.350533
2    0.318798
4    0.176357
5    0.114341
1    0.039971
Name: count, dtype: float64
'''
```  

`계층적 샘플링` 을 위해 만들었던 `income_cat` 은 이후에 사용하지 않기 때문에 아래 코드로 삭제해 준다.  

```python
for set_ in (strat_train_set, strat_test_set):
    set_.drop("income_cat", axis=1, inplace=True)
```  

### 데이터 탐색 및 시각화
주택 가격은 지리적인 위치에 따라 가격 편차가 크게 발생할 수 있다. 
이를 확인하기 위해 지리적 데이터를 시각화 해본다. 
위도와 경도를 그리고 알파(투명도)를 사용해 산점도 그래프를 그려보면 아래와 같다.  

```python
housing = strat_train_set.copy()
housing.plot(kind="scatter", x="longitude", y="latitude", alpha=0.2)
plt.xlabel('lng')
plt.ylabel('lat')
plt.show()
```

![ml-project-overview-2.png]({{site.baseurl}}/img/datascience/ml-project-overview-2.png)

위 그래프를 통해 주택 가격이 밀집된 지역을 확인할 수 있다. 
좀 더 데이터를 명확히 보기위해 반지름은 구역의 인구수 그리고, 
색상은 가격을 나타내는 산점도 그래프를 그려보면 아래와 같다.  

```python
housing.plot(kind="scatter", x="longitude", y="latitude", grid=True,
             # 원 반지름
             s=housing["population"] / 100, label="population",
             # 원 색상
             c="median_house_value", cmap="jet", colorbar=True,
             legend=True, figsize=(10, 7))
cax = plt.gcf().get_axes()[1]
cax.set_ylabel('median_house_value')
plt.xlabel('lng')
plt.ylabel('lat')
plt.show()
```  

![ml-project-overview-3.png]({{site.baseurl}}/img/datascience/ml-project-overview-3.png)

위 그래프를 통해 인구 밀집 지역 및 주택 위치에 따라 주택 가격이 큰 영향을 받는 다는 것을 시각적으로 확인 할 수 있다.  

다음으로 주택 가격에 따른 각 특성들의 상관관계에 대해 알아본다. 
이는 `표준 상관계수(pearson)` 을 사용하므로 `corr()` 을 통해 쉽게 구할 수 있다.  

```python
corr_matrix = housing.corr(numeric_only=True)
corr_matrix["median_house_value"].sort_values(ascending=False)

'''
median_house_value    1.000000
median_income         0.688380
total_rooms           0.137455
housing_median_age    0.102175
households            0.071426
total_bedrooms        0.054635
population           -0.020153
longitude            -0.050859
latitude             -0.139584
Name: median_house_value, dtype: float64
'''
```  

표준 상관계수는 `+1 ~ -1` 사이의 값을 가지며, `+1` 이면 강한 양의 상관관계, `-1` 이면 강한 음의 상관관계를 의미한다. 
`0` 이면 상관관계가 없다는 것을 의미한다. 
결과를 보면 `median_income` 이 `median_house_value` 와 가장 강한 양의 상관관계를 가지고 있음을 알 수 있고, 
`latitude` 가 가장 강한 음의 상관관계를 가지고 있음을 알 수 있다.  

각 특성 사이의 상관관계를 확인하는 다른 방법은 `pandas` 의 `scatter_matrix` 를 사용해 숫자형 특성 간 산점도를 그려보는 것이다. 
예측하려는 주택 가격과 관계가 있어보이는 주요 특성간 상관관계 산점도를 그려보면 아래와 같다.   

```python
attributes = ["median_house_value", "median_income", "total_rooms", "housing_median_age"]
scatter_matrix(housing[attributes], figsize=(12, 8))
plt.show()
```  

![ml-project-overview-4.png]({{site.baseurl}}/img/datascience/ml-project-overview-4.png)

위 그래프에서 `median_income` 그래프를 보면 `median_house_value` 와 상관관계가 매우 강하다는 것을 다시 확인할 수 있다. 
그래프로 우상향을 띄면서 너무 많이 퍼져 있지도 않다.  

마지막으로 가장 유용한 특성을 찾아보기 위해 특성 조합으로 새로운 특성을 만들어 상관관계를 살펴본다. 
현재 데이터셋의 특성 중 `total_rooms`, `total_bedrooms`, `population`, `households` 는 의미가 없어 보이지만,
이들을 조합해 새로운 특성을 만들어보면 의미 있는 특성을 만들 수 있다.  

```python
housing["rooms_per_house"] = housing["total_rooms"] / housing["households"]
housing["bedrooms_ratio"] = housing["total_bedrooms"] / housing["total_rooms"]
housing["population_per_house"] = housing["population"] / housing["households"]

corr_matrix = housing.corr(numeric_only=True)
print(corr_matrix["median_house_value"].sort_values(ascending=False))

'''
median_house_value      1.000000
median_income           0.688380
rooms_per_house         0.143663
total_rooms             0.137455
housing_median_age      0.102175
households              0.071426
total_bedrooms          0.054635
population             -0.020153
population_per_house   -0.038224
longitude              -0.050859
latitude               -0.139584
bedrooms_ratio         -0.256397
Name: median_house_value, dtype: float64
'''
```  

새로 추가한 조합 특성 `rooms_per_house`, `bedrooms_ratio`, `population_per_house` 는 기존 단일 특성에 비해 `median_house_value` 와의 상관관계가 더 높은 것을 확인할 수 있다. 
그 중에서도 `bedrooms_ratio` 는 가장 강한 음의 상관관계를 가지고 있음을 알 수 있다.  


### ML 을 위한 데이터 준비
머신러닝 알고리즘을 위한 데이터 준비는 자동화하는 것이 좋다. 
그 이유를 나열하면 아래와 같다.  

- 어떤 데이터셋에 대해서도 데이터 변환을 손쉽게 반복할 수 있다.
- 향후 프로젝트에 사용할 수 있는 변환 라이브러리를 구축한다.
- 실제 시스템에서 알고리즘에 새 데이터를 주입하기 전에 변환을 쉽게 시도할 수 있다.
- 여러 가지 데이터 변환을 쉽게 시도해볼 수 있고 어떤 조합이 가장 좋은지 확인할 수 있다.

먼저 훈련 세트와 레이블을 분리해야 한다. 
기존 훈련 세트에서 `median_house_value` 를 제거한 데이터와 `median_house_value` 만 가진 `label` 데이터로 분리한다.    

```python
housing = strat_train_set.drop('median_house_value', axis=1)
print(housing.info())

'''
<class 'pandas.core.frame.DataFrame'>
Index: 16512 entries, 13096 to 19888
Data columns (total 9 columns):
 #   Column              Non-Null Count  Dtype  
---  ------              --------------  -----  
 0   longitude           16512 non-null  float64
 1   latitude            16512 non-null  float64
 2   housing_median_age  16512 non-null  float64
 3   total_rooms         16512 non-null  float64
 4   total_bedrooms      16344 non-null  float64
 5   population          16512 non-null  float64
 6   households          16512 non-null  float64
 7   median_income       16512 non-null  float64
 8   ocean_proximity     16512 non-null  object 
dtypes: float64(8), object(1)
memory usage: 1.3+ MB
None
'''

housing_labels = strat_train_set['median_house_value'].copy()
print(housing_labels.info())

'''
<class 'pandas.core.series.Series'>
Index: 16512 entries, 13096 to 19888
Series name: median_house_value
Non-Null Count  Dtype  
--------------  -----  
16512 non-null  float64
dtypes: float64(1)
memory usage: 258.0 KB
None
'''
```  

#### 데이터 정제
데이터 정제에서는 누락된 값에 대한 처리를 해야 한다. 
데이터셋 특성 중 `total_bedrooms` 특성은 값이 없는 경우가 있는데, `ML` 알고리즘은 누락된 값에 대해 처리하지 못하기 때문에 정제가 필요하다. 
대표적인 정제 방법은 아래 3가지가 있다.  

1. 해당 구역 제거(행 삭제) `housing.dropna(subset=["total_bedrooms"], inplace=True)`
2. 해당 특성 제거(열 삭제) `housing.drop("total_bedrooms", axis=1, inplace=True)`
3. 어떤 값으로 대체(0, 평균, 중간값 등) `median = housing["total_bedrooms"].median() housing["total_bedrooms"].fillna(median, inplace=True)`

데이터를 최대한 유지하는 것이 중요하기 때문에 3번 방법을 사용해 누락된 값에 중간값을 대체한다. 
매번 특정 특성의 중간값을 계산해야 하므로 이를 자동화하기 위해 사이킷런에서 제공하는 `SimpleImputer` 를 사용한다. 
이는 각 특성의 중간값을 저장하고 훈련 세트 이후 검증 세트, 테스트 세트 그리고 새로 주입될 새로운 데이터에도 누락값을 대체할 수 있다.  

```python
imputer = SimpleImputer(strategy='median')
housing_num = housing.select_dtypes(include=[np.number])
imputer.fit(housing_num)
print(imputer.statistics_)

'''
[-118.51     34.26     29.     2129.     435.     1166.     409.       3.5348]
'''

print(housing_num.median().values)

'''
[-118.51     34.26     29.     2129.     435.     1166.     409.       3.5348]
'''

housing_tr = imputer.transform(housing_num)
housing_tr = pd.DataFrame(housing_tr, columns=housing_num.columns, index=housing_num.index)
print(housing_tr.info())

'''
<class 'pandas.core.frame.DataFrame'>
Index: 16512 entries, 13096 to 19888
Data columns (total 8 columns):
 #   Column              Non-Null Count  Dtype  
---  ------              --------------  -----  
 0   longitude           16512 non-null  float64
 1   latitude            16512 non-null  float64
 2   housing_median_age  16512 non-null  float64
 3   total_rooms         16512 non-null  float64
 4   total_bedrooms      16512 non-null  float64
 5   population          16512 non-null  float64
 6   households          16512 non-null  float64
 7   median_income       16512 non-null  float64
dtypes: float64(8)
memory usage: 1.1 MB
None
'''
```  

계산된 각 특성의 중간값은 `imputer.statistics_` 에 저장되어 있으며, 이를 사용해 훈련 세트의 누락된 값을 대체할 수 있다.  


#### 텍스트와 범주형 특성 다루기
데이터셋에 숫자형이 아닌 데이터는 숫자형으로 변환하는 것이 필요하다. 
먼저 다를 데이터셋에서 숫자형이 아닌 `ocean_proximity` 를 확인해보면 아래와 같다.  

```python
housing_cat = housing[['ocean_proximity']]

cat_head = housing_cat.head(8)

'''
ocean_proximity
13096        NEAR BAY
14973       <1H OCEAN
3785           INLAND
14689          INLAND
20507      NEAR OCEAN
1286           INLAND
18078       <1H OCEAN
4396         NEAR BAY
'''
```  

위와 같은 텍스트 범주형을 숫자로 변환하는 방법중 하나는 `OrdinalEncoder` 를 사용하는 것이다. 
하지만 `OrdinalEncoder` 는 순서가 있는 범주형 데이터에 적합한데, 
가장 대표적인 예가 바로 `bad`, `average`, `good`, `excellent` 와 같은 데이터이다. 
하지만 `ocean_proximity` 의 경우 순서가 없는 범주형 데이터이기 때문에 `OneHotEncoder` 를 사용해야 한다. 
`OneHotEncoder` 는 각 범주를 별도의 이진 특성으로 변환하며, 이를 `원-핫 인코딩(One-Hot Encoding)` 이라고 한다.  

```python
cat_encoder = OneHotEncoder(handle_unknown="ignore")
# or cat_encoder.handle_unknown = "ignore"

housing_cat_1hot = cat_encoder.fit_transform(housing_cat)
cat_1_hot_head = housing_cat_1hot.toarray()[:8]

'''
[[0. 0. 0. 1. 0.]
 [1. 0. 0. 0. 0.]
 [0. 1. 0. 0. 0.]
 [0. 1. 0. 0. 0.]
 [0. 0. 0. 0. 1.]
 [0. 1. 0. 0. 0.]
 [1. 0. 0. 0. 0.]
 [0. 0. 0. 1. 0.]]
'''
```  

위 결과를 통해 `NEAR BAY` 는 `[0. 0. 0. 1. 0.]` 인 회소행렬로 변환되었음을 확인할 수 있다. 
희소행렬은 대부분 0으로 이루어진 행렬을 의미하며, 이러한 행렬은 메모리를 효율적으로 사용할 수 있다. 
내부적으로는 0이 아닌 값과 그 위치만 저장해 범주형 특성을 숫자형으로 다룰 수 있다.  

만약 카테고리 특성이 많은 경우 `원-핫 인코딩` 을 사용하면 특성의 차원이 매우 커질 수 있다. 
이는 훈련을 느리게 할 뿐만 아니라 성능을 저하시킬 수 있다. 
이를 해결하기 위해 `임베딩(Embedding)` 이라는 방법을 사용할 수 있다. 
임베딩은 각 범주형 특성을 학습 가능한 저차원 벡터로 변환하는 방법이다. 
이를 통해 차원을 줄이고 효율적으로 훈련할 수 있다. 
대표적인 방법으로는 해안까지의 거리등으로 나타내어 범주형 특성을 숫자형으로 변환하는 것이다.  

`OneHotEncodder` 에서 각 열 이름은 `feature_names_in_` 에 저장되어 있으며, `cat_encoder.get_feature_names_out()` 을 사용해 각 열 이름을 확인할 수 있다.  


#### 특성 스케일링
머신러닝 알고리즘은 입력 숫자 특성들의 스케일이 다르면 잘 작동하지 않는다. 
다룰 데이터셋에서는 전체 방 개수와 중간 소득 특성의 스케일이 크게 다른 것을 확인할 수 있다.  

```python
room_min = housing['total_rooms'].min()
# 2.0
room_max = housing['total_rooms'].max()
# 39320.0
income_min = housing['median_income'].min()
# 0.4999
income_max = housing['median_income'].max()
# 15.0001
```  

이렇게 특성이 다를 경우 이를 그대로 모델에 사용하면 모델은 중간 소득 특성을 무시하고 방 개수에 더 초점을 맞추게 된다. 
모든 특성의 범위를 같게 만들어주는 방법 중 가장 대표적인 방법이 바로 `min-max 스케일링` 이다. 
이는 각 특성에 데이터에 최솟값을 빼고 최댓값과 최솟값의 차이로 나누어 `0~1` 범위에 들도록 감을 이동해 스케일을 조정하는 방식이다. 
사이킷 런에서는 `MinMaxScaler` 를 사용해 이를 쉽게 구현할 수 있고, 일반적으로는 `-1 ~ 1` 의 범위로 스케일링을 진행한다.  

```python
min_max_scaler = MinMaxScaler(feature_range=(-1, 1))
housing_num_min_max_scaled = min_max_scaler.fit_transform(housing_num)
housing_num_min_max_scaled_head = housing_num_min_max_scaled[:8]

'''
[[-0.60851927  0.11702128  1.         -0.83117147 -0.64116605 -0.80701754
  -0.61433638 -0.7794789 ]
 [ 0.21095335 -0.66170213  0.52941176 -0.90014752 -0.88629409 -0.91866029
  -0.86708979 -0.22929339]
 [-0.51926978  0.23617021  0.25490196 -0.94501246 -0.93042358 -0.93141946
  -0.92458466 -0.73336919]
 [ 0.46855984 -0.74468085 -0.37254902 -0.78778168 -0.7262039  -0.77401546
  -0.70916558 -0.75698266]
 [ 0.25760649 -0.74042553  0.37254902 -0.77801516 -0.6102432  -0.76579561
  -0.56281501 -0.58217128]
 [-0.38336714  0.15106383  1.         -0.90706547 -0.90336608 -0.91522513
  -0.88127683 -0.60735714]
 [ 0.21501014 -0.72340426  0.29411765 -0.94485986 -0.93686584 -0.93792173
  -0.9413851  -0.22574861]
 [-0.54969574  0.03404255  0.37254902 -0.75660003 -0.71042036 -0.7502147
  -0.66809782 -0.32326451]]
'''
```

다른 방법은 표준화를 사용하는 방법이 있다. 
데이터에 평균을 뺀 후 표준편차로 나누어 데이터의 분산이 1이 되도록 만드는 방법이다. 
`MinMaxScaler` 와 달리 표준화는 특정 범위로 값을 제한하지 않아, 이상치에 영향을 덜 받는다. 
사이킷런에서는 `StandardScaler` 를 사용해 표준화를 쉽게 구현할 수 있다. 

```python
std_scaler = StandardScaler()
housing_num_std_scaler = std_scaler.fit_transform(housing_num)
housing_num_std_scaler_head = housing_num_std_scaler[:8]

'''
[[-1.42303652  1.0136059   1.86111875  0.31191221  1.35909429  0.13746004
   1.39481249 -0.93649149]
 [ 0.59639445 -0.702103    0.90762971 -0.30861991 -0.43635598 -0.69377062
  -0.37348471  1.17194198]
 [-1.2030985   1.27611874  0.35142777 -0.71224036 -0.75958421 -0.78876841
  -0.77572662 -0.75978881]
 [ 1.23121557 -0.88492444 -0.91989094  0.70226169  0.73623112  0.38317548
   0.73137454 -0.85028088]
 [ 0.71136206 -0.87554898  0.58980003  0.79012465  1.58558998  0.44437597
   1.75526303 -0.18036472]
 [-0.86819286  1.08860957  1.86111875 -0.37085617 -0.56140048 -0.66819429
  -0.47273921 -0.27688255]
 [ 0.60639163 -0.83804715  0.43088519 -0.7108675  -0.80677082 -0.83718074
  -0.89326484  1.18552636]
 [-1.27807737  0.83078446  0.58980003  0.98278248  0.8518383   0.56038289
   1.01869019  0.81182372]]
'''
```  

이후 숫자형 특성에 대한 스케일링은 `StandardScaler` 를 사용해 진행한다. 
스케일랑과 데이터 전처리는 훈련 데이터에 한해서만 이뤄져야 한다는 것을 잊으면 안된다.  

#### 사용자 정의 변환기
사이킷런은 다양한 변환기를 제공하지만, 특별한 데이터 처리 작업을 위해 사용자 정의 변환기를 만들어야 할 때가 있다. 
별도의 훈련이 필요없는 변환의 경우 `FunctionTransformer` 를 사용할 수 있고, 
별도의 훈련이 필요한 변환의 경우 `BaseEstimator` 를 상속받아 `fit()`, `transform()` 메서드를 구현할 수 있다.  

먼저 `FunctionalTransformer` 를 사용하는 몇 예시는 아래와 같다. 
만야 특성의 분포를 차트로 보았을 때 꼬리부분이 두꺼운 경우 로그 변환을 사용해 정규분포에 가깝게 만들 수 있다. 
아래는 `population` 특성에 대한 로그 변환을 진행한 예시이다.  

```python
log_transformer = FunctionTransformer(np.log, inverse_func=np.exp)
log_pop = log_transformer.transform(housing[['population']])
log_pop_head = log_pop[:8]

'''
       population
13096    7.362645
14973    6.501290
3785     6.331502
14689    7.520235
20507    7.555905
1286     6.542472
18078    6.232448
4396     7.620215
'''
```  

그리고 `FunctionTransformer` 를 사용하면 여러 특성값을 함께 계산해야 할 떄도 유용하다. 
아래는 첫 번째 입력 특성과 두 번째 특성 사이의 비율을 계산하는 예시이다. 

```python
def column_ratio(X):
    return X[:, 0] / X[:, 1]

ratio_transformer = FunctionTransformer(column_ratio)
# or FunctionTransformer(lambda X: X[:, 0] / X[:, 1])
ratio_result = ratio_transformer.transform(np.array([[1., 2.], [3., 4.]]))

'''
[0.5  0.75]
'''
```  

다음은 `fit()` 메서드를 사용해서 특정 파라미터를 학습하고 이후에 `transform()` 메서드를 통해 훈련 가능한 변환기를 만드는 방법에 대해 알아본다. 
`BaseEstimator` 와 `TransformerMixin` 을 상속받고, `fit()`, `transform()` 메서드를 구현하면 된다. 
아래는 `k-평균` 와 `RBF` 유사도 사용해 지리적 좌표를 클러스터링하는 예시이다.   
n_clusters 는 클러스터의 개수를 의미하며, `fit()` 메서드에서 클러스터 중심을 계산하고, `transform()` 메서드에서 각 샘플의 거리를 계산한다.  

```python
class ClusterSimilarity(BaseEstimator, TransformerMixin):
    def __init__(self, n_clusters=10, gamma=1.0, random_state=None):
        self.n_clusters = n_clusters
        self.gamma = gamma
        self.random_state = random_state

    def fit(self, X, y=None, sample_weight=None):
        self.kmeans_ = KMeans(self.n_clusters, random_state=self.random_state)
        self.kmeans_.fit(X, sample_weight=sample_weight)

        return self

    def transform(self, X):
        return rbf_kernel(X, self.kmeans_.cluster_centers_, gamma=self.gamma)

    def get_feature_names_out(self, names=None):
        return [f"클러스터 {i} 유사도" for i in range(self.n_clusters)]


cluster_simil = ClusterSimilarity(n_clusters=10, gamma=1.0, random_state=42)
similarities = cluster_simil.fit_transform(housing[['latitude', 'longitude']], sample_weight=housing_labels)
similarities_head = similarities[:8].round(2)

'''
[[0.   0.98 0.   0.   0.   0.   0.13 0.55 0.   0.56]
 [0.64 0.   0.11 0.04 0.   0.   0.   0.   0.99 0.  ]
 [0.   0.65 0.   0.   0.01 0.   0.49 0.59 0.   0.28]
 [0.63 0.   0.   0.52 0.   0.   0.   0.   0.2  0.  ]
 [0.87 0.   0.03 0.14 0.   0.   0.   0.   0.89 0.  ]
 [0.   0.38 0.   0.   0.   0.01 0.76 0.11 0.   0.41]
 [0.72 0.   0.07 0.08 0.   0.   0.   0.   0.96 0.  ]
 [0.   0.85 0.   0.   0.   0.   0.13 0.21 0.   0.92]]
'''
```  

`k-평균` 을 사용해 클러스터 구역을 찾고 각 구역과 10개의 클러스터 중심 사이의 `RBF` 유사도를 측정한 결과이다. 
그 결과를 시각적으로 표현하면 아래와 같다.  

![ml-project-overview-5.png]({{site.baseurl}}/img/datascience/ml-project-overview-5.png) 

#### 변환 파이프라인
앞서 머신러닝 모델을 훈련시키기 위해 데이터를 변환하는 몇가지 과정을 살펴보았다. 
이러한 변환 단계는 올바른 순서대로 실행돼야 하는데, 사이킷런에서 이를 도와주는 것이 바로 `Pipeline` 이다. 
`Pipeline` 을 생성하는 방법은 아래 2가지 방법이 있다.  

1. `Pipeline` 생성자를 사용해 연속적인 단계를 이름과 추정기 쌍으로 나열하는 방법

```python
num_pipeline = Pipeline(["impute", SimpleImputer(strategy="median"), "std_scaler", StandardScaler()])
```

2. `make_pipeline()` 을 사용해 추정기만 나열하고 이름은 자동으로 생성하는 방법(e.g. `)

```python
num_pipeline_2 = make_pipeline(SimpleImputer(strategy="median"), StandardScaler())
```  

`Pipeline` 은 나열된 단계를 차례대로 실행하며, 한 단계의 출력을 다음 단계의 입력으로 전달한다. 
만약 `fit()` 을 호출하면 마지막을 제외한 단계에서는 `fit_transform()` 메서드를 호출하고 마지막 단계는 `fit()` 메서드만 호출한다. 
그리고 파이프라인은 마지막 단계에서 제공하는 메서드를 제공한다. 
마지막 단계가 변환기라면 `transform()` 메서드를 제공하고, 예측기라면 `predict()` 메서드를 제공한다.  

파이프라인으로 변환한 데이터셋을 다시 데이터 프레임으로 재구성하려면 아래와 같이. `get_feature_names_out()` 메서드를 사용해 구성할 수 있다.  

```python
num_pipeline_named_steps = num_pipeline_2.named_steps

'''
{'simpleimputer': SimpleImputer(strategy='median'), 'standardscaler': StandardScaler()}
'''

housing_num_prepared = num_pipeline_2.fit_transform(housing_num)
df_housing_num_prepared = pd.DataFrame(housing_num_prepared, columns=num_pipeline_2.get_feature_names_out(), index=housing_num.index)
print(df_housing_num_prepared.head())

'''
       longitude  latitude  housing_median_age  total_rooms  total_bedrooms  \
13096  -1.423037  1.013606            1.861119     0.311912        1.368167   
14973   0.596394 -0.702103            0.907630    -0.308620       -0.435925   
3785   -1.203098  1.276119            0.351428    -0.712240       -0.760709   
14689   1.231216 -0.884924           -0.919891     0.702262        0.742306   
20507   0.711362 -0.875549            0.589800     0.790125        1.595753   

       population  households  median_income  
13096    0.137460    1.394812      -0.936491  
14973   -0.693771   -0.373485       1.171942  
3785    -0.788768   -0.775727      -0.759789  
14689    0.383175    0.731375      -0.850281  
20507    0.444376    1.755263      -0.180365  
'''
```  

파이프라인은 `num_pipeline[1]` 과 같이 인덱스로 접근할 수 있으며, `named_steps` 속성을 사용해 `num_pipeline['simpleimputer']` 이름으로 접근할 수 있다.  

다음은 `ColumnTransformer` 를 하나의 변환기를 사용해 숫자형 특성과 범주형 특성을 함께 변환하는 방법에 대해 알아본다. 
`ColumnTransformer` 을 사용하는 몇가지 방법을 소개하면 아래와 같다.    

1. `ColumnTransformer` 생성자를 사용해 이름, 변환기, 변환할 열 이름을 나열하는 방법

```python
num_attributes = ["longitude", "latitude", "housing_median_age", "total_rooms", "total_bedrooms", "population", "households", "median_income"]
cat_attributes = ["ocean_proximity"]

cat_pipeline = make_pipeline(
    SimpleImputer(strategy="most_frequent"),
    OneHotEncoder(handle_unknown="ignore"))

preprocessing = ColumnTransformer([
    ("num", num_pipeline_2, num_attributes),
    ("cat", cat_pipeline, cat_attributes)])
```

2. `make_column_transformer()` 를 사용해 변환기만 나열하고 특성은 타입에 따라 선택되도록 하고 이름은 자동으로 생성하는 방법

```python
preprocessing_2 = make_column_transformer(
    (num_pipeline_2, make_column_selector(dtype_include=np.number)),
    (cat_pipeline, make_column_selector(dtype_include=object)))
```  

이제 앞서 알아본 머신러닝을 위한 데이터 준비 과정 전체를 파이프라인으로 작성하면 아래와 같다.  

```python
def column_ratio(X):
    return X[:, [0]] / X[:, [1]]

def ratio_name(function_transformer, feature_names_in):
    return ["ratio"]

def ratio_pipeline():
    return make_pipeline(
        SimpleImputer(strategy="median"),
        FunctionTransformer(column_ratio, feature_names_out=ratio_name),
        StandardScaler())

log_pipeline = make_pipeline(
    SimpleImputer(strategy="median"),
    FunctionTransformer(np.log, feature_names_out="one-to-one"),
    StandardScaler())
cluster_simil = ClusterSimilarity(n_clusters=10, gamma=1, random_state=42)
default_num_pipeline = make_pipeline(SimpleImputer(strategy="median"),
                                     StandardScaler())

preprocessing = ColumnTransformer([
    ("bedrooms", ratio_pipeline(), ["total_bedrooms", "total_rooms"]),
    ("rooms_per_house", ratio_pipeline(), ["total_rooms", "households"]),
    ("people_per_house", ratio_pipeline(), ["population", "households"]),
    ("log", log_pipeline, ["total_bedrooms", "total_rooms", "population", "households", "median_income"]),
    ("geo", cluster_simil, ["latitude", "longitude"]),
    ("cat", cat_pipeline, make_column_selector(dtype_include=object))
],
remainder=default_num_pipeline)

housing_prepared = preprocessing.fit_transform(housing)
df_housing_prepared = pd.DataFrame(housing_prepared, columns=housing_prepared_feature_names, index=housing.index)
df_housing_prepared_head = df_housing_prepared.head(3)

'''
       bedrooms__ratio  rooms_per_house__ratio  people_per_house__ratio  \
13096         1.846624               -0.866027                -0.330204   
14973        -0.508121                0.024550                -0.253616   
3785         -0.202155               -0.041193                -0.051041   

       log__total_bedrooms  log__total_rooms  log__population  \
13096             1.324114          0.637892         0.456906   
14973            -0.252671         -0.063576        -0.711654   
3785             -0.925266         -0.859927        -0.941997   

       log__households  log__median_income  geo__클러스터 0 유사도  geo__클러스터 1 유사도  \
13096         1.310369           -1.071522     4.581829e-01     1.241847e-14   
14973        -0.142030            1.194712     6.511495e-10     9.579596e-01   
3785         -0.913030           -0.756981     3.432506e-01     4.261141e-15   

       geo__클러스터 2 유사도  geo__클러스터 3 유사도  geo__클러스터 4 유사도  geo__클러스터 5 유사도  \
13096     8.141160e-02     3.434950e-24     3.329278e-07         0.000210   
14973     8.649502e-14     2.712990e-02     3.992038e-02         0.000208   
3785      4.483096e-01     1.041079e-24     5.084725e-08         0.000411   

       geo__클러스터 6 유사도  geo__클러스터 7 유사도  geo__클러스터 8 유사도  geo__클러스터 9 유사도  \
13096     8.489216e-04     9.770322e-01     2.382191e-08     3.819126e-18   
14973     5.614049e-27     1.260964e-13     1.103491e-01     3.547610e-01   
3785      5.641131e-03     7.303265e-01     2.508224e-08     2.669659e-18   

       cat__ocean_proximity_<1H OCEAN  cat__ocean_proximity_INLAND  \
13096                             0.0                          0.0   
14973                             1.0                          0.0   
3785                              0.0                          1.0   

       cat__ocean_proximity_ISLAND  cat__ocean_proximity_NEAR BAY  \
13096                          0.0                            1.0   
14973                          0.0                            0.0   
3785                           0.0                            0.0   

       cat__ocean_proximity_NEAR OCEAN  remainder__housing_median_age  
13096                              0.0                       1.861119  
14973                              0.0                       0.907630  
3785                               0.0                       0.351428  
'''

```   

### 모델 선택과 훈련
앞선 과정까지 모두해서 머신러닝에 사용할 데이터셋을 훈련 세트와 테스트 세트로 나누고, 데이터를 변환하고, 모델에 주입할 수 있는 파이프라인을 작성했다. 
가장 먼저 `LinearRegression` 선형 회귀 모델을 사용해서 훈련 시키고, 그 예측 결과와 실제 값 사이의 `RMSE` 평균 제곱 오차를 계산해본다.   

```python
lin_reg = make_pipeline(preprocessing, LinearRegression())
lin_reg.fit(housing, housing_labels)

housing_predictions = lin_reg.predict(housing)
housing_predictions_head = housing_predictions[:5].round(-2)

'''
[246000. 372700. 135700.  91400. 330900.]
'''

housing_labels_head = housing_labels[:5].values

'''
[458300. 483800. 101700.  96100. 361800.]
'''

housing_predictions = tree_reg.predict(housing)
tree_rmse = mean_squared_error(housing_labels, housing_predictions) ** (1/2)

'''
68972.88910758459
'''

```  

예측 결과와 레이블된 데이터의 치이를 보면 예측결과가 크게 벗어나는 것을 확인할 수 있다.
그리고 `RMSE` 를 사용해서 오차를 측정하면 생각보다 큰 오차를 확인할 수 있다. (주택 가격은 120,000 ~ 265,000 사이로 예측했지만 오차가 68,972로 크게 벗어남)
이는 모델이 훈련 데이터에 과소적합된 상태로, 모델이 강력하지 못하거나 좋은 예측을 만들어 낼 수 있을 정도로 충분한 데이터를 제공하지 못한 걸로 생각할 수 있다.   

과소적합을 해결하는 주요한 방법은 더 강력한 모델을 사용하거나, 
훈련 알고리즘에 좋은 특성을 더 부여 혹은 모델 규제를 감소하는 것이다. 
그 중에서 좀 더 복잡한 모델을 사용하는 방법으로 오차를 개선해 본다.  

선현 회귀 모델보다 좀 더 강력한 모델인 `DecisionTreeRegressor` 결정 트리 모델을 사용해 비선형 관계도 찾을 수 있도록 개선해 본다. 

```python
tree_reg = make_pipeline(preprocessing, DecisionTreeRegressor(random_state=42))
tree_reg.fit(housing, housing_labels)

housing_predictions = tree_reg.predict(housing)
tree_rmse = mean_squared_error(housing_labels, housing_predictions) ** (1/2)

'''
0.0
'''
```  

결정 트리 모델을 사용하면 `RMSE` 가 0으로 나타나는데, 이는 모델이 훈련 데이터에 과대적합된 상태로, 
`k-fold cross-validation(k-fold 교차 검증)`  을 사용해 모델의 성능을 평가하도록 한다. 
이는 훈련 데이터를 여러 서브셋으로 나누어 여러 모델을 훈련시키고 평가하는 방법이다. 
이를 통해 모델이 새로운 데이터에 대해 얼마나 잘 일반화되는지 평가할 수 있다.  

```python
tree_rmses = -cross_val_score(tree_reg, housing, housing_labels, scoring="neg_root_mean_squared_error", cv=10)
pd_tree_rmses_desc = pd.Series(tree_rmses).describe()

'''
count       10.000000
mean     66573.734600
std       1103.402323
min      64607.896046
25%      66204.731788
50%      66388.272499
75%      66826.257468
max      68532.210664
dtype: float64
'''
```  

`cv=10` 을 사용해 10개의 서브셋으로 나누어 평가한 결과, `RMSE` 가 64,607 ~ 68,532 사이로 나타났다. 
이는 실제로 앞서 사용한 선형 회귀 모델과 비슷하게 오차가 높은 것 을 알 수 있다.  

다음으로는 `RandomForestRegressor` 랜덤 포레스트 모델을 사용해 모델을 개선해 본다. 
이는 특성을 랜덤으로 선택해 많은 결정 트리를 만들고 예측의 평균을 구하는 방식이다. 
그리고 랜덤 포레스트는 서로 다른 모델로 구성된 `emsemble` 앙상블 모델이기 때문에 과대적합을 줄이고 안정적인 모델을 만들 수 있다.  

```python
forest_reg = make_pipeline(preprocessing, RandomForestRegressor(random_state=42))
forest_rmses = -cross_val_score(forest_reg, housing, housing_labels, scoring="neg_root_mean_squared_error", cv=10)
pd_forest_rmses_desc = pd.Series(forest_rmses).describe()

'''
count       10.000000
mean     47038.092799
std       1021.491757
min      45495.9766493
25%      46510.418013
50%      47118.719249
75%      47480.519175
max      49140.832210
dtype: float64
'''
``` 

`RMSE` 가 45,495 ~ 49,140 사이로 나타나며, 이는 앞서 사용한 결정 트리 모델보다 훨씬 낮은 것을 확인할 수 있다. 


### 모델 튜닝
앞서 여러 모델을 사용해 모델을 훈련하고 평가해본 결과, 랜덤 포레스트 모델이 가장 좋은 성능을 보였다. 
이번에는 해당 모델을 사용해서 하이퍼파라미터를 튜닝해 모델의 성능을 개선해본다.  

가장 간단한 튜닝 방법은 `GridSearchCV` 그리드 탐색을 사용하는 것이다. 
이는 탐색하고자 하는 하이퍼파라미터와 시도해볼 값을 지정하면, 가능한 모든 하이퍼파라미터 조합에 대해 교차 검증을 사용해 평가한다. 
그리고 가장 좋은 조합을 찾아 최종 모델을 훈련시킨다.  

```python
full_pipeline = Pipeline([
    ("preprocessing", preprocessing),
    ("random_forest", RandomForestRegressor(random_state=42))
])

param_grid = [
    {'preprocessing__geo__n_clusters' : [5, 8, 10],
     'random_forest__max_features' : [4 ,6, 8]},
    {'preprocessing__geo__n_clusters' : [10, 15],
     'random_forest__max_features' : [6, 8, 10]}
]

grid_search = GridSearchCV(full_pipeline, param_grid, cv=3, scoring="neg_root_mean_squared_error")
grid_search.fit(housing, housing_labels)
grid_search_best_params = grid_search.best_params_

'''
{'preprocessing__geo__n_clusters': 15, 'random_forest__max_features': 6}
'''

grid_search_best_estimator = grid_search.best_estimator_

'''
Pipeline(steps=[('preprocessing',
                 ColumnTransformer(remainder=Pipeline(steps=[('simpleimputer',
                                                              SimpleImputer(strategy='median')),
                                                             ('standardscaler',
                                                              StandardScaler())]),
                                   transformers=[('bedrooms',
                                                  Pipeline(steps=[('simpleimputer',
                                                                   SimpleImputer(strategy='median')),
                                                                  ('functiontransformer',
                                                                   FunctionTransformer(feature_names_out=<function ratio_name at 0x1350a4c...
                                                                    n_clusters=15,
                                                                    random_state=42),
                                                  ['latitude', 'longitude']),
                                                 ('cat',
                                                  Pipeline(steps=[('simpleimputer',
                                                                   SimpleImputer(strategy='most_frequent')),
                                                                  ('onehotencoder',
                                                                   OneHotEncoder(handle_unknown='ignore'))]),
                                                  <sklearn.compose._column_transformer.make_column_selector object at 0x135386500>)])),
                ('random_forest',
                 RandomForestRegressor(max_features=6, random_state=42))])
'''

grid_search_cv_results = pd.DataFrame(grid_search.cv_results_)
grid_search_cv_results.sort_values(by="mean_test_score", ascending=False, inplace=True)
grid_search_cv_results_head = grid_search_cv_results.head()

'''
    mean_fit_time  std_fit_time  mean_score_time  std_score_time  \
12       4.084342      0.085883         0.093229        0.001272   
13       5.148865      0.021240         0.095692        0.001686   
7        3.949567      0.014981         0.094121        0.001407   
9        3.814266      0.023778         0.093028        0.002168   
6        2.759176      0.031862         0.094886        0.002125   

    param_preprocessing__geo__n_clusters  param_random_forest__max_features  \
12                                    15                                  6   
13                                    15                                  8   
7                                     10                                  6   
9                                     10                                  6   
6                                     10                                  4   

                                               params  split0_test_score  \
12  {'preprocessing__geo__n_clusters': 15, 'random...      -42725.423800   
13  {'preprocessing__geo__n_clusters': 15, 'random...      -43486.844936   
7   {'preprocessing__geo__n_clusters': 10, 'random...      -43709.661050   
9   {'preprocessing__geo__n_clusters': 10, 'random...      -43709.661050   
6   {'preprocessing__geo__n_clusters': 10, 'random...      -43803.337116   

    split1_test_score  split2_test_score  mean_test_score  std_test_score  \
12      -43709.972137      -44334.935606    -43590.110514      662.524047   
13      -43812.534498      -44899.968680    -44066.449371      604.198781   
7       -44158.585837      -44966.539107    -44278.261998      520.049613   
9       -44158.585837      -44966.539107    -44278.261998      520.049613   
6       -44072.242306      -44960.694004    -44278.757809      494.540347   

    rank_test_score  
12                1  
13                2  
7                 3  
9                 3  
6                 5  
'''
```  

`full_pipeline` 정의 부분을 보면 알겠지만, 
파이프라인이나 `ColumnTransformer` 가 여러번 랩핑되어 있는 것을 볼 수 있다. 
이런 경우 `preprocessing__geo__n_clusters` 와 같이 하이퍼파라미터를 설정할 때는 `__` 를 사용해 하위 이름을 지정할 수 있다. 
그리고 `GridSearchCV` 를 사용해 최적의 하이퍼파라미터를 찾아낸 결과는 `best_params_` 를 통해 확인할 수 있는데, 
`preprocessing__geo__n_clusters` 가 15, `random_forest__max_features` 가 6인 것을 확인할 수 있다.  

`best_estimator_` 을 사용하면 최상의 추정기 정보를 얻을 수 있고, 
`cv_results_` 를 사용하면 교차 검증 결과에 대한 정보를 얻을 수 있다. 
결과적으로 그리드 서치 튜닝전 `RMSE` 가 47,038 ~ 49,140 사이로 나타났던 것을 42,725 ~ 44,334 사이로 개선된 것을 확인할 수 있다.  

다음으로 알아볼 튜닝 방법은 `RandomizedSearchCV` 랜덤 탐색을 사용하는 방법이다. 
이는 `GridSearchCV` 와 비슷하지만 모든 하이퍼파라미터 조합을 시도하는 대신 각 반복마다 하이퍼파라미터에 임의의 수를 대입해 지정한 횟수만큼 평가한다.
하이퍼파라미터가 많고 탐색 공간이 커지면 `GridSearchCV` 를 사용하는 것 보다는 `RandomizedSearchCV` 를 사용하는 것이 더 좋은 솔루션을 찾을 가능성이 높아진다. 
`RandomizedSearchCV` 는 `n_iter` 를 사용해 반복 횟수를 지정하고, `param_distributions` 를 사용해 탐색할 하이퍼파라미터의 분포를 지정한다. 


```python
param_distribs = {'preprocessing__geo__n_clusters': randint(low=3, high=50),
                  'random_forest__max_features': randint(low=2, high=20)}

rnd_search = RandomizedSearchCV(
    full_pipeline, param_distributions=param_distribs, n_iter=10, cv=3, scoring="neg_root_mean_squared_error", random_state=42)

rnd_search.fit(housing, housing_labels)
rnd_search_best_params = rnd_search.best_params_

'''
{'preprocessing__geo__n_clusters': 45, 'random_forest__max_features': 9}
'''

rnd_search_cv_results = pd.DataFrame(rnd_search.cv_results_)
rnd_search_cv_results.sort_values(by="mean_test_score", ascending=False, inplace=True)
rnd_search_cv_results_head = rnd_search_cv_results.head()

'''
   mean_fit_time  std_fit_time  mean_score_time  std_score_time  \
1       6.100925      0.051159         0.122319        0.019098   
8       4.949723      0.071942         0.123621        0.003883   
0      10.691056      0.133879         0.124185        0.010074   
5       3.207108      0.036648         0.127999        0.010028   
2       5.299480      0.055173         0.092626        0.000761   

   param_preprocessing__geo__n_clusters  param_random_forest__max_features  \
1                                    45                                  9   
8                                    32                                  7   
0                                    41                                 16   
5                                    42                                  4   
2                                    23                                  8   

                                              params  split0_test_score  \
1  {'preprocessing__geo__n_clusters': 45, 'random...      -41341.655457   
8  {'preprocessing__geo__n_clusters': 32, 'random...      -41808.411878   
0  {'preprocessing__geo__n_clusters': 41, 'random...      -42238.113363   
5  {'preprocessing__geo__n_clusters': 42, 'random...      -41883.574436   
2  {'preprocessing__geo__n_clusters': 23, 'random...      -42473.208374   

   split1_test_score  split2_test_score  mean_test_score  std_test_score  \
1      -42242.449878      -43106.582615    -42230.229317      720.580310   
8      -42288.468626      -43243.957345    -42446.945950      596.676346   
0      -42938.035062      -43353.747344    -42843.298590      460.355691   
5      -43380.439860      -43705.761329    -42989.925208      793.501749   
2      -42949.922813      -43724.928000    -43049.353063      515.826383   

   rank_test_score  
1                1  
8                2  
0                3  
5                4  
2                5  
'''

```  

`RandomizedSearchCV` 를 사용해 최적의 하이퍼파라미터를 찾아낸 결과는 `preprocessing__geo__n_clusters` 가 45, `random_forest__max_features` 가 9인 것을 확인할 수 있다. 
이때 오차는 `GridSearchCV` 를 사용한 것보다 좀 더 좋은 41,341 ~ 43,106 사이로 개선된 것을 확인할 수 있다.  

추가로 `RandomForestRegressor` 는 정확한 예측을 위해 각 특성에 대한 상대적 중요도를 제공한다. 

```python
final_model = rnd_search.best_estimator_
feature_importances = final_model["random_forest"].feature_importances_
feature_importances.round(2)
feature_importances_desc = sorted(zip(feature_importances, final_model["preprocessing"].get_feature_names_out()), reverse=True)

'''
[(0.18673440937412847, 'log__median_income'), 
(0.0732445627048804, 'cat__ocean_proximity_INLAND'), 
(0.06573054277260683, 'bedrooms__ratio'), 
(0.05353772260040246, 'rooms_per_house__ratio'), 
(0.04599485538141581, 'people_per_house__ratio'), 
(0.04179425251060039, 'geo__클러스터 30 유사도'), 
...
(0.0015109964177570441, 'cat__ocean_proximity_NEAR OCEAN'), 
(0.0004295477685850398, 'cat__ocean_proximity_NEAR BAY'), 
(3.019022110267028e-05, 'cat__ocean_proximity_ISLAND')]
'''
```  

만약 시스템이 특정한 오차를 만들었다면 이를 분석하고, 
이를 보완하가 위한 특성을 추가하거나 불필요한 특성 제고 및 이상치 제외등을 고민해야 한다.  


모델 튜닝까지 완료됐으니 테스트 세트를 사용해 최종 모델을 펑가한다. 
테스트 세트를 사용해 예측 결과를 만들고, `RMSE` 를 계산해 모델의 성능을 평가한다.  

```python
x_test = strat_test_set.drop('median_house_value', axis=1)
y_test = strat_test_set['median_house_value'].copy()

final_predictions = final_model.predict(x_test)

final_rmse = mean_squared_error(y_test, final_predictions) ** (1 / 2)

'''
41448.084299370465
'''

# 신뢰 구간 95% 계산
confidence = 0.95
squared_errors = (final_predictions - y_test) ** 2
confidence_interval = np.sqrt(stats.t.interval(confidence, len(squared_errors) - 1,
                                               loc=squared_errors.mean(),
                                               scale=stats.sem(squared_errors)))
'''
[39293.29060201 43496.26073402]
'''
```  

테스트 세트를 사용해 최종 모델을 평가한 결과, `RMSE` 가 41,448로 나타났다. 
그리고 `신뢰 구간 95%` 를 계산한 결과, 39,293 ~ 43,496 사이로 나타났다. 
이는 모델이 새로운 데이터에 대해 얼마나 잘 일반화되는지 평가한 것으로, 
이를 통해 모델의 성능을 평가하고, 모델이 얼마나 신뢰할 수 있는지 확인할 수 있다.  

### 모델 배포
마지막으로 모델을 배포하는 단계로, 모델을 사용해 새로운 데이터에 대한 예측을 만들어야 한다. 
이를 위해 모델을 저장하고, 웹 서비스나 모바일 앱에 통합해야 한다. 
모델을 저장하는 방법은 `joblib` 라이브러리를 사용해 모델을 저장하고 로드하는 방법이 있다.  

```python
# 모델 저장
joblib.dump(final_model, "my_housing_model.pkl")

# 모델 로드
# 모델을 다른 시스템에서 로드 할때는 사용자 정의 함수 및 클래스를 먼저 정의 및 임포트해야 한다. (e.g. ClusterSimilarity)
final_model_reloaded = joblib.load('my_housing_model.pkl')
predictions = final_model_reloaded.predict(new_data)
```  




---  
## Reference
[handson-ml3](https://github.com/rickiepark/handson-ml3)  

