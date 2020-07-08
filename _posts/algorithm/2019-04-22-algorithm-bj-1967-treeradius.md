--- 
layout: single
classes: wide
title: "[풀이] 백준 1967 트리의 지름"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '특정 트리가 주어 졌을 때 트리의 최대 지름을 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Tree
  - DFS
---  

# 문제
- 트리의 지름이란, 트리에서 임의의 두 점 사이의 거리 중 가장 긴 것을 말한다.
- 특정 트리가 주어 졌을 때, 이 트리의 지름을 구하라.

## 입력
파일의 첫 번째 줄은 노드의 개수 n(1 ≤ n ≤ 10,000)이다. 둘째 줄부터 n번째 줄까지 각 간선에 대한 정보가 들어온다. 간선에 대한 정보는 세 개의 정수로 이루어져 있다. 첫 번째 정수는 간선이 연결하는 두 노드 중 부모 노드의 번호를 나타내고, 두 번째 정수는 자식 노드를, 세 번째 정수는 간선의 가중치를 나타낸다. 간선에 대한 정보는 부모 노드의 번호가 작은 것이 먼저 입력되고, 부모 노드의 번호가 같으면 자식 노드의 번호가 작은 것이 먼저 입력된다. 루트 노드의 번호는 항상 1이라고 가정하며, 간선의 가중치는 100보다 크지 않은 양의 정수이다.

## 출력
첫째 줄에 트리의 지름을 출력한다.

## 예제 입력

```
12
1 2 3
1 3 2
2 4 5
3 5 11
3 6 9
4 7 1
4 8 7
5 9 15
5 10 4
6 11 6
6 12 10
```  

## 예제 출력

```
45
```  

## 풀이
- 부모의 노드를 저장하는 배열과 비용을 저장하는 배열 2개를 둔다.
- Leaf 노드에서 부터 시작해서 Root 노드 까지 오르며 자신 node 의 cost + 부모의 node 의 cost 를 반복 수행 한다.
- cost 갱신 작업이 끝나면 모든 노드를 탐색하며 부모 노드 중 가장 cost 큰를 찾고 이를 통해 부모의 최대 반지름 값을 구해 나간다.

```java
public class Main {
    private int result;
    private int nodeCount;
    private int[] parentNodeArray;
    private int[] costArray;
    private int[] updateCostArray;
    private int[] maxNodeRadius;
    private int[] maxNodeHalfRadius;
    private boolean[] isNotLeaf;

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.nodeCount = Integer.parseInt(reader.readLine());
            this.parentNodeArray = new int[this.nodeCount + 1];
            this.costArray = new int[this.nodeCount + 1];
            this.isNotLeaf = new boolean[this.nodeCount + 1];
            this.maxNodeRadius = new int[this.nodeCount + 1];
            this.updateCostArray = new int[this.nodeCount + 1];
            this.maxNodeHalfRadius = new int[this.nodeCount + 1];

            StringTokenizer token;
            int parent, child, cost;

            for(int i = 1; i < this.nodeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                parent = Integer.parseInt(token.nextToken());
                child = Integer.parseInt(token.nextToken());
                cost = Integer.parseInt(token.nextToken());

                this.parentNodeArray[child] = parent;
                this.costArray[child] = cost;
                this.updateCostArray[child] = cost;
                this.isNotLeaf[parent] = true;
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        this.result = 0;
        int parent, radius;

        // Leaf 노드라면 부모로 올라가면서 cost를 갱신한다.
        for(int i = 1; i <= this.nodeCount; i++) {
            if(!this.isNotLeaf[i]) {
                this.updateParentCost(i);
            }
        }

        // 갱신된 비용 배열을 통해 부모 노드 의 자식중 가장 큰 cost 를 찾으면서 부모의 지름값 구하기
        for(int i = 1; i <= this.nodeCount; i++) {
            parent = this.parentNodeArray[i];

            if(parent != 0) {
                // 현재 부모의 지름 = 현재 부모의 자식 중 가장 큰 cost + 현재 노드의 업데이트된 cost
                if(this.maxNodeRadius[parent] < (radius = this.updateCostArray[i] + this.maxNodeHalfRadius[parent])) {
                    this.maxNodeRadius[parent] = radius;
                }

                // 현재 부모의 지름이 결과 값 보다 크다면 갱신
                if(this.maxNodeRadius[parent] > this.result) {
                    this.result = this.maxNodeRadius[parent];
                }

                // 현재 부모의 가장 큰 자식 cost 가 현재 노드의 업데이트된 cost 보다 작다면 갱신
                if(this.updateCostArray[i] > this.maxNodeHalfRadius[parent]) {
                    this.maxNodeHalfRadius[parent] = this.updateCostArray[i];
                }
            }
        }
    }

    // node 의 부모로 타고 오르면서 cost 를 갱신
    public void updateParentCost(int node) {
        int parent = this.parentNodeArray[node], updatedCost;

        // 루트 노드이면
        if(parent == 0) {
            return;
        }

        // 현재 부모의 업데이트된 cost 가 현재 노드의 업데이트 된 cost + 부모의 cost 보다 작으면 갱신
        if(this.updateCostArray[parent] < (updatedCost = this.updateCostArray[node] + this.costArray[parent])) {
            this.updateCostArray[parent] = updatedCost;
        }

        // 계속 해서 부모로 오르기
        this.updateParentCost(parent);
    }
}
```  

---
## Reference
[1967-트리의 지름](https://www.acmicpc.net/problem/1967)  
