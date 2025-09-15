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
