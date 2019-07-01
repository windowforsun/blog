--- 
layout: single
classes: wide
title: "[Algorithm 개념] 세그먼트 트리(Segment Tree)"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '세그먼트 트리를 구현하고 특정 구간의 값을 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
use_math : true
---  

## 세그먼트 트리(Segment Tree)란
- 이름 그대로 구간을 보존하고 있는 트리이다.
- 주로 완전 이진트리 혹은 포화 이진트리 형태를 사용한다.
- 주워진 쿼리에 대해 빠르게 응답하기 위한 자료구조이다.
	- 특정 구간의 합을 구한다.
	- 특정 구간의 최대, 최소 값을 구한다.
- 쿼리를 수행할 배열을 트리로 변환하여 관리하는데, 포인터 형식의 트리가 아닌 배열을 통해 트리를 구성한다.
	
## 세그먼트 트리의 특징
- 대체적으로 3개의 연산으로 구성되고, 시간 복잡도는 아래와 같다.
	- 세그먼트 트리 구성 : $O(2^{\lceil \log_2 N \rceil + 1})$ (이후 추가 설명)
	- 쿼리 수행(구간 합, 최대, 최소 ..) : $O(\log_2 N)$
	- 쿼리 수행 배열의 값 변경 : $O(\log_2 N)$
- 리프 노드는 쿼리를 수행 할 배열에 해당하는 데이터를 의미한다.
- 리프 노드를 제외한 다른 노드(루트 노드 포함)는 왼쪽 자식과 오른쪽 자식에 대한 쿼리 결과를 저장한다.

![그림1]({{site.baseurl}}/img/algorithm/concept-segmenttree-1.png)



---
## Reference
[세그먼트 트리(Segment Tree) (수정: 2019-02-12)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220791986409)  
[세그먼트 트리(Segment Tree)](https://www.crocus.co.kr/648)  
[Segment Tree를 활용한 2D Range Update/Query](http://www.secmem.org/blog/2018/12/23/segment-tree-2d-range-update-query/)  
[세그먼트 트리 (Segment Tree)](https://www.acmicpc.net/blog/view/9)  
