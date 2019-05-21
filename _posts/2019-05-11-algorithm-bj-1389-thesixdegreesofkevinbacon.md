--- 
layout: single
classes: wide
title: "[풀이] 백준 1389 케빈 베이컨의 6단계 법칙"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '케빈 베이턴의 수가 가장 적은 사람을 구하자'
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

# 문제
- 케빈 베이컨의 6단계 법칙이란
	- 지구에 있는 모든 사람들은 최대 6단계 이내에서 서로 아는 사람으로 연결 될 수 있다.
	- 케빈 베이컨 게임은 임의의 두 사람이 최소 몇 단계 만에 이어질 수 있는지 계산하는 게임이다.
- 사람인 노드와 친구관계인 간선이 주어질 때 케빈 베이컨의 수가 가장 적은 사람을 구하라.

## 입력
첫째 줄에 유저의 수 N (2 ≤ N ≤ 100)과 친구 관계의 수 M (1 ≤ M ≤ 5,000)이 주어진다. 둘째 줄부터 M개의 줄에는 친구 관계가 주어진다. 친구 관계는 A와 B로 이루어져 있으며, A와 B가 친구라는 뜻이다. A와 B가 친구이면, B와 A도 친구이며, A와 B가 같은 경우는 없다. 친구 관계는 중복되어 들어올 수도 있으며, 친구가 한 명도 없는 사람은 없다. 또, 모든 사람은 친구 관계로 연결되어져 있다.

## 출력
첫째 줄에 BOJ의 유저 중에서 케빈 베이컨의 수가 가장 작은 사람을 출력한다. 그런 사람이 여러 명일 경우에는 번호가 가장 작은 사람을 출력한다.

## 예제 입력

```
5 5
1 3
1 4
4 5
4 3
3 2
```  

## 예제 출력

```
3
```  

## 풀이
- 주어진 모든 노드의 최단 거리를 구하면 된다.
- 그래프에서 모든 노드에 대한 최단 거리를 구하는 알고리즘은 [플로이드 와샬 알고리즘]({{site.baseurl}}{% link _posts/2019-05-10-algorithm-concept-floydwarshall.md %}) 이 있다.
- 위 알고리즘을 모든 노드에 대한 최단 거리를 구하고, 이중 가장 적은 값을 가진 가장 적은 노드 번호가 결과값이 된다.

```java
public class Main {
    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    final int MAX = 10000000;
    private int result;
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

        int sum, maxSum = MAX;
        this.result = MAX;

        for(int i = this.nodeCount; i > 0; i--) {
            // 현재 노드에서 다른 노드까지의 가중치 총합
            sum = 0;
            for(int j = 1; j <= this.nodeCount; j++) {
                if(i != j) {
                    sum += this.adjMatrix[i][j];
                }
            }

            // 최소값을 결과값으로
            if(maxSum >= sum) {
                this.result = i;
                maxSum = sum;
            }
        }
    }
}
```  

---
## Reference
[1389-케빈 베이컨의 6단계 법칙](https://www.acmicpc.net/problem/1389)  
