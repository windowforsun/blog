--- 
layout: single
classes: wide
title: "[풀이] 백준 1647 도시 분할 계획"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 그래프에서 SCC 집합을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - MST
  - Disjoint Set
---  

# 문제
- 집과 길로 연결된 한 마을을 아래의 조건으로 2개의 마을로 분리한다.
	- 한개의 마을은 최소 1개 이상의 집으로 구성된다.
	- 한 마을에서 모든 집 간의 경로는 존재해야 한다.
	- 한 마을에서 집 간의 경로는 최소 비용으로 구성되고, 그 외의 길은 없앨 수 있다.

## 입력
첫째 줄에 집의 개수N, 길의 개수M이 주어진다. N은 2이상 100,000이하인 정수이고, M은 1이상 1,000,000이하인 정수이다. 그 다음 줄부터 M줄에 걸쳐 길의 정보가 A B C 세 개의 정수로 주어지는데 A번 집과 B번 집을 연결하는 길의 유지비가 C (1 ≤ C ≤ 1,000)라는 뜻이다.

## 출력
첫째 줄에 없애고 남은 길 유지비의 합의 최솟값을 출력한다.

## 예제 입력

```
7 12
1 2 3
1 3 2
3 2 1
2 5 2
3 4 4
7 3 6
5 1 5
1 6 2
6 4 1
6 5 3
4 5 3
6 7 4
```  

## 예제 출력

```
8
```  

## 풀이
- 주어진 문제를 아래와 같이 정리 할 수 있다.
	- 비용이 가장 적은 경로로 모든 노드가 연결 되어야 한다.
	- A-B 간의 경로는 하나만 존재할 수 있으므로 총 경로의 수는 n-1 개가 될 수 있다.
	- 2개의 마을 사이에 있는 경로는 필요하지 않기 때문에 n-2 개의 경로만 구하면 된다.
	- MST 와 Disjoint Set 을 사용해서 n-2 에 대한 최소 비용의 경로를 구하면 된다.

```java
public class Main {
    private int result;
    private int nodeCount;
    private int edgeCount;
    private PriorityQueue<Edge> edgeList;
    private int[] rootNodeArray;
    private int[] rankArray;

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
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int node1, node2, cost;

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.edgeCount = Integer.parseInt(token.nextToken());
            // 비용의 크기순으로 관리하기 위해 우선순위 큐 사용
            this.edgeList = new PriorityQueue<>();
            this.rootNodeArray = new int[this.nodeCount + 1];
            this.rankArray = new int[this.nodeCount + 1];
            this.result = 0;

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine());
                node1 = Integer.parseInt(token.nextToken());
                node2 = Integer.parseInt(token.nextToken());
                cost = Integer.parseInt(token.nextToken());

                this.edgeList.add(new Edge(node1, node2, cost));
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 자신의 루트노드를 자신으로 만든다.
        for(int i = 1; i <= this.nodeCount; i++) {
            this.rootNodeArray[i] = i;
        }

        // n-2 개의 최단 경로를 구하기 위해
        int connectedCount = 2;
        boolean unionResult;
        Edge e;

        while(connectedCount < this.nodeCount) {
            e = this.edgeList.poll();

            unionResult = this.unionNodeWithCompression(e.node1, e.node2, e.cost);

            if(unionResult) {
                connectedCount++;
            }
        }
    }

    public int findNodeRoot(int node) {
        int root = this.rootNodeArray[node];

        if(node != root) {
            root = this.findNodeRoot(root);

            this.rootNodeArray[node] = root;
        }

        return root;
    }

    public boolean unionNodeWithCompression(int node1, int node2, int cost) {
        int node1Root = this.findNodeRoot(node1);
        int node2Root = this.findNodeRoot(node2);
        int bigRankRoot, smallRankRoot;

        if(node1Root == node2Root) {
            return false;
        }

        if(this.rankArray[node1Root] >= this.rankArray[node2Root]) {
            bigRankRoot = node1Root;
            smallRankRoot = node2Root;
        } else {
            bigRankRoot = node2Root;
            smallRankRoot = node1Root;
        }

        this.rootNodeArray[smallRankRoot] = bigRankRoot;
        this.result += cost;

        if(this.rankArray[bigRankRoot] == this.rankArray[smallRankRoot]) {
            this.rankArray[bigRankRoot]++;
        }

        return true;
    }

    class Edge implements Comparable<Edge>{
        public int node1;
        public int node2;
        public int cost;

        public Edge(int node1, int node2, int cost) {
            this.node1 = node1;
            this.node2 = node2;
            this.cost = cost;
        }

        @Override
        public int compareTo(Edge o) {
            return this.cost - o.cost;
        }
    }
}
```  

---
## Reference
[1647-도시 분할 계획](https://www.acmicpc.net/problem/1647)  
