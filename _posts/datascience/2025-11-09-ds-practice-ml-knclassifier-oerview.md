--- 
layout: single
classes: wide
title: "[ML] K-Nearest Neighbors Classifier"
header:
overlay_image: /img/data-science-bg.jpg
excerpt: 'K-Nearest Neighbors(KNN), K-최근접 이웃 알고리즘에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - ML
    - KNN
    - Classifier
    - K-Nearest Neighbors
    - Supervised Learning
    - Regression
toc: true
use_math: true
---  


## K-Nearest Neighbors Classifier
`K-Nearest Neighbors(KNN)`(K-최근접 이웃) 알고리즘은 지도 학습(`Supervised Learning`)의 한 종류로, 
분류(`Classifier`) 및 회귀(`Regression`)에 사용되는 알고리즘이다. 
거리 기반 분류 알고리즘으로 새로운 데이터 포인트가 주어지면 기존 학습 데이터 중 가장 가까운 `K` 개의 데이터를 참고해 분류를 수행한다.  

작동 원리는 아래와 같다. 

- 분류하는 새로운 데이터 포인트와 기존 학습 데이터 간의 거리를 계산한다. 
- 가장 가까운 `K` 개의 데이터를 선택한다. 
- `K` 개 데이터 중 가장 많이 등장하는 클래스를 새 데이터의 클래스로 지정한다. 

`KNN` 에서 `K` 값을 선택할 때는 아래와 같은 고려 사항이 있다. 

- 작은 `K` 값 : 과적합(`Overfitting`)이 발생할 수 있다. 
- 큰 `K` 값 : 모델이 더 일반화될 수 있지만, 계산량이 늘어나고 경계가 흐려진다. 
- 일반적으로 홀수를 선택해 다수결이 가능하도록 한다. 

`KNN` 의 장단점은 아래와 같다. 

- 장점
  - 이해하기 쉽고 구현이 간단하다. 
  - 모델 훈련 과정이 거의 없다.(`Instance-based Learning`)
  - 다중 클래스 분류가 가능하다. 
- 단점
  - 데이터가 많이지면 거리 계산이 많아져 성능이 좋지 못하다. 
  - 차원의 저주의 영향을 받을 수 있다. 
  - 이상치(`Outlier`)에 민감하다. 

## 예제
`KNN` 의 예제로는 도미와 빙어를 분류하는 예제로 진행한다. 
이는 아주 간단한 예제로, 도메와 빙어의 길기와 무게를 통해 분류하는 예제이다. 
데이터 세트 준비 및 코드 전반적인 내용이 실무에서 사용하기에 적합하지 않고, 
`KNN` 및 기본 `ML` 모델을 이해하는데 도움이 될 수 있는 예제임을 기억해야 한다. 

<details><summary>전체 코드</summary>
<div markdown="1">

```
matplotlib===3.7.1
numpy===1.23.5
scikit-learn===1.2.2
tensorflow===2.14.0
```

```python
import matplotlib.pyplot as plt
import numpy as np
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import train_test_split

fish_length = [25.4, 26.3, 26.5, 29.0, 29.0, 29.7, 29.7, 30.0, 30.0, 30.7, 31.0, 31.0,
               31.5, 32.0, 32.0, 32.0, 33.0, 33.0, 33.5, 33.5, 34.0, 34.0, 34.5, 35.0,
               35.0, 35.0, 35.0, 36.0, 36.0, 37.0, 38.5, 38.5, 39.5, 41.0, 41.0, 9.8,
               10.5, 10.6, 11.0, 11.2, 11.3, 11.8, 11.8, 12.0, 12.2, 12.4, 13.0, 14.3, 15.0]
fish_weight = [242.0, 290.0, 340.0, 363.0, 430.0, 450.0, 500.0, 390.0, 450.0, 500.0, 475.0, 500.0,
               500.0, 340.0, 600.0, 600.0, 700.0, 700.0, 610.0, 650.0, 575.0, 685.0, 620.0, 680.0,
               700.0, 725.0, 720.0, 714.0, 850.0, 1000.0, 920.0, 955.0, 925.0, 975.0, 950.0, 6.7,
               7.5, 7.0, 9.7, 9.8, 8.7, 10.0, 9.9, 9.8, 12.2, 13.4, 12.2, 19.7, 19.9]

# 전체 학습 데이터
fish_data = np.column_stack((fish_length, fish_weight))
# 전체 타겟
fish_target = np.concatenate((np.ones(35), np.zeros(14)))

# 훈련 데이터와 테스트 데이터로 나누기
train_input, test_input, train_target, test_target = train_test_split(fish_data, fish_target, stratify=fish_target,
                                                                      random_state=42)

print(test_input)

plt.scatter(train_input[:, 0], train_input[:, 1])
plt.scatter(test_input[:, 0], test_input[:, 1])
plt.xlabel('length')
plt.ylabel('weight')
plt.show()


kn = KNeighborsClassifier()
kn.fit(train_input, train_target)
score = kn.score(test_input, test_target)

print('score')
print(score)


# 새로운 도미 데이터를 빙어로 예측
print(kn.predict([[25, 150]]))




plt.scatter(train_input[:, 0], train_input[:, 1])
plt.scatter(25, 150, marker='^')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()

distances, indexes = kn.kneighbors([[25, 150]])

plt.scatter(train_input[:, 0], train_input[:, 1])
plt.scatter(25, 150, marker='^')
plt.scatter(train_input[indexes, 0], train_input[indexes, 1], marker='D')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()

print(train_target[indexes])

print(distances)


mean = np.mean(train_input, axis=0)
std = np.std(train_input, axis=0)

train_scaled = (train_input - mean) / std

new = ([25, 150] - mean) / std
plt.scatter(train_scaled[:,0], train_scaled[:,1])
plt.scatter(new[0], new[1], marker='^')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()

kn.fit(train_scaled, train_target)
test_scaled = (test_input - mean) / std

print(kn.score(test_scaled, test_target))

print(kn.predict([new]))


distances, indexes = kn.kneighbors([new])
plt.scatter(train_scaled[:,0], train_scaled[:,1])
plt.scatter(new[0], new[1], marker='^')
plt.scatter(train_scaled[indexes, 0], train_scaled[indexes, 1], marker='D')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()

```

</div>
</details>

### 데이터 세트 준비

도미와 빙어의 데이터는 미리 아래와 같이 준비한다. 

```python
fish_length = [25.4, 26.3, 26.5, 29.0, 29.0, 29.7, 29.7, 30.0, 30.0, 30.7, 31.0, 31.0,
               31.5, 32.0, 32.0, 32.0, 33.0, 33.0, 33.5, 33.5, 34.0, 34.0, 34.5, 35.0,
               35.0, 35.0, 35.0, 36.0, 36.0, 37.0, 38.5, 38.5, 39.5, 41.0, 41.0, 9.8,
               10.5, 10.6, 11.0, 11.2, 11.3, 11.8, 11.8, 12.0, 12.2, 12.4, 13.0, 14.3, 15.0]
fish_weight = [242.0, 290.0, 340.0, 363.0, 430.0, 450.0, 500.0, 390.0, 450.0, 500.0, 475.0, 500.0,
               500.0, 340.0, 600.0, 600.0, 700.0, 700.0, 610.0, 650.0, 575.0, 685.0, 620.0, 680.0,
               700.0, 725.0, 720.0, 714.0, 850.0, 1000.0, 920.0, 955.0, 925.0, 975.0, 950.0, 6.7,
               7.5, 7.0, 9.7, 9.8, 8.7, 10.0, 9.9, 9.8, 12.2, 13.4, 12.2, 19.7, 19.9]

# 전체 데이터
fish_data = np.column_stack((fish_length, fish_weight))
# 전체 타겟 1: 도미, 0: 빙어
fish_target = np.concatenate((np.ones(35), np.zeros(14)))
```  

그리고 `ML` 모델을 학습할 때 모든 데이터 세트를 사용하는 것 보다는, 
보유하고 있느 데이터를 일부만 훈련 세트(`train set`)로 사용하고 일부는 테스트 세트(`test set`)로 사용하는 것이 좋다. 
이렇게 데이터가 분리가 돼 있어야 훈련된 모델이 얼마나 잘 일반화 됐는지 등 성능을 최종 평가할 수 있기 때문이다.  

이렇게 훈련 세트와 테스트 세트를 나눌 때 전체 데이터를 바탕으로 직접 섞어서 나눌 수도 있지만, 
`sklearn` 의 `train_test_split()` 를 사용하면 더 편리하게 나눌 수 있다. 
`train_test_split()` 은 기본으로 `25%` 를 테스트 세트로 떼어 놓는다. 
또한 무작정 무작위로 데이터를 나눌경우 클래스별 샘플의 비율이 골고루 나뉘지 않을 수 있다. 
이를 위해서는 `stratify` 매개변수에 타깃 데이터를 전달해 클래스 비율에 맞게 데이터를 나눌 수 있다.  

```python
# 훈련 데이터와 테스트 데이터로 나누기
train_input, test_input, train_target, test_target = train_test_split(fish_data, fish_target, stratify=fish_target,
                                                                      random_state=42)
```  

훈련 세트와 테스트 세트로 나누어진 결과를 확인해 보면 아래와 같다.  

```python

plt.scatter(train_input[:, 0], train_input[:, 1])
plt.scatter(test_input[:, 0], test_input[:, 1])
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```  

![그림 1]({{site.baseurl}}/img/datascience/knclassifier-overview-1.png)


### KNN 모델 훈련
`KNN` 모델을 훈련해 본다. 
훈련 세트를 사용해 모델을 훈련하고, 테스트 세트를 사용해 모델을 평가하면 아래와 같다. 
사이킷런에서 `KNN` 모델을 사용하려면 `KNeighborsClassifier` 클래스를 임포트하고 객체를 생성한 뒤 `fit()` 메서드로 모델을 훈련한다. 
그리고 `score()` 메서드로 모델을 평가한다. 

```python
kn = KNeighborsClassifier()
kn.fit(train_input, train_target)
score = kn.score(test_input, test_target)

print('score')
print(score)
# 1.0
```  

모델의 평가 결과인 `score` 는 `1` 로 나오는데, 이는 모델이 모든 데이터를 정확하게 분류했다는 의미이다. 
즉 훈련 세트를 바탕으로 훈련한 모델이 테스트 세트를 모두 정확하게 분류했다는 의미로 해석할 수 있고, 
평가 결과만봐서는 완벽한 모델을 만든 것이라고 판단할 수 있다.  


### 특성 스케일 조정
`KNN` 은 거리 기반 알고리즘이기 때문에 특성의 스케일이 다르면 제대로 작동하지 않을 수 있다. 
따라서 특성을 표준화(`Standardization`) 하는 것이 좋다.
우선 특정 스케일을 조정하지 않았을 때 어떤 결과가 나오는지 아래 예시를 확인해 본다.  

```python
# 새로운 도미 데이터를 빙어로 예측
print(kn.predict([[25, 150]]))
# [0]
```  

이를 시각적으로 확인하기 위해 산점도 그래프를 그리면 아래와 같다.  

```python
plt.scatter(train_input[:, 0], train_input[:, 1])
plt.scatter(25, 150, marker='^')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```  

![그림 1]({{site.baseurl}}/img/datascience/knclassifier-overview-2.png)

산점도 그래프를 봤을 떄도 새로운 도미 데이터는 다른 훈련 세트의 도메 데이터들과 더 가까운 것을 확인할 수 있다. 
하지만 모델이 이를 빙어로 예측한 것은 두 특성(길이, 무게)의 스케일이 다르기 때문이다. 
우선 새로운 도미 데이터 예측에 사용한 이웃 샘플이 어떤 건지 `kneighbors()` 메서드로 확인해 본다.  

```python
distances, indexes = kn.kneighbors([[25, 150]])

plt.scatter(train_input[:, 0], train_input[:, 1])
plt.scatter(25, 150, marker='^')
plt.scatter(train_input[indexes, 0], train_input[indexes, 1], marker='D')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```  

![그림 1]({{site.baseurl}}/img/datascience/knclassifier-overview-3.png)

새로운 도미 데이터의 가장 가까운 이웃 중에는 도미가 하나만 있고 모두 빙어인 것을 볼 수 있다. 
좀 더 명확하게 확인하기 위해 `kneighbors()` 메서드로 반환된 인덱스의 타겟 클래스와 거리를 확인해 본다.  

```python
print(train_target[indexes])
# [[1 0 0 0 0]]

print(distances)
# [[ 92.00086956 130.48375378 130.73859415 138.32150953 138.39320793]]
```  

산점도 그래프에서 `x` 축인 길이의 범위는 `10 ~ 40` 으로 비교적 좁고, 
`y` 축인 무게의 범위는 `0 ~ 1000` 으로 넓다. 
즉 `y` 축으로 조금만 멀어져도 거리가 아주 큰 값으로 계산되기 때문에 예상대로 `KNN` 모델이 동작하지 못한 것이다.  

이를 해결하기 위해 특성값을 일정한 기준으로 맞춰야 한다. 
이러한 작업을 `data preprocessing` 데이터 전처리 이라고 한다. 
가장 보편적인 방법은 `standard score` 표준 점수로 
각 특성값이 평균에서 표준편차의 몇 배만큼 떨어져 있는지를 나타내는 값으로 변환하는 것이다.  
이러한 데이터 전처리르 제공해주는 편리한 클래스가 존재하지만, 
이번에는 직접 계산해 본다.  

표준점수를 계산하기 위해서는 `np.mean()` 으로 평균을 구한 뒤, `np.std()` 로 표준 편차를 계산해 값에 평균을 빼고 표준편차로 나누어 주면 된다. 
`train_input` 은 `(36, 2)` 크기의 배열이기 때문에 평균과 표준편차를 각 특성별로 즉 열을 따라 모든 행에 대해 계산해야 하므로 `axis=0` 으로 지정한다.  

```python
mean = np.mean(train_input, axis=0)
std = np.std(train_input, axis=0)

train_scaled = (train_input - mean) / std
```  

특성이 스케일을 변경했을 때 주의해야 할점은 훈련 세트의 평균과 표준편차를 사용해 테스트 세트와 이후에 사용할 새로운 데이터에도 같은 변환을 적용해야 한다. 
이제 새로운 도미 데이터도 훈련 세트의 평균과 표준편차를 사용해 표준점수로 변환해 다시 산점도 그래프를 그린다. 
그리고 다시 훈련과 평가 그리고 다시 새로운 데이터로 예측을 하면 아래와 같다.  

```python
new = ([25, 150] - mean) / std
plt.scatter(train_scaled[:,0], train_scaled[:,1])
plt.scatter(new[0], new[1], marker='^')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```  

![그림 1]({{site.baseurl}}/img/datascience/knclassifier-overview-4.png)


```python
kn.fit(train_scaled, train_target)
test_scaled = (test_input - mean) / std

print(kn.score(test_scaled, test_target))
# 1.0

print(kn.predict([new]))
# [1]
```  

특성의 스케일을 조정해 주니 훈련된 모델이 새로운 데이터도 예상대로 예측을 성공했다. 
정확한 확인을 위해 새로운 도미 데이터의 가장 가까운 이웃을 확인하면 아래와 같다. 

```python
distances, indexes = kn.kneighbors([new])
plt.scatter(train_scaled[:,0], train_scaled[:,1])
plt.scatter(new[0], new[1], marker='^')
plt.scatter(train_scaled[indexes, 0], train_scaled[indexes, 1], marker='D')
plt.xlabel('length')
plt.ylabel('weight')
plt.show()
```

![그림 1]({{site.baseurl}}/img/datascience/knclassifier-overview-5.png)

산점도 그래프에서 확인할 수 있듯이 새로운 도메 샘플의 가장 가까운 이웃은 모두 도미로 정확하게 예측한 것을 확인할 수 있다.


---  
## Reference
[rickiepark/hg-mldl](https://github.com/rickiepark/hg-mldl)  

