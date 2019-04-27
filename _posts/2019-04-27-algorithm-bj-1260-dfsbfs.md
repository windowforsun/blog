--- 
layout: single
classes: wide
title: "[풀이] 백준 1260 DFS 와 BFS"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 노드와 간선 정보를 통해 DFS, BFS 탐색을 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - DFS
  - BFS
---  

# 문제
- 노드와 간선의 정보가 주어질 때 DFS와 BFS로 탐색한 결과를 구하라
- 방문 가능한 노드가 여러개일 경우 노드 번호가 작은 것을 먼저 방문한다.

## 입력
첫째 줄에 정점의 개수 N(1 ≤ N ≤ 1,000), 간선의 개수 M(1 ≤ M ≤ 10,000), 탐색을 시작할 정점의 번호 V가 주어진다. 다음 M개의 줄에는 간선이 연결하는 두 정점의 번호가 주어진다. 어떤 두 정점 사이에 여러 개의 간선이 있을 수 있다. 입력으로 주어지는 간선은 양방향이다.

## 출력
첫째 줄에 DFS를 수행한 결과를, 그 다음 줄에는 BFS를 수행한 결과를 출력한다. V부터 방문된 점을 순서대로 출력하면 된다.

## 예제 입력

```
4 5 1
1 2
1 3
1 4
2 4
3 4
```  

## 예제 출력

```
1 2 4 3
1 2 3 4
```  

## 예제 입력

```
5 5 3
5 4
5 2
1 2
3 4
3 1
```  

## 예제 출력

```
3 1 2 5 4
3 1 4 2 5
```  

## 예제 입력

```
1000 1 1000
999 1000
```  

## 예제 출력

```
1000 999
1000 999
```  

## 풀이
- DFS 와 BFS 는 그래프에서 가장 기본이 되는 탐색 방법이다.
- DFS 는 깊이우선 탐색으로 노드에서 연결된 다음 노드를 계속 방문하면서 탐색을 수행한다.
- BFS 는 너비우선 탐색으로 노드에서 연결된 모든 노드를 모두 방문하면서 탐색을 수행한다.
- 두 탐색 방법은 탐색이 되는 노드는 같으나 방문하는 순서는 다를 수 있다.

```java
public class Main {
    public TestHelper testHelper = new TestHelper();

    class TestHelper {
        private ByteArrayOutputStream out;

        public TestHelper() {
            this.out = new ByteArrayOutputStream();

            System.setOut(new PrintStream(this.out));
        }

        public String getOutput() {
            return this.out.toString().trim();
        }
    }

    // 출력 결과 저장
    private StringBuilder result;
    // 노드 수
    private int nodeCount;
    // 간선 수
    private int edgeCount;
    // 시작 노드
    private int startNode;
    // 노드와 간선 정보 매트릭스
    private boolean[][] matrix;
    // 방문 여부
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
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int node1, node2;

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.edgeCount = Integer.parseInt(token.nextToken());
            this.startNode = Integer.parseInt(token.nextToken());
            this.matrix = new boolean[this.nodeCount + 1][this.nodeCount + 1];
            this.isVisited = new boolean[this.nodeCount + 1];
            this.result = new StringBuilder();

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine());
                node1 = Integer.parseInt(token.nextToken());
                node2 = Integer.parseInt(token.nextToken());

                this.matrix[node1][node2] = true;
                this.matrix[node2][node1] = true;
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // DFS 탐색
        this.dfs(this.startNode);
        this.isVisited = new boolean[this.nodeCount + 1];
        this.result.setCharAt(this.result.length() - 1, '\n');
        // BFS 탐색
        this.bfs(this.startNode);
    }

    public void dfs(int node){
        // 방문 처리 및 결과값에 넣어주기
        this.isVisited[node] = true;
        this.result.append(node).append(" ");

        // 인접 노드가 방문하지 않았으면 방문하기
        for(int i = 1; i <= this.nodeCount; i++) {
            if(this.matrix[node][i] && !this.isVisited[i]) {
                this.dfs(i);
            }
        }
    }

    public void bfs(int node) {
        // BFS 에서 사용 할 큐
        LinkedList<Integer> queue = new LinkedList<>();
        int current;

        // 방문 처리 및 큐에 넣어주기
        this.isVisited[node] = true;
        queue.addLast(node);

        // 큐가 비어있을 때까지
        while(!queue.isEmpty()) {
            // 큐 에서 노드 방문 및 결고 값에 넣어주기
            current = queue.removeFirst();
            this.result.append(current).append(" ");

            // 인접한 노드가 방문하지 않았으면 큐에 넣어주기
            for(int i = 1; i <= this.nodeCount; i++) {
                if(this.matrix[current][i] && !isVisited[i]) {
                    this.isVisited[i] = true;
                    queue.addLast(i);
                }
            }
        }
    }
}
```  

---
## Reference
[1260-DFS와 BFS](https://www.acmicpc.net/problem/1260)  
