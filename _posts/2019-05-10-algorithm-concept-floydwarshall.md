--- 
layout: single
classes: wide
title: "[Algorithm 개념] Floyd Warshall (플로이드 와샬) 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '모든 정점의 최단거리를 구하는 Floyd Warshall 알고리즘에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - BFS
  - DFS
  - Floyd Warshall
---  

## Floyd Warshall Algorithm 이란
- 플로이드 와샬 알고리즘은 그래프에서 모든 꼭지점 사이의 최단 경로의 거리를 구하는 알고리즘이다.
- 다익스트라 알고리즘을 모든 정점에서 수행한 것과 같은 알고리즘이지만 프로이드 와샬 알고리즘의 구현은 간단하다.
- 음의 가중치를 갖는 간선이라도, 사이클이 존재하지 않는다면 결과값을 도출해 낼 수 있다.
- 플로이드 와샬 알고리즘의 원리는 아래와 같다.
	- 두 정점을 잇는 최소 비용 경로는 경유지를 거치거나 거치지 않는 경로 중 하나이다.
	- 경유지를 거친다면 그것을 이루는 부분 경로 역시 최소 비용 경로어야 한다.

## Floyd Warshall Algorithm
- 플로이드 와샬 알고리즘은 3개의 반복문으로 이루어 진다.
- 가장 바깥쪽 반복문은 `A->..->Z` 에서 `...` 에 해당되는 거쳐가는 노드이다.
- 그 안쪽 반복문은 출발하는 노드 A에 해당된다.
- 가장 안쪽 반복문은 도착하는 노드 Z에 해당된다.
- 반복문 3개로 이루어져 있기 때문에 시간 복잡도는 O(n^3) 이 된다.
	
### 예시
- 아래와 같은 총 4개의 정점과 6개의 정점으로 이루어진 그래프가 있다.

![플로이드 와샬 1]({{site.baseurl}}/img/algorithm/concept-floydwarshall-1.png)

- 5*5 이차원 배열을 INF(무한대)값으로 초기화 해준다.
	- 사이클 간선을 뜻하는 `1*1, 2*2` .. 의 배열의 위체는 0값을 넣어준다.

행\열|1|2|3|4|5
---|---|---|---|---|---
1|0|INF|INF|INF|INF
2|INF|0|INF|INF|INF
3|INF|INF|0|INF|INF
4|INF|INF|INF|0|INF
5|INF|INF|INF|INF|0


- 위의 그래프에서 노드-간선->노드의 정보를 2차원 배열에 설정한다.


행\열|1|2|3|4|5
---|---|---|---|---|---
1|0|INF|INF|INF|4
2|-3|0|5|9|3
3|INF|INF|0|2|INF
4|INF|INF|INF|0|3
5|INF|INF|1|INF|0

- 그래프에서 경유(거쳐가는) 노드는 노란색, 출발 노드는 붉은색, 도착노드는 파란색으로 표시한다.
- 경유하는 간선이 있는 경우만 예시로 든다.
- 1번 노드를 경유하는 경우는 아래와 같다.
	- 2->5 로 바로가는 경로보다 2->1->5 로가는 경로가 더 짧기 때문에 2->5 로가는 경로 정보를 갱신한다.

		![플로이드 와샬 2]({{site.baseurl}}/img/algorithm/concept-floydwarshall-2.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|INF|INF|4
		2|-3|0|5|9|1
		3|INF|INF|0|2|INF
		4|INF|INF|INF|0|3
		5|INF|INF|1|INF|0

- 3번 노드를 경유하는 경우는 아래와 같다.
	- 2->4 의 경로 보다 2->3->4 의 경로가 더 짧기 때문에 2->4 의 경로 정보를 갱신한다.
	
		![플로이드 와샬 3]({{site.baseurl}}/img/algorithm/concept-floydwarshall-3.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|INF|INF|4
		2|-3|0|5|7|1
		3|INF|INF|0|2|INF
		4|INF|INF|INF|0|3
		5|INF|INF|1|INF|0
	
	- 5->4 경로는 INF 이고, 5->3->4 의 경로가 존재하므로 5->4의 경로 정보를 갱신한다.
	
		![플로이드 와샬 4]({{site.baseurl}}/img/algorithm/concept-floydwarshall-4.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|INF|INF|4
		2|-3|0|5|7|1
		3|INF|INF|0|2|INF
		4|INF|INF|INF|0|3
		5|INF|INF|1|3|0

- 4번 노드를 경유하는 경우는 아래와 같다.
	- 3->5 로 가는 경로는 INF 값이고, 3->4->5 로가는 경로가 존재 하므로 3->5 의 경로 정보를 갱신한다.
	
		![플로이드 와샬 5]({{site.baseurl}}/img/algorithm/concept-floydwarshall-5.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|INF|INF|4
		2|-3|0|5|7|1
		3|INF|INF|0|2|5
		4|INF|INF|INF|0|3
		5|INF|INF|1|3|0
		
- 5번 노드를 경유하는 경우는 아래와 같다.
	- 1->3 경로는 INF 이고, 1->5->3 의 경로가 존재하기 때문에 1->3 경로 정보를 갱신해준다.
	
		![플로이드 와샬 6]({{site.baseurl}}/img/algorithm/concept-floydwarshall-6.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|5|INF|4
		2|-3|0|5|7|1
		3|INF|INF|0|2|5
		4|INF|INF|INF|0|3
		5|INF|INF|1|3|0
		
	- 1->4 경로는 INF 이고, 1->5->4 의 경로가 존재하기 때문에 1->4 경로 정보를 갱신한다.
	
		![플로이드 와샬 7]({{site.baseurl}}/img/algorithm/concept-floydwarshall-7.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|5|7|4
		2|-3|0|5|7|1
		3|INF|INF|0|2|5
		4|INF|INF|INF|0|3
		5|INF|INF|1|3|0
		
	- 2->3 경로 보다 2->5->3 의 경로가 짧기 때문에 2->3 경로 정보를 갱신한다.
	
		![플로이드 와샬 8]({{site.baseurl}}/img/algorithm/concept-floydwarshall-8.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|5|7|4
		2|-3|0|4|7|1
		3|INF|INF|0|2|5
		4|INF|INF|INF|0|3
		5|INF|INF|1|3|0
		
	- 4->3 경로는 INF 이고, 4->5->3 의 경로가 존재하기 때문에 4->3 의 경로 정보를 갱신한다.
	
		![플로이드 와샬 9]({{site.baseurl}}/img/algorithm/concept-floydwarshall-9.png)
		
		행\열|1|2|3|4|5
		---|---|---|---|---|---
		1|0|INF|5|7|4
		2|-3|0|4|7|1
		3|INF|INF|0|2|5
		4|INF|INF|4|0|3
		5|INF|INF|1|3|0
		
- 최종적으로 주어진 그래프에서 모든 정점에 대한 최단거리는 아래와 같다.

행\열|1|2|3|4|5
---|---|---|---|---|---
1|0|INF|5|7|4
2|-3|0|4|7|1
3|INF|INF|0|2|5
4|INF|INF|4|0|3
5|INF|INF|1|3|0

### 소스코드

```java
public class Main {

    private final int MAX = 10000;
    private int[][] adjMatrix;

    public FloydWarshall() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        this.adjMatrix = new int[][]{
                {0, MAX, MAX, MAX, 4},
                {-3, 0, 5, 9, 3},
                {MAX, MAX, 0, 2, MAX},
                {MAX, MAX, MAX, 0, 3},
                {MAX, MAX, 1, MAX, 0}
        };
    }

    public void output() {
        for(int i = 0; i < 5; i++) {
            System.out.println(Arrays.toString(this.adjMatrix[i]));
        }
    }

    public void solution() {
        int middleValue;

        for(int middle = 0; middle < 5; middle++) {
            for(int start = 0; start < 5; start++) {
                for(int end = 0; end < 5; end++) {
                    middleValue = this.adjMatrix[start][middle] + this.adjMatrix[middle][end];

                    if(this.adjMatrix[start][end] > middleValue) {
                        System.out.println((start + 1) + ", " + (middle + 1) + ", " + (end + 1) + " = " + middleValue);
                        this.adjMatrix[start][end] = middleValue;
                    }
                }
            }
        }
    }
}
```

---
## Reference
[Floyd-Warshall(플로이드 와샬) 알고리즘](https://dongdd.tistory.com/107)  
[[알고리즘] ASP(3) - 플로이드 워셜 알고리즘 ( Flyod-Warshall Algorithm)](https://victorydntmd.tistory.com/108)  
[플로이드 와샬 알고리즘 :: 마이구미](https://mygumi.tistory.com/110)  
[플로이드-워셜 알고리즘(Floyd-Warshall Algorithm)](https://hsp1116.tistory.com/45)  

