--- 
layout: single
classes: wide
title: "[Algorithm 개념] Network Flow 포드 풀커슨(Ford Fulkerson), 에드몬드 카프(Edmonds-Karp) 알고리즘"
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

## 네트워크 플로우의 용어
- 소스 Source(S)
	- 시작 위치를 의미한다.
- 싱크  Sink, Target(T)
	- 끝 위치를 의미한다.
- 용량 Capacity(c)
	- 유량이 흐를 수 있는 크기를 의미한다.
- 유량 Flow(f)
	- 간선에 흐르는 현재 유량의 크기를 의미한다.
- 잔류 유량 Residual Flow
	- Capacity - Flow 의 값으로 현재 간선에 흐를 수 있는 유량이 얼마인지 의미한다. ($c(u,v)-f(u,v) > 0$)
- c(u,v)
	- u 에서 v(u->v) 로 흐를 수 있는 간선의 용량을 의미한다.
- f(u,v) 
	- u 에서 v(u->v) 로 흐르는 실제 유량을 의미한다.
	
## 네트워크 플로우의 성질
- 특정 경로(ex, s->1->4->5->t)로 유량을 보낼 때는 경로에 포함된 간선 중 가장 작은 용량의 간선에 의해 용량이 결정된다.
- $f(u,v) \le c(u,v)$ : 용량 제한 속성
	- 흐르는 양(f)은 그 간선의 용량(c)를 넘을 수 없다.
- $f(u,v) \eq -f(v,u)$ :  유량의 대칭성
	- 1->2 로 3의 유량을 흘려 보냈다는 것은, 2->1 로 -3의 유량을 흘려 보냈다는 것과 같은 의미이다.
- $\sum_ f(u,v) = 0$ : 유량의 보존
	- 소스와 싱크를 제외한 모든 노드에서 들어오는 유량의 합과 나가는 유량의 합은 같아야 한다.
	
## 네트워크 플로우 알고리즘 원리
- S에서 T로 가는 아직 잔류 용량이 있는 경로를 찾는다.
- 경로 중에서 $c(u,v)-f(u,v)$ 값이 최소인 간선을 찾고(현재 경로에서 흘려 보낼 수 있는 유량), 이 값을 F 라고 한다.
- 경로상의 모든 간선에 F 만큼 유량을 추가한다. 
	- $f(u,v) += F$
	- $f(v,u) -= F$
- 위의 연산을 더 이상 잔류 유량이 있는 경로가 없을 때 까지 반복한다.
- S에서 T로 가는 경로를 DFS 로 찾으면 포드 풀커슨(Ford Fulkerson)
- S에서 T로 가는 경로를 BFS 로 찾으면 에드몬드 카프(Edmonds-Karp)

## 에드몬드 카프(Edmonds-Karp) 알고리즘
	

---
## Reference
[[Algorithm] 네트워크 유량(Network Flow)](https://engkimbs.tistory.com/353)  
[네트워크 플로우(Network Flow)](https://www.crocus.co.kr/741)  
[네트워크 유량(Network Flow) (수정: 2018-12-13)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220804885235)  
[네트워크플로우(Network flow) - 포드 풀커슨(Ford-Fulkerson) 알고리즘](https://coderkoo.tistory.com/4)  
[네트워크 플로우(network flow)](https://www.zerocho.com/category/Algorithm/post/5893405b588acb00186d39e0)  

