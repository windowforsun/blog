--- 
layout: single
classes: wide
title: "[TimeSeries] MA(Moving Average)"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: '이동평균(MA, Moving Average) 프로세스 모델링과 예측에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - Data Science
    - Time Series
    - MA
    - Moving Average
toc: true
use_math: true
---  

## Moving Average Process
`이동평균(MA, Moving Average)` 는 시계열 데이터의 현재 값이 과거의 백색잡음 항들의 선형결합으로 표현되는 과정을 의미한다. 
다르게 설명하면 현재값이 현재의 과거 오차에 선형적으로 비례한다고 할 수 있다.  
이동평균 모델의 일반적인 표기는 `MA(q)` 를 사용하는데 여기서 `q` 는 이동평균의 차수를 의미한다. 

`q` 는 시게열 모델이 현재 시점의 데이터 값을 설명할 때 얼마나 이전까지의 오차(백색소음) 항을 포함하지는지를 나타내는 숫자이다. 
즉, 몇 시점 전까지의 오차가 현재 값에 명향을 미치는지를 결정하는 파라미터라고 할 수 있다. 
만약 `q` 가 크다면 더 오랜 과거의 오차까지 현재 값에 영향을 미친다고 할 수 있고, 
`q` 가 작다면 최근 오차 항만을 고려한다고 할 수 있다.  

