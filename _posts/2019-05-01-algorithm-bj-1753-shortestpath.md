--- 
layout: single
classes: wide
title: "[풀이] 백준 1753 최단경로"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '방향 그래프가 주어질 때 한점에서 다른 모든 점까지의 최단거리를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Dijkstra
---  

# 문제
- 방향 그래프가 주어지면 시작 점에서 모든 정점까지의 최단거리를 구하라.

## 입력
첫째 줄에 정점의 개수 V와 간선의 개수 E가 주어진다. (1≤V≤20,000, 1≤E≤300,000) 모든 정점에는 1부터 V까지 번호가 매겨져 있다고 가정한다. 둘째 줄에는 시작 정점의 번호 K(1≤K≤V)가 주어진다. 셋째 줄부터 E개의 줄에 걸쳐 각 간선을 나타내는 세 개의 정수 (u, v, w)가 순서대로 주어진다. 이는 u에서 v로 가는 가중치 w인 간선이 존재한다는 뜻이다. u와 v는 서로 다르며 w는 10 이하의 자연수이다. 서로 다른 두 정점 사이에 여러 개의 간선이 존재할 수도 있음에 유의한다.

## 출력
첫째 줄부터 V개의 줄에 걸쳐, i번째 줄에 i번 정점으로의 최단 경로의 경로값을 출력한다. 시작점 자신은 0으로 출력하고, 경로가 존재하지 않는 경우에는 INF를 출력하면 된다.

## 예제 입력

```
5 6
1
5 1 1
1 2 2
1 3 3
2 3 4
2 4 5
3 4 6
```  

## 예제 출력

```
0
2
3
7
INF
```  

## 풀이
- 시작 정점에서 모든 정점까지의 최단 거리 즉 1:N 의 최단거리이기 때문에 Dijkstra 알고리즘을 이용해서 문제를 해결한다.
	- 자세한 내용은 [Dijkstra]({{site.baseurl}}{% link _posts/2019-05-01-algorithm-concept-dijkstra.md %}) 에서 확인 가능하다.

```java
public class Main {
    // 출력 결과
    private StringBuilder result;
    // 노드 수
    private int nodeCount;
    // 간선 수
    private int edgeCount;
    // 시작 노드
    private int startNode;
    // 단방향 연결 그래프 정보
    private ArrayList<Node>[] adjNodeListArray;
    // 최단 거리 배열
    private int[] costArray;

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
            int start, end, cost;

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.edgeCount = Integer.parseInt(token.nextToken());
            this.startNode = Integer.parseInt(reader.readLine());
            this.adjNodeListArray = new ArrayList[this.nodeCount + 1];
            this.costArray = new int[this.nodeCount + 1];
            this.result = new StringBuilder();

            Arrays.fill(this.costArray, Integer.MAX_VALUE);

            for(int i = 1; i <= this.nodeCount; i++) {
                this.adjNodeListArray[i] = new ArrayList<>();
            }

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine());
                start = Integer.parseInt(token.nextToken());
                end = Integer.parseInt(token.nextToken());
                cost = Integer.parseInt(token.nextToken());

                this.adjNodeListArray[start].add(new Node(end, cost));
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        this.dijkstra(this.startNode);

        for(int i = 1; i <= this.nodeCount; i++) {
            if(this.costArray[i] == Integer.MAX_VALUE) {
                this.result.append("INF");
            } else {
                this.result.append(this.costArray[i]);
            }
            this.result.append("\n");
        }
    }

    public void dijkstra(int startNode) {
        // 최소 비용 순으로 뺄 우선순위 큐
        PriorityQueue<Node> queue = new PriorityQueue<>();
        Node currentNode, adjNode;
        ArrayList<Node> adjNodeList;
        int size;

        // 시작 노드에서 시작노드의 비용은 0
        queue.offer(new Node(startNode, 0));

        // 큐가 빌때까지 반복
        while(!queue.isEmpty()) {
            // 혅재 방문할 노드
            currentNode = queue.poll();

            // 방문하지 않은 노드라면
            if(this.costArray[currentNode.node] == Integer.MAX_VALUE) {
                // 큐에 넣은 비용을 최단 거리로 설정
                this.costArray[currentNode.node] = currentNode.cost;

                // 현재 노드의 인접 노드들
                adjNodeList = this.adjNodeListArray[currentNode.node];
                size = adjNodeList.size();

                for(int i = 0; i < size; i++) {
                    adjNode = adjNodeList.get(i);

                    // 인접 노드가 방문하지 않았다면
                    if(this.costArray[adjNode.node] == Integer.MAX_VALUE) {
                        // 인접 노드의 최단 거리 = 현재 노드의 최소 비용 + 인접 노드의 비용
                        adjNode.cost += currentNode.cost;
                        queue.offer(adjNode);
                    }
                }
            }
        }
    }
    
    class Node implements Comparable<Node> {
        public int node;
        public int cost;

        public Node(int node, int cost) {
            this.node = node;
            this.cost = cost;
        }

        @Override
        public int compareTo(Node o) {
            return this.cost - o.cost;
        }
    }
}
```  

---
## Reference
[1753-최단경로](https://www.acmicpc.net/problem/1753)  
