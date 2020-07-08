--- 
layout: single
classes: wide
title: "[풀이] 백준 1854 K번째 최단경로 찾기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 단방향 그래프에서 K 번째 최단경로를 찾아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Dijkstra
---  

# 문제
- 그래프를 구성하는 노드와 엣지의 정보가 주어진다.
- 주어진 그래프에서 K 번째에 해당하는 최단경로를 구하라.

## 입력
첫째 줄에 n, m, k가 주어진다. (1 ≤ n ≤ 1000, 0 ≤ m ≤ 2000000, 1 ≤ k ≤ 100) n과 m은 각각 김 조교가 여행을 고려하고 있는 도시들의 개수와, 도시 간에 존재하는 도로의 수이다.

이어지는 m개의 줄에는 각각 도로의 정보를 제공하는 세 개의 정수 a, b, c가 포함되어 있다. 이것은 a번 도시에서 b번 도시로 갈 때는 c의 시간이 걸린다는 의미이다. (1 ≤ a, b ≤ n. 1 ≤ c ≤ 1000)

도시의 번호는 1번부터 n번까지 연속하여 붙어 있으며, 1번 도시는 시작도시이다.

## 출력
n개의 줄을 출력한다. i번째 줄에 1번 도시에서 i번 도시로 가는 k번째 최단경로의 소요시간을 출력한다.

경로의 소요시간은 경로 위에 있는 도로들을 따라 이동하는데 필요한 시간들의 합이다. i번 도시에서 i번 도시로 가는 최단경로는 0이지만, 일반적인 k번째 최단경로는 0이 아닐 수 있음에 유의한다. 또, k번째 최단경로가 존재하지 않으면 -1을 출력한다.


## 예제 입력

```
5 10 2
1 2 2
1 3 7
1 4 5
1 5 6
2 4 2
2 3 4
3 4 6
3 5 8
5 2 4
5 4 1
```  

## 예제 출력

```
-1
10
7
5
14
```  

## 풀이
- 기존 [Dijkstra 알고리즘]({{site.baseurl}}{% link _posts/algorithm/2019-05-01-algorithm-concept-dijkstra.md %})을 활용한다.
- Dijkstra 알고리즘에서 기존에 사용하던 방문 노드의 순서를 거리의 오름차순으로 저장하던 우선순위 큐외에, 각 노드에 도착할 수 있는 거리를 내림차순으로 저장하는 우선순위 큐배열을 추가적으로 사용한다.
- Dijkstra 알고리즘에서 아래 조건에 따라 목적지에 대한 최단경로가 아니여서 제외되는 경로의 정보를 취합하면 문제를 해결 할 수 있다.
	- 노드의 우선순위 큐 원소의 개수가 k 보다 작으면 큐에 추가한다.
	- 노드의 우선순위 큐 원소의 개수가 k 와 같거나 크지만, 가장 큰 수가 현재 거리보다 작다면 우선순위 큐를 갱신해준다.
- PriorityQueue 가 아닌 Array 를 통해 푼거 추가하기

```java
public class Main {
    private StringBuilder result;
    private int nodeCount;
    private int edgeCount;
    private int k;
    private ArrayList<Node>[] adjListArray;
    
    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int start, end, cost;

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.adjListArray = new ArrayList[this.nodeCount + 1];
            this.edgeCount = Integer.parseInt(token.nextToken());
            this.k = Integer.parseInt(token.nextToken());
            this.result = new StringBuilder();

            for (int i = 1; i <= this.nodeCount; i++) {
                this.adjListArray[i] = new ArrayList<>();
            }

            for (int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                start = Integer.parseInt(token.nextToken());
                end = Integer.parseInt(token.nextToken());
                cost = Integer.parseInt(token.nextToken());

                this.adjListArray[start].add(new Node(end, cost));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // Dijkstra 우선순위 큐는 오름차순
        PriorityQueue<Node> queue = new PriorityQueue<>();
        PriorityQueue<Integer>[] nodeCostPriorityArray = new PriorityQueue[this.nodeCount + 1];
        ArrayList<Node> adjList;
        int adjListSize, adjCost;
        Node currentNode, adjNode;

        // k 번째에 대한 최단 거리를 저장하는 우선순위 큐는 내림차순
        Comparator<Integer> asc = new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                return o2 - o1;
            }
        };

        for (int i = 1; i <= this.nodeCount; i++) {
            nodeCostPriorityArray[i] = new PriorityQueue<>(asc);
        }

        // 시작 노드 설정
        nodeCostPriorityArray[1].offer(0);
        queue.offer(new Node(1, 0));

        while (!queue.isEmpty()) {
            currentNode = queue.poll();

            adjList = this.adjListArray[currentNode.node];
            adjListSize = adjList.size();

            for (int i = 0; i < adjListSize; i++) {
                adjNode = adjList.get(i);
                adjCost = adjNode.cost + currentNode.cost;

                if(nodeCostPriorityArray[adjNode.node].size() < this.k) {
                    // 노드의 최단거리 개수가 k 보다 작으면 2개의 큐에 추가
                    nodeCostPriorityArray[adjNode.node].offer(adjCost);
                    queue.offer(new Node(adjNode.node, adjCost));
                } else if(nodeCostPriorityArray[adjNode.node].peek() > adjCost) {
                    // 노드의 최단거리 개수가 k개가 만족 되었지만, 현재 k 번째 거리보다 더 작은 k번짹 거리가 있는 경우 큐 갱신
                    nodeCostPriorityArray[adjNode.node].poll();
                    nodeCostPriorityArray[adjNode.node].offer(adjCost);
                    queue.offer(new Node(adjNode.node, adjCost));
                }
            }
        }

        for (int i = 1; i <= this.nodeCount; i++) {
            if (nodeCostPriorityArray[i].size() >= this.k) {
                this.result.append(nodeCostPriorityArray[i].peek());
            } else {
                this.result.append(-1);
            }

            this.result.append("\n");
        }
    }

    class Node implements Comparable<Node> {
        private int node;
        private int cost;

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
[1854-K번째 최단경로 찾기](https://www.acmicpc.net/problem/1854)  
