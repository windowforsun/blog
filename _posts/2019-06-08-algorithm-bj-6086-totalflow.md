--- 
layout: single
classes: wide
title: "[풀이] 백준 6086 최대 유량"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 그래프에서 최대 유량을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Network Flow
  - Edmonds Karp
---  

# 문제
- 그래프의 정보에서 간선에 흐를수 있는 물의 용량이 주어진다.
- A-Z 까지 이물을 흘려 보내야 할때 최대유량을 구하라
- 물은 양방향으로 흐를 수 있다.

## 입력
첫째 줄에 정수 N (1 ≤ N ≤ 700)이 주어진다. 둘째 줄부터 N+1번째 줄까지 파이프의 정보가 주어진다. 첫 번째, 두 번째 위치에 파이프의 이름(알파벳 대문자 또는 소문자)이 주어지고, 세 번째 위치에 파이프의 용량이 주어진다.

## 출력
첫째 줄에 A에서 Z까지의 최대 유량을 출력한다.

## 예제 입력

```
5
A B 3
B C 3
C D 5
D Z 4
B Z 6
```  

## 예제 출력

```
3
```  

## 풀이
- 기본적인 네트워크 유량 문제로 [Network Flow 알고리즘]({{site.baseurl}}{% link _posts/2019-06-07-algorithm-concept-networkflow.md %}) 를 통해 문제를 해결 할 수 있다.
- 주의 해야 할 점은 간선이 단방향이 아니라 양방향이라는 점과 같은 파이프가 2번 이상 올 수 있다는 점을 유의해야 한다.

```java
public class MainDFS {
    private final int ALPABET_COUNT = 'z' - 'a';
    private final int MAX = (ALPABET_COUNT + 1) * 2;
    private int result;
    private int edgeCount;
    private int source;
    private int sink;
    private ArrayList<Integer>[] isAdjListArray;
    private int[][] flow;
    private int[][] capacity;
    private int[] prev;

    public static void main(String[] args) {
        new MainDFS();
    }

    public MainDFS() {
        this.input();
        this.solution();
        this.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            StringTokenizer token;
            int start, end, capacity;
            this.edgeCount = Integer.parseInt(reader.readLine());
            this.isAdjListArray = new ArrayList[this.MAX + 1];
            this.flow = new int[this.MAX + 1][this.MAX + 1];
            this.capacity = new int[this.MAX + 1][this.MAX + 1];
            this.source = ('A' - 'A') + 1;
            this.sink = ('Z' - 'A') + 1;

            for(int i = 0; i <= MAX; i++) {
                this.isAdjListArray[i] = new ArrayList<>();
            }

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                start = (int)(token.nextToken().charAt(0) - 'A');
                end = (int)(token.nextToken().charAt(0) - 'A');
                capacity = Integer.parseInt(token.nextToken());

                if(start > ALPABET_COUNT) {
                    start -= 6;
                }

                if(end > ALPABET_COUNT) {
                    end -= 6;
                }

                start += 1;
                end += 1;

                this.isAdjListArray[start].add(end);
                this.isAdjListArray[end].add(start);
                this.capacity[start][end] += capacity;
                this.capacity[end][start] += capacity;
            }


        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
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

---
## Reference
[6086-최대 유량](https://www.acmicpc.net/problem/6086)  
