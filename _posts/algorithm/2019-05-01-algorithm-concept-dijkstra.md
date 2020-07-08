--- 
layout: single
classes: wide
title: "[Algorithm 개념] Dijkstra 최단 경로 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '한 정점에서 모든 정점까지의 최단거리를 구하는 Dijkstra 알고리즘에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - BFS
  - Dijkstra
---  

## Dijkstra Algorithm 이란
- 다익스트라 알고리즘은 간선의 비용이 있는 단방향 연결 그래프에서 1:N 최단 거리를 구하는 알고리즘이다.
	- 1:N 이라는 것은 한 정점에서 다른 모든 정점까지의 최단 거리를 뜻한다.
	- N:N 의 최단거리는 플로이드 워셜 O(N^3) 인 알고리즘을 사용할 수 있다.
- 다익스트라 알고리즘의 기본 개념은 `최단 거리는 최단 거리로 이루어져 있다.` 라는 Greedy(탐욕) 적인 접근 방식을 사용하였다.
	
## Dijkstra Algorithm 특징
- 가중치 값에 음수가 있다면 다익스트라 알고리즘을 사용해서 1:N 최단 거리를 구할 수 없다.
	- 음수가 포함될 경우 벨만포드 알고리즘 O(VE) 혹은 SPFA O(VE) 를 사용하여 1:N 최단 거리를 구해야 한다.
- 1:N 의 경우에 대한 최단 거리 이므로, 단방향 연결 그래프 상에서 전체적인(N:N) 노드에 대한 최선은 아닐 수 있다. 

## Dijkstra Algorithm
- 다음과 같은 데이터 구조가 필요하다.
	1. 시작 노드에서 부터 각 노드 까지의 최단 거리를 저장하는 배열
	1. 단방향 연결 그래프의 노드와 엣지 연결 정보를 저장하는 이차원 배열 혹은 이차원 리스트
	1. 다음 방문할 노드를 저장할 우선순위 큐 혹은 힙(다음 방문 노드는 가중치가 가장 적은 노드이어야 한다.)
- 알고리즘 순서
	1. 시작 정점의 최단 거리 배열의 값을 0을 넣어주고 우선순위 큐에 넣어 준다.
	1. 우선순위 큐에서 가중치가 가장 적은 노드를 빼내고, 해당 노드를 방문하지 않았거나, 최단 거리가 아니라면 최단 거리 배열의 값을 갱신해준다.
	1. 방문한 노드에서 인접한 노드에서 방문하지 않은 노드를 우선순위 큐에 넣어준다.
	1. 우선순위 큐가 비었을 때까지 위 동작을 반복한다.
	
### 예시
- 아래와 같이 주어진 단방향 연결 그래프가 있다.
- 방문한 정점은 붉은색으로 표시하고, 우선순위 큐에 들어간 간선 및 노드는 간선에 붉은 색으로 표시한다.
- 최단 거리와 우선순위에 대한 정보는 아래 표로 표시하도록 한다.
- 시작 노드는 1번으로 한다.
- 1번 노드를 시작으로 우선순위 큐에 (1, 0) 1번 노드에서 1번노드의 가중치는 0이라는 값을 넣어준다.

![dijkstra ex]({{site.baseurl}}/img/algorithm/concept-dijkstra-1.png)

노드|1|2|3|4|5
---|---|---|---|---|---
최단거리|INF|INF|INF|INF|INF

우선순위|1
---|---
(노드,가중치)|(1,0)|

- 우선순위 큐에서 가중치가 가장 적은 노드인 1번 노드를 빼내고 1번 노드에 대한 방문 처리와 인접 노드들을 우선순위 큐에 넣어준다.
	- 우선순위 큐에 인접 노드를 넣어 줄때 현재 노드(1번)의 가중치 + 인접 노드(3, 4번)의 가중치를 더한 값을 넣어준다.
	
![dijkstra ex]({{site.baseurl}}/img/algorithm/concept-dijkstra-2.png)

노드|1|2|3|4|5
---|---|---|---|---|---
최단거리|0|INF|INF|INF|INF

우선순위|1|2
---|---|---
(노드,가중치)|(4,3)|(3,6)

- 큐에서 가중치가 가장 적은 4번 노드를 빼내어 방문 처리 및 인접 노드에 대한 가중치 갱신하여 큐에 넣어준다.

![dijkstra ex]({{site.baseurl}}/img/algorithm/concept-dijkstra-3.png)

노드|1|2|3|4|5
---|---|---|---|---|---
최단거리|0|INF|INF|3|INF

우선순위|1|2|3
---|---|---|---
(노드,가중치)|(2,4)|(3,5)|(3,6)

- 큐에서 가중치가 가장 적은 2번 노드를 빼내어 방문 처리를 해준다.
	- 인접 노드 처리는 2번 인접 노드인 1번은 이미 방문했으면서 가중치가 2번을 거쳐가는 것보다 적기 때문에 갱신하지 않고, 5번 노드에 대한 갱신 작업만 해준다.

![dijkstra ex]({{site.baseurl}}/img/algorithm/concept-dijkstra-4.png)

노드|1|2|3|4|5
---|---|---|---|---|---
최단거리|0|4|INF|3|INF

우선순위|1|2|
---|---|---
(노드,가중치)|(3,5)|(3,6)

- 큐에서 가중치가 가장 적은 3번 노드를 빼내어 방문 처리를 해준다.
	- 3번 노드에서 인접한 4번 노드 또한 이미 방문 했으면서 가중치가 3번 노드를 거쳐 가는 것보다 적기 때문에 갱신하지 않는다.
- 큐에서 다시 노드를 빼내면 이미 방문한 노드이면서 가중치가 더큰 3번 노드이므로 아무런 작업을 수행하지 않는다.

![dijkstra ex]({{site.baseurl}}/img/algorithm/concept-dijkstra-5.png)

노드|1|2|3|4|5
---|---|---|---|---|---
최단거리|0|4|5|3|INF

우선순위|
---|---
(노드,가중치)|

- 결과적으로 1번 노드에 대한 다른 노드들의 최단거리는 아래와 같고 5번 노드는 1번 노드에서 방문할 수 없기 때문에 무한대 값을 가지게 된다.

	노드|1|2|3|4|5
	---|---|---|---|---|---
	최단거리|0|4|5|3|INF
	
- 우선순위 큐나 힙을 사용할 경우 다익스트라 알고리즘의 시간복잡도는 O(ElogV) 가 된다.
	- 노드마다 인접한 간선들을 모드 검사하는 작업과 우선순위 큐에 원소를 넣고 삭제하는 작업에서 각 노드는 한번씩 방문되고, 모든 간선은 한번씩 검사되므로 모든 간선을 검서하는데 O(E) 의 시간이 걸린다.
	- 큐에 들어가 O(E) 에서 큐에서 빼내고 다시 검사하는 작업을 추가하면 최종적으로 O(ElogE) 가되는데 그래프 간선의 개수는 노드의 개수보다 작기 때문에 O(logE) = O(logV) 로 볼수 있어서 최종적으로 O(ElogV) 가된다.

### 소스코드

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
        this.dijkstraWithPriorityQueue(this.startNode);
//        this.dijkstra(this.startNode);

        for (int i = 1; i <= this.nodeCount; i++) {
            if (this.costArray[i] == Integer.MAX_VALUE) {
                this.result.append("INF");
            } else {
                this.result.append(this.costArray[i]);
            }
            this.result.append("\n");
        }
    }

    public void dijkstraWithPriorityQueue(int startNode) {
        // 최소 비용 순으로 뺄 우선순위 큐
        PriorityQueue<Node> queue = new PriorityQueue<>();
        Node currentNode, adjNode;
        ArrayList<Node> adjNodeList;
        int size;

        // 시작 노드에서 시작노드의 비용은 0
        queue.offer(new Node(startNode, 0));

        // 큐가 빌때까지 반복
        while (!queue.isEmpty()) {
            // 혅재 방문할 노드
            currentNode = queue.poll();

            // 방문하지 않은 노드라면
            if (this.costArray[currentNode.node] == Integer.MAX_VALUE) {
                // 큐에 넣은 비용을 최단 거리로 설정
                this.costArray[currentNode.node] = currentNode.cost;

                // 현재 노드의 인접 노드들
                adjNodeList = this.adjNodeListArray[currentNode.node];
                size = adjNodeList.size();

                for (int i = 0; i < size; i++) {
                    adjNode = adjNodeList.get(i);

                    // 인접 노드가 방문하지 않았다면
                    if (this.costArray[adjNode.node] == Integer.MAX_VALUE) {
                        // 인접 노드의 최단 거리 = 현재 노드의 최소 비용 + 인접 노드의 비용
                        adjNode.cost += currentNode.cost;
                        queue.offer(adjNode);
                    }
                }
            }
        }
    }

    public void dijkstra(int startNode) {
        LinkedList<Node> queue = new LinkedList<>();
        Node currentNode, adjNode;
        ArrayList<Node> adjList;
        int size;

        queue.offer(new Node(startNode, 0));

        while(!queue.isEmpty()) {
            currentNode = queue.poll();

            if(this.costArray[currentNode.node] > currentNode.cost) {
                this.costArray[currentNode.node] = currentNode.cost;

                adjList = this.adjNodeListArray[currentNode.node];
                size = adjList.size();

                for(int i = 0; i < size; i++) {
                    adjNode = adjList.get(i);

                    if(this.costArray[adjNode.node] > adjNode.cost + currentNode.node) {
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

## Dijkstra Algorithm 참고사항
- 우선순위 큐를 사용하지 않을 경우 결과는 같지만 성능에 차이가 있다.
	- 다익스트라 알고리즘의 속도는 큐에서 노드를 꺼내오는 획수와 우선순위 큐의 갱신 횟수이다.
	- 가중치가 가장 적은 노드를 먼저 방문해야 연산의 횟수를 줄일 수 있다.
	- 하지만 정점의 수가 적거나, 간선의 수가 매우 많을 경우 우선순위 큐를 사용할 때가 더 성능적으로 안좋을 수도 있다.
- 다익스트라 알고리즘을 구현할때는 상황에 따라 우선순위 큐에 대한 조건이 달라질 수 있다.
	1. 큐에 넣은 순서
	1. 정점까지의 거리 먼/가까운
	1. 간선의 가중치 큰/적은
	1. 정점 번호 큰/적은
- 음의 가중치일 때는 사용못하는 이유는 이전 노드까지 계산해둔 최소 거리값이 최소라고 보장이 되지 않기 때문이다.
	- 다익스트라 알고리즘은 정점을 지날때마다 가중치는 증가한다는 가정에서 구현된 알고리즘이다.
	- 위의 전제에서 음의 가중치가 있을 경우 정점을 지날때마다 가중치가 감소할 수 있다.
	- 음의 가중치의 경우는 위에서 언급했던 것처럼 벨만포드 알고리즘을 사용한다.

---
## Reference
[다익스트라 알고리즘(Dijkstra's Algorithm)](https://jason9319.tistory.com/307)  
[Dijkstra의 최단 경로 알고리즘 기본개념과 알고리즘](https://mattlee.tistory.com/50)  
[13. 다익스트라(Dijkstra) 최단 거리 알고리즘](https://makefortune2.tistory.com/26)  
[다익스트라 알고리즘(Dijkstra)](https://sungjk.github.io/2016/05/13/Dijkstra.html)  
