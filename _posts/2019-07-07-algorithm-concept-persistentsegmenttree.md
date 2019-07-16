--- 
layout: single
classes: wide
title: "[Algorithm 개념] Persistent Segment Tree"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: 'Persistent Segment Tree 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
  - Persistent Segment Tree
use_math : true
---  

## Persistent Segment Tree 란
- Persistent Segment Tree 는 가상으로 N 개의 Segment Tree 를 두는데 공간 복잡도는 $O(\log_2 N)$ 인 Segment Tree 를 만드는 방법이다.
- Persistent Segment Tree 는 시간 복잡도
	- 트리 생성 : $O(N \log_2 N)$
	- 쿼리 수행 : $O(\log_2 N)$
	- 값 변경 : $O(\log_2 N)$
	
## Persistent Segment Tree 의 원리


## 예시

---
## Reference
[Persistent Segment Tree | Set 1 (Introduction)](https://www.geeksforgeeks.org/persistent-segment-tree-set-1-introduction/)  
[Persistent segment trees – Explained with spoj problems](https://blog.anudeep2011.com/persistent-segment-trees-explained-with-spoj-problems/)  
[Persistent Segment Tree](https://blog.myungwoo.kr/100)  
[배열 기반 Persistent Segment Tree](https://junis3.tistory.com/8)  
[Persistent Segment Tree](https://softgoorm.tistory.com/12)  
[Persistent Segment Tree](https://hongjun7.tistory.com/64)  
