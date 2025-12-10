--- 
layout: single
classes: wide
title: "[TimeSeries] Random Walk"
header:
  overlay_image: /img/data-science-bg.jpg
excerpt: '시계열에서 Random Walk(확률 보행)의 개념과 시뮬레이션, 정상성 검정, 예측 방법을 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Data Science
tags:
    - Practice
    - Data Science
    - Time Series
    - Random Walk
    - Stationary Time Series
    - ADF Test
    - ACF
toc: true
use_math: true
---  


## Random Walk
`Random Walk`(확률 보행)은 다음 값이 이전 값에 무작위로 더해지거나 빼지는, 
즉 상승이나 하락의 확률이 동일한 시계열 프로세스이다. 
금융 데이터처럼 실제 데이터에서도 자주 나타는 형태이다. 
이런 확률 보행의 특징은 때로는 한 방향의 추세가 길게 지속되기도 하고, 갑작스러운 변화가 일어나기도 한다. 
