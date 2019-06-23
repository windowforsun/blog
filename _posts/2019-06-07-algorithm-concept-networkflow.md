--- 
layout: single
classes: wide
title: "[Algorithm 개념] 네트워크 플로우(Network Flow) 포드 풀커슨(Ford Fulkerson), 에드몬드 카프(Edmonds-Karp) 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '포드 풀커슨, 에드몬드 카프 알고리즘으로 네트워크 플로우를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Network Flow
  - Max Flow
  - Ford Fulkerson
  - Edmonds Karp
use_math : true
---  

## 네트워크 플로우란
- 그래프에서 각 노드들 간의 용량(Capacity)이 정의되어 있을때, 시작점(Source)에서 도착점(Target, Sink) 까지 흐를 수 있는 최대 유량 구하는 것이다.
- DFS 를 사용해서 포드 풀커슨(Ford-Fulkerson) 알고리즘으로 해결 할 수 있다.
- BFS 를 사용해서 에드몬드 카프(Edmonds-Karp) 알고리즘으로 해결 할 수 있다.

---
## Reference
[[Algorithm] 네트워크 유량(Network Flow)](https://engkimbs.tistory.com/353)  
[네트워크 플로우(Network Flow)](https://www.crocus.co.kr/741)  
[네트워크 유량(Network Flow) (수정: 2018-12-13)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220804885235)  
[네트워크플로우(Network flow) - 포드 풀커슨(Ford-Fulkerson) 알고리즘](https://coderkoo.tistory.com/4)  
[네트워크 플로우(network flow)](https://www.zerocho.com/category/Algorithm/post/5893405b588acb00186d39e0)  

