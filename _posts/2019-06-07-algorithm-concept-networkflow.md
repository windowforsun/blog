--- 
layout: single
classes: wide
title: "[Algorithm 개념] 네트워크 플로우(Network Flow) 포드 풀커슨(Ford-Fulkerson), 에드몬드 카프(Edmonds-Karp) 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '포드 풀커슨, 에드몬드 카프 알고리즘으로 네트워크 플로우를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Network Flow
  - Max Flow
  - Ford Fulkerson
  - Edmonds Karp
use_math : true
---  

## 네트워크 플로우란
- 그래프에서 각 노드들 간의 용량(Capacity)이 정의되어 있을때, 시작점(Source)에서 도착점(Target, Sink) 까지 흐를 수 있는 최대 유량 구하는 것이다.
- DFS 를 사용해서 포드 풀커슨(Ford-Fulkerson) 알고리즘으로 해결 할 수 있다.
- BFS 를 사용해서 에드몬드 카프(Edmonds-Karp) 알고리즘으로 해결 할 수 있다.

## 네트워크 플로우의 용어
- 소스 Source(S)
	- 시작 위치를 의미한다.
- 싱크  Sink, Target(T)
	- 끝 위치를 의미한다.
- 용량 Capacity(c)
	- 유량이 흐를 수 있는 크기를 의미한다.
- 유량 Flow(f)
	- 간선에 흐르는 현재 유량의 크기를 의미한다.
- 잔류 유량 Residual Flow
	- Capacity - Flow 의 값으로 현재 간선에 흐를 수 있는 유량이 얼마인지 의미한다. ($c(u,v)-f(u,v)$)
- $c(u,v)$
	- u 에서 v(u->v) 로 흐를 수 있는 간선의 용량을 의미한다.
- $f(u,v)$ 
	- u 에서 v(u->v) 로 흐르는 실제 유량을 의미한다.
	
## 네트워크 플로우의 성질
- 특정 경로(ex, s->1->4->5->t)로 유량을 보낼 때는 경로에 포함된 간선 중 가장 작은 용량의 간선에 의해 용량이 결정된다.
- $f(u,v) \le c(u,v)$ : 용량 제한 속성
	- 흐르는 양(f)은 그 간선의 용량(c)를 넘을 수 없다.
- $f(u,v) = -f(v,u)$ :  유량의 대칭성
	- 1->2 로 3의 유량을 흘려 보냈다는 것은, 2->1 로 -3의 유량을 흘려 보냈다는 것과 같은 의미이다.
- $\sum_{} f(u,v) = 0$ : 유량의 보존
	- 소스와 싱크를 제외한 모든 노드에서 들어오는 유량의 합과 나가는 유량의 합은 같아야 한다.
	
## 네트워크 플로우 알고리즘 원리
- S에서 T로 가는 아직 잔류 용량이 있는 경로를 찾는다.
- 경로 중에서 $c(u,v)-f(u,v)$ 값이 최소인 간선을 찾고(현재 경로에서 흘려 보낼 수 있는 유량), 이 값을 F 라고 한다.
- 경로상의 모든 간선에 F 만큼 유량을 추가한다. 
	- $f(u,v)$ += $F$
	- $f(v,u)$ -= $F$
- 위의 연산을 더 이상 잔류 유량이 있는 경로가 없을 때 까지 반복한다.
- S에서 T로 가는 경로를 DFS 로 찾으면 포드 풀커슨(Ford Fulkerson)
- S에서 T로 가는 경로를 BFS 로 찾으면 에드몬드 카프(Edmonds-Karp)

## 예시
- 아래와 같은 그래프가 있다.

![network flow 1]({{site.baseurl}}/img/algorithm/concept-networkflow-1.png)

- 간선의 정보를 Flow/Capacity 로 다시 표현하면 아래와 같다.
	
![network flow 2]({{site.baseurl}}/img/algorithm/concept-networkflow-2.png)

- S-> .. ->T 까지 잔류 유량이 있는 경로를 하나 찾는다. (아무 경로나 상관없다)
	- 찾은 경로의 간선은 붉은 색으로 표시한다.
	- 해당 경로에는 3의 유량을 흘려 보낼 수 있다.
	- result = 3
	
![network flow 3]({{site.baseurl}}/img/algorithm/concept-networkflow-3.png)

- 기억해야 할점은 3의 유량을 위의 경로로 흘려 보냈을 때 역방향에 대한 유량정보도 함께 갱신된다.
	- f(1,2) += 3, f(2,1) -= 3

- S-> .. ->T 까지 잔류 유량이 있는 다른 경로를 하나 찾는다.
	- 해당 경로에는 4의 유량을 흘려 보낼 수 있다.
	- result = 7

![network flow 4]({{site.baseurl}}/img/algorithm/concept-networkflow-4.png)

- S-> .. ->T 까지 잔류 유량이 있는 다른 경로를 하나 찾는다.
	- 해당 경로에는 1의 유량을 흘려 보낼 수 있다.
	- result = 8

![network flow 5]({{site.baseurl}}/img/algorithm/concept-networkflow-5.png)

- S-> .. ->T 까지 잔류 유량이 있는 다른 경로를 하나 찾는다.
	- 해당 경로에는 1의 유량을 흘려 보낼 수 있다.
	- result = 9

![network flow 6]({{site.baseurl}}/img/algorithm/concept-networkflow-6.png)

- 최종적으로 위 그래프에서 S->T 까지 보낼 수 있는 최대 유량은 9가 된다.

![network flow 7]({{site.baseurl}}/img/algorithm/concept-networkflow-7.png)

- 실제 코드를 돌렸을 때 경로의 선정과 각 간선들에서 흐르는 유량의 값은 다를 수 있지만 최대 유량은 변함없다.

## 에드몬드 카프(Edmonds-Karp) 소스코드 (BFS)

```java
public class Main {
    private int MAX;
    private int result;
    private int source;
    private int sink;
    private ArrayList<Edge>[] adjEdgeListArray;

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        this.MAX = 7;
        // 시작 노드
        this.source = 1;
        // 끝 노드
        this.sink = 7;
        this.adjEdgeListArray = new ArrayList[MAX + 1];

        for(int i = 0; i <= MAX; i++) {
            this.adjEdgeListArray[i] = new ArrayList<>();
        }

        this.add(1, 2, 3);
        this.add(1, 4, 3);
        this.add(1, 3, 4);
        this.add(2, 4, 1);
        this.add(2, 5, 3);
        this.add(3, 6, 5);
        this.add(4, 3, 2);
        this.add(4, 5, 3);
        this.add(4, 6, 4);
        this.add(5, 7, 4);
        this.add(6, 7, 5);
    }

    public void add(int start, int end, int capacity) {
        Edge edge = new Edge(end, capacity);
        Edge reverseEdge = new Edge(start, 0);

        edge.reverseEdge = reverseEdge;
        reverseEdge.reverseEdge = edge;
        this.adjEdgeListArray[start].add(edge);
        this.adjEdgeListArray[end].add(reverseEdge);
    }

    public void output() {
        System.out.println(this.result);

        for(int i = 1; i <= MAX; i++) {
            System.out.println(Arrays.toString(this.flow[i]));
        }
    }

    public void solution() {
        LinkedList<Integer> queue = new LinkedList<>();
        ArrayList<Edge> adjList;
        Edge[] path;
        int[] prev;
        int current, adjListSize, minFlow;
        Edge adj;

        // 잔류 유량이 있는 경로가 있을 때 까지 반복
        while(true) {
            // 해당 노드가 연결된 이전 노드를 저장 1->2 라면 prev[2] = 1
            prev = new int[MAX + 1];
            // 한 루프의 경로 정보 저장 path[Node] = Edge
            path = new Edge[MAX + 1];
            queue.clear();
            // 시작 노드 큐에 저장
            queue.offer(this.source);

            while(!queue.isEmpty()) {
                current = queue.poll();

                adjList = this.adjEdgeListArray[current];
                adjListSize = adjList.size();

                for(int i = 0; i < adjListSize; i++) {
                    adj = adjList.get(i);

                    // 아직 경로에 선정이 되지 않았으면
                    if(prev[adj.endNode] == 0) {
                        // 잔류 유량이 있을 때만
                        if(adj.getResidualFlow() > 0) {
                            queue.offer(adj.endNode);
                            prev[adj.endNode] = current;
                            path[adj.endNode] = adj;

                            // 끝노드 싱크에 도착하면 루프 종료
                            if(adj.endNode == this.sink) {
                                break;
                            }
                        }
                    }
                }
            }

            // 더 이상 잔류 유량이 있는 경로가 없으면 루프 종료
            if(prev[this.sink] == 0) {
                break;
            }

            // 현재 경로의 최소 유량 저장
            minFlow = Integer.MAX_VALUE;
            for(int node = this.sink; node != this.source; node = prev[node]) {
                minFlow = Integer.min(minFlow, path[node].getResidualFlow());
            }

            // 현재 경로의 최소 유량 만큼 유량을 흘려 보냄
            for(int node = this.sink; node != this.source; node = prev[node]) {
                path[node].setFlow(minFlow);
            }

            // 총 유량 증가
            this.result += minFlow;
        }
    }

    class Edge {
        // 간선이 도착하는 노드
        public int endNode;
        // 간선의 용량
        public int capacity;
        // 간선의 유량
        public int flow;
        // 간선의 역방향 간선
        public Edge reverseEdge;

        public Edge(int endNode, int capacity) {
            this.endNode = endNode;
            this.capacity = capacity;
            this.flow = 0;
            this.reverseEdge = null;
        }

        // 잔류 유량 반환
        public int getResidualFlow() {
            return this.capacity - this.flow;
        }

        // 유량을 흘려 보냄
        public void setFlow(int minFlow) {
            this.flow += minFlow;
            this.reverseEdge.flow -= minFlow;
        }
    }
}
```  

## 포드 풀커슨(Ford-Fulkerson) 소스코드 (DFS)

```java
public class MainMatrix {
    private int MAX;
    private int result;
    private int source;
    private int sink;
    // 실제 흐르는 유량
    private int[][] flow;
    // 간선의 흐를 수 있는 용량
    private int[][] capacity;
    // 인접 여부
    private ArrayList<Integer>[] isAdjListArray;
    private int[] prev;

    public MainMatrix() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new MainMatrix();
    }

    public void input() {
        this.MAX = 7;
        // 시작 노드
        this.source = 1;
        // 끝 노드
        this.sink = 7;
        this.flow = new int[MAX + 1][MAX + 1];
        this.capacity = new int[MAX + 1][MAX + 1];
        this.isAdjListArray = new ArrayList[MAX + 1];

        for(int i = 0; i <= MAX; i++) {
            this.isAdjListArray[i] = new ArrayList<>();
        }

        this.add(1, 2, 3);
        this.add(1, 4, 3);
        this.add(1, 3, 4);
        this.add(2, 4, 1);
        this.add(2, 5, 3);
        this.add(3, 6, 5);
        this.add(4, 3, 2);
        this.add(4, 5, 3);
        this.add(4, 6, 4);
        this.add(5, 7, 4);
        this.add(6, 7, 5);
    }

    public void add(int start, int end, int capacity) {
        this.isAdjListArray[start].add(end);
        this.isAdjListArray[end].add(start);
        this.capacity[start][end] += capacity;
//        this.capacity[end][start] = capacity;
    }

    public void output() {
        System.out.println(this.result);

        for(int i = 1; i <= MAX; i++) {
            System.out.println(Arrays.toString(this.flow[i]));
        }
    }

    public void solution() {
        int minFlow;
        this.prev = new int[this.MAX + 1];

        // 잔류 유량이 있는 경로가 있을 때까지 반복
        while(this.dfs(this.source)) {

            // 현재 경로의 최소 유량 저장
            minFlow = Integer.MAX_VALUE;
            for(int node = this.sink; node != this.source; node = this.prev[node]) {
                minFlow = Integer.min(minFlow, this.capacity[this.prev[node]][node] - this.flow[prev[node]][node]);
            }

            // 현재 경로의 최소 유량 만큼 유량을 흘려 보냄
            for(int node = this.sink; node != this.source; node = this.prev[node]) {
                this.flow[this.prev[node]][node] += minFlow;
                this.flow[node][this.prev[node]] -= minFlow;
            }

            // 총 유량 증가
            this.result += minFlow;
            this.prev = new int[this.MAX + 1];
        }
    }

    public boolean dfs(int start) {
        boolean result = false;

        if(start == this.sink) {
            // 싱크에 도착했으면 종료
            result = true;
        } else {
            // start 노드와 연결되어 있는 노드들
            ArrayList<Integer> adjList = this.isAdjListArray[start];
            int adj, adjListSize = adjList.size();

            for(int i = 0; i < adjListSize; i++) {
                adj = adjList.get(i);

                // 경로에 설정되지 않았으면서
                if(this.prev[adj] == 0) {
                    // 잔류 유량이 있으면
                    if(this.capacity[start][adj] - this.flow[start][adj] > 0) {
                        // 경로에 추가
                        this.prev[adj] = start;

                        // 추가되 노드부터 다시 경로 탐색
                        if(this.dfs(adj)) {
                            result = true;
                        }
                    }
                }
            }
        }

        return result;
    }
}
```

## 추가사항
- 단방향일경우 `reserveEdge` 의 용량에 0을 넣어주고 양방향일 경우에는 순방향 간선의 용량과 동일한 용량을 넣어 주면된다.

---
## Reference
[[Algorithm] 네트워크 유량(Network Flow)](https://engkimbs.tistory.com/353)  
[네트워크 플로우(Network Flow)](https://www.crocus.co.kr/741)  
[네트워크 유량(Network Flow) (수정: 2018-12-13)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220804885235)  
[네트워크플로우(Network flow) - 포드 풀커슨(Ford-Fulkerson) 알고리즘](https://coderkoo.tistory.com/4)  
[네트워크 플로우(network flow)](https://www.zerocho.com/category/Algorithm/post/5893405b588acb00186d39e0)  
[[Algorithm] 네트워크플로우(network flow) ( 2188 축사 배정)](https://lyhyun.tistory.com/57)  

