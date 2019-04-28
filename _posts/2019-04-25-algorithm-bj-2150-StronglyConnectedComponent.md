--- 
layout: single
classes: wide
title: "[풀이] 백준 2150 Strongly Connected Component"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 그래프에서 SCC 집합을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - SCC
  - DFS
  - Kosaraju’s 
---  

# 문제
- 방향 그래프가 주어졌을 때, 그 그래프를 SCC 들로 나누는 프로그램을 작성하라.
- 방향 그래프의 SCC 는 우선 정점의 최대 부분집합이다.
- 그 부분집합에 들어 있는 서로 다른 임의의 두 정점 u, v 에 대해서 u 에서 v 로 가는 경로와 v 에서 u 로 가는 경로가 모두 존재하는 경우를 말한다.

## 입력
첫째 줄에 두 정수 V(1≤V≤10,000), E(1≤E≤100,000)가 주어진다. 이는 그래프가 V개의 정점과 E개의 간선으로 이루어져 있다는 의미이다. 다음 E개의 줄에는 간선에 대한 정보를 나타내는 두 정수 A, B가 주어진다. 이는 A번 정점과 B번 정점이 연결되어 있다는 의미이다. 이때 방향은 A->B가 된다.

## 출력
첫째 줄에 SCC의 개수 K를 출력한다. 다음 K개의 줄에는 각 줄에 하나의 SCC에 속한 정점의 번호를 출력한다. 각 줄의 끝에는 -1을 출력하여 그 줄의 끝을 나타낸다. 각각의 SCC를 출력할 때 그 안에 속한 정점들은 오름차순으로 출력한다. 또한 여러 개의 SCC에 대해서는 그 안에 속해있는 가장 작은 정점의 정점 번호 순으로 출력한다.

## 예제 입력

```
7 9
1 4
4 5
5 1
1 6
6 7
2 7
7 3
3 7
7 2
```  

## 예제 출력

```
3
1 4 5 -1
2 3 7 -1
6 -1
```  

## 풀이
- 연결 그래프에서 SCC 를 구하는 방법은 코사라주 알고리즘과 타잔 알고리즘이 있는데, 이 중 코사라주 알고리즘을 사용하였다.
- 코사라주 알고리즘은 연결 그래프, 역방향 연결 그래프, 스택이 필요로 하다.
- 먼저 연결 그래프에서 DFS 탐색을 하며 더 이상 인접하거나 방문 할 수 있는 노드가 없으면 DFS 를 빠져나오면서 스택에 현재 노드를 넣는다.
- 스택에 모든 노드가 들어가면 역방향 연결 그래프를 기준으로 스택에서 노드를 하나씩 빼며 DFS 탐색을 수행하는 방식이다.
- 더 자세한 설명은 [SCC]({{site.baseurl}}{% link _posts/2019-04-25-algorithm-concept-scc.md %})에서 확인 가능하다.

```java
public class Main {
    private StringBuilder result;
    private int nodeCount;
    private int edgeCount;
    // 정방향 그래프
    private ArrayList[] graph;
    // 역방향 그래프
    private ArrayList[] reverseGraph;
    // SCC 집합의 개수에 따른 노드 번호 배열
    private ArrayList[] sccArray;
    // DFS 에서 사용하는 배열
    private LinkedList<Integer> stack;
    // 방문 플래그 배열
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
            StringTokenizer token = new StringTokenizer(reader.readLine());
            int start, end;

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.edgeCount = Integer.parseInt(token.nextToken());
            this.stack = new LinkedList<>();
            this.isVisited = new boolean[this.nodeCount + 1];
            this.sccArray = new ArrayList[this.nodeCount];
            this.graph = new ArrayList[this.nodeCount + 1];
            this.reverseGraph = new ArrayList[this.nodeCount + 1];

            for(int i = 0; i <= this.nodeCount; i++) {
                this.graph[i] = new ArrayList();
                this.reverseGraph[i] = new ArrayList();
            }

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                start = Integer.parseInt(token.nextToken());
                end = Integer.parseInt(token.nextToken());

                this.graph[start].add(end);
                this.reverseGraph[end].add(start);
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int currentNode, count = 0;
        ArrayList<Integer> scc;

        this.result = new StringBuilder();

        // 정방향 그래프 DFS 탐색 하며 스택에 노드 넣어주기
        for(int i = 1; i <= this.nodeCount; i++) {
            if(!this.isVisited[i]) {
                this.dfs(i);
            }
        }

        this.isVisited = new boolean[this.nodeCount + 1];

        // 스택에서 노드를 빼면서 역방향 그래프 DFS 탐색
        while(!this.stack.isEmpty()) {
            currentNode = this.stack.removeFirst();

            // 스택에서 빼내어진 노드를 DFS 탐색 수행 할때마다 SCC 집합이 하나씩 생성됨
            if(!this.isVisited[currentNode]) {
                this.sccArray[count] = new ArrayList();
                this.reverseDfs(currentNode, count);
                count++;
            }

        }

        // SCC 집합 노드 정렬
        for(int i = 0; i < count; i++) {
            scc = this.sccArray[i];
            Collections.sort(scc);
        }

        // SCC 집합 리스트 정렬
        Arrays.sort(this.sccArray, 0, count, new Comparator<ArrayList>() {
            @Override
            public int compare(ArrayList o1, ArrayList o2) {
                int result = 0;

                if(o1 == null) {
                    result = 1;
                } else if(o2 == null) {
                    result = -1;
                } else {
                    result = (int)o1.get(0) - (int)o2.get(0);
                }

                return result;
            }
        });

        this.result.append(count).append("\n");

        for(int i = 0; i < count; i++) {
            scc = this.sccArray[i];
            Collections.sort(scc);

            for(int node : scc) {
                this.result.append(node).append(" ");
            }
            this.result.append(-1).append("\n");
        }
    }

    public void dfs(int node) {
        this.isVisited[node] = true;

        ArrayList<Integer> adjNodeList = this.graph[node];

        for(int adjNode : adjNodeList) {
            if(!this.isVisited[adjNode]) {
                this.dfs(adjNode);
            }
        }

        // 현재 node 에서 모든 노드를 방문하고 나면 스택에 넣음
        this.stack.addFirst(node);
    }

    public void reverseDfs(int node, int count) {
        this.isVisited[node] = true;

        ArrayList<Integer> adjNodeList = this.reverseGraph[node];

        for(int adjNode : adjNodeList) {
            if(!this.isVisited[adjNode]) {
                this.reverseDfs(adjNode, count);
            }
        }

        // SCC 집합 개수를 기준으로 해당되는 node 추가
        this.sccArray[count].add(node);
    }
}
```  

---
## Reference
[2150-Strongly Connected Component](https://www.acmicpc.net/problem/2150)  
