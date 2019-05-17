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

## Floyd Warshall Algorithm
- 플로이드 와샬 알고리즘은 3개의 반복문으로 이루어 진다.
- 가장 바깥쪽 반복문은 `A->..->Z` 에서 `...` 에 해당되는 거쳐가는 노드이다.
- 그 안쪽 반복문은 출발하는 노드 A에 해당된다.
- 가장 안쪽 반복문은 도착하는 노드 Z에 해당된다.
- 반복문 3개로 이루어져 있기 때문에 시간 복잡도는 O(n^3) 이 된다.
	
### 예시


### 소스코드

```java
public class Main {
    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    final int MAX = 10000000;
    private int nodeCount;
    private int edgeCount;
    private int[][] adjMatrix;


    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int a, b;

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.edgeCount = Integer.parseInt(token.nextToken());
            this.adjMatrix = new int[this.nodeCount + 1][this.nodeCount + 1];

            for (int i = 1; i <= this.nodeCount; i++) {
                for (int j = 1; j <= this.nodeCount; j++) {
                    if (i == j) {
                        this.adjMatrix[i][j] = 0;
                    } else {
                        this.adjMatrix[i][j] = MAX;
                    }
                }

            }

            for (int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                a = Integer.parseInt(token.nextToken());
                b = Integer.parseInt(token.nextToken());

                this.adjMatrix[a][b] = 1;
                this.adjMatrix[b][a] = 1;
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int value;

        // 거쳐가는 중간 노드
        for(int middle = 1; middle <= this.nodeCount; middle++) {
            // 시작 노드
            for(int start = 1; start <= this.nodeCount; start++) {
                // 도착 노드
                for(int end = 1; end <= this.nodeCount; end++) {
                    // (시작 - 중간) + (중간 + 도착)
                    value = this.adjMatrix[start][middle] + this.adjMatrix[middle][end];
                    // 중간노드를 거쳐가는게 더 가중치가 적다면 갱신
                    if(this.adjMatrix[start][end] > value) {
                        this.adjMatrix[start][end] = value;
                    }
                }
            }
        }

        for(int i = 0; i <= this.nodeCount; i++) {
            System.out.println(Arrays.toString(this.adjMatrix[i]));
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

