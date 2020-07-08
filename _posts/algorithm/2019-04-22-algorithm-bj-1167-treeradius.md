--- 
layout: single
classes: wide
title: "[풀이] 백준 1167 트리의 지름"
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
트리가 입력으로 주어진다. 먼저 첫 번째 줄에서는 트리의 정점의 개수 V가 주어지고 (2≤V≤100,000)둘째 줄부터 V개의 줄에 걸쳐 간선의 정보가 다음과 같이 주어진다. (정점 번호는 1부터 V까지 매겨져 있다고 생각한다)

먼저 정점 번호가 주어지고, 이어서 연결된 간선의 정보를 의미하는 정수가 두 개씩 주어지는데, 하나는 정점번호, 다른 하나는 그 정점까지의 거리이다. 예를 들어 네 번째 줄의 경우 정점 3은 정점 1과 거리가 2인 간선으로 연결되어 있고, 정점 4와는 거리가 3인 간선으로 연결되어 있는 것을 보여준다. 각 줄의 마지막에는 -1이 입력으로 주어진다. 주어지는 거리는 모두 10,000 이하의 자연수이다.

## 출력
첫째 줄에 트리의 지름을 출력한다.

## 예제 입력

```
5
1 3 2 -1
2 4 4 -1
3 1 2 4 3 -1
4 2 4 3 3 5 6 -1
5 4 6 -1
```  

## 예제 출력

```
11
```  

## 풀이
- 트리에서 특정 노드 A 를 기준으로 생각해본다.
	- A 노드의 반지름은 인접한 노드에서 Leaf 노드까지의 거리이다.
	- A 노드가 가질 수 있는 반지름의 개수는 인접한 노드 수와 동일 하다.
	- A 노드의 지름은 A 노드의 반지름 중 가장 큰 2개를 더한 값이다.
- 여기서 인접 노드란, 부모노드를 제외한 자식 노드들이 될 수 있다.
- 특정 노드에서 DFS 탐색을 시작하게 되는데 이 노드를 Root 노드라고 가정한다.
	- 모든 노드는 Root 노드가 될 수 있다.
- DFS 탐색을 수행하며 계속 자식노드로 방문하고 해당 노드의 가장 큰 반지름 2개를 구해 지름으로 만들면서 모든 노드를 방문 할때 까지 탐색을 수행한다.

```java
public class Main {
    private int result;
    private int nodeCount;
    private ArrayList[] adjArrayList;
    private int[] radiusArray;
    private boolean[] isVisited;

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
            this.adjArrayList = new ArrayList[this.nodeCount + 1];
            this.radiusArray = new int[this.nodeCount + 1];
            this.isVisited = new boolean[this.nodeCount + 1];
            StringTokenizer token;
            String str;
            int parent, child, cost;

            for(int i = 1; i <= this.nodeCount; i++) {
                this.adjArrayList[i] = new ArrayList();
            }

            for(int i = 1; i <= this.nodeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                parent = Integer.parseInt(token.nextToken());

                while(!(str = token.nextToken()).equals("-1")) {
                    child = Integer.parseInt(str);
                    cost = Integer.parseInt(token.nextToken());

                    this.adjArrayList[parent].add(new int[]{child, cost});
                }
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution(){
        // 항상 1번 노드에서 부터 시작해서 DFS 탐색을 통해 모든 모드의 최대 트리의 지름 값을 구한다.
        this.dfs(1);

        for(int i = 1; i <= this.nodeCount; i++) {
            this.result = Integer.max(this.result, this.radiusArray[i]);
        }
    }

    // node 의 최대 트리 지름을 구하는 DFS
    public int dfs(int node) {
        this.isVisited[node] = true;
        int halfRadius1 = 0, halfRadius2 = 0, nextNode, nextWeight, currentHalfRadius;

        // 현재 노드의 인접 노드들
        ArrayList<int[]> adjs = this.adjArrayList[node];

        for(int[] adj : adjs) {
            // 다음 인접 노드
            nextNode = adj[0];
            // 다음 인접 노드의 가중치
            nextWeight = adj[1];

            // 방문 한적이 없다면
            if(!this.isVisited[nextNode]) {
                // 현재 노드의 반지름 = 다음 인접 노드의 최대 반지름 + 다음 인접 노드의 가중치
                currentHalfRadius = this.dfs(nextNode) + nextWeight;

                // 현재 노드의 반지름 중에서 가장 큰 2개의 반지름을 구한다.
                if(currentHalfRadius > halfRadius1) {
                    halfRadius2 = halfRadius1;
                    halfRadius1 = currentHalfRadius;
                } else if(currentHalfRadius > halfRadius2) {
                    halfRadius2 = currentHalfRadius;
                }
            }
        }

        this.radiusArray[node] = halfRadius1 + halfRadius2;

        // 현재 노드의 반지름 중 가장 큰 반지름 을 반환 한다.
        return halfRadius1;
    }
}
```  

---
## Reference
[1167-트리의 지름](https://www.acmicpc.net/problem/1167)  
