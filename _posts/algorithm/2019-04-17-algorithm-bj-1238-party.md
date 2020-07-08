--- 
layout: single
classes: wide
title: "[풀이] 백준 1238 파티"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '집과 파티를 오고 갈때 가장 거리가 먼사람을 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Dijkstra
---  

# 문제
- N 명의 학생생과 M 개의 일방통행 도로가 있다.
- 특정 학생의 집에서 파티가 열리게 된다.
- 학생들은 집에서 파티를 가고나서, 다시 집으로 돌아간다. 이때 학생들은 최단 거리만 사용하게 된다.
- N 명의 학생들 중 오고 가는데 가장 많은 시간을 소비하는 학생이 누구인지 구하라.

## 입력
첫째 줄에 N(1 <= N <= 1,000), M(1 <= M <= 10,000), X가 공백으로 구분되어 입력된다. 두 번째 줄부터 M+1번째 줄까지 i번째 도로의 시작점, 끝점, 그리고 이 도로를 지나는데 필요한 소요시간 Ti가 들어온다.

모든 학생들은 집에서 X에 갈수 있고, X에서 집으로 돌아올 수 있는 데이터만 입력으로 주어진다.

## 출력
첫 번째 줄에 N명의 학생들 중 오고 가는데 가장 오래 걸리는 학생의 소요시간을 출력한다.

## 예제 입력

```
4 8 2
1 2 4
1 3 2
1 4 7
2 1 1
2 3 5
3 1 2
3 4 4
4 2 3
```  

## 예제 출력

```
10
```  

## 풀이
- 문제를 해결하기 위해 2가지의 비용을 구하게 된다.
	- 각 학생들의 집 -> 파티 까지의 최소 거리
	- 각 학생들의 파티 -> 집 까지의 최소 거리
- 문제에 제시되어 있는 것처럼 도로는 일방통행 이기 때문에 갈때와 올때의 거리가 다를 수 있기 때문에, 인접 배열을 2개를 만들어 사용한다.
	- 집 -> 파티 일때 인접 배열
	- 파티 -> 집 일때 인접 배열
- 위 의 두 데이터를 사용해서 최단 거리 알고리즘인 Dijkstra 을 2번 호출해서 최단 거리를 구한다.
	- 집 -> 파티의 경우 호출
	- 파티 -> 집의 경우 호출
- 최단 거리 알고리즘을 사용할때 시작노드는 항상 파티의 노드 값이다.
	- 집 -> 파티 의 경우 파티 노드로 올 수 있는 인접 노드의 거리를 갱신 하고 이를 다시 인접 배열로 넣어준다.
	- 그리고 이 과정을 반복하며 2 가지 상황에 대한 각 학생들의 최단 거기를 구하게 된다.
	- 파티 -> 집 또한 파티 노드로 시작한다. 하지만 인접 노드를 집 -> 파티 인접노드인 from -> to 의 방향을 바꾸게 되면 파티 -> 집으로 가게되는 인접 노드가 되어 이를 이용한다.

```java
public class Main {
    private int result;
    private int peopleCount;
    private int homeCount;
    private int partyHome;
    private ArrayList[] adjToHomeArrayList;
    private ArrayList[] adjToPartyArrayList;
    private int[] toHomeCostArray;
    private int[] toPartyCostArray;

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
            this.peopleCount = Integer.parseInt(token.nextToken());
            this.homeCount = Integer.parseInt(token.nextToken());
            this.partyHome = Integer.parseInt(token.nextToken());

            // 집으로 올 수 있는 인접 배열
            this.adjToHomeArrayList = new ArrayList[this.homeCount + 1];
            // 파티로 올 수 있는 인접 배열
            this.adjToPartyArrayList = new ArrayList[this.homeCount + 1];
            // 집에서 파티로 갈때 각 비용
            this.toHomeCostArray = new int[this.peopleCount + 1];
            // 파티에서 집으로 갈때 각 비용
            this.toPartyCostArray = new int[this.peopleCount + 1];

            for(int i = 1; i <= this.homeCount; i++) {
                this.adjToHomeArrayList[i] = new ArrayList();
                this.adjToPartyArrayList[i] = new ArrayList();
            }

            Arrays.fill(this.toHomeCostArray, Integer.MAX_VALUE);
            Arrays.fill(this.toPartyCostArray, Integer.MAX_VALUE);

            int from, to, cost;
            for(int i = 0; i < this.homeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                from = Integer.parseInt(token.nextToken());
                to = Integer.parseInt(token.nextToken());
                cost = Integer.parseInt(token.nextToken());

                this.adjToPartyArrayList[from].add(new Node(to, cost));
                this.adjToHomeArrayList[to].add(new Node(from, cost));
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
        // 파티 -> 집
        dijkstra(this.partyHome, this.adjToPartyArrayList, this.toPartyCostArray);
        // 집 -> 파티
        dijkstra(this.partyHome, this.adjToHomeArrayList, this.toHomeCostArray);

        // 집 -> 파티, 파티 -> 집 각 비용 중 최대 값을 결과값으로 설정
        for(int i = 1; i <= this.peopleCount; i++) {
            if(this.result < this.toPartyCostArray[i] + this.toHomeCostArray[i]) {
                this.result = this.toPartyCostArray[i] + this.toHomeCostArray[i];
            }
        }
    }

    public void dijkstra(int startNode, ArrayList[] adjArrayList, int[] costArray) {
        LinkedList<Node> queue = new LinkedList<>();
        Node currentNode, adjNode;
        ArrayList<Node> adjNodes;
        int size;

        // 파티 노드는 비용을 0 으로 설정하고 큐에 넣는다.
        costArray[startNode] = 0;
        queue.offer(new Node(startNode, costArray[startNode]));

        while(!queue.isEmpty()) {
            // 큐에서 노드하나를 뺀다.
            currentNode = queue.poll();

            // 현재 노드의 비용 보다 비용 배열이 크다면 갱신
            if(costArray[currentNode.node] >= currentNode.cost) {
                // 현재 노드로 올 수 있는 인접 노드
                adjNodes = adjArrayList[currentNode.node];
                size = adjNodes.size();

                for(int i = 0; i < size; i++) {
                    adjNode = adjNodes.get(i);

                    // 인접 노드의 비용이 현재 노드를 거쳐가는게 더 적다면 갱신
                    if(costArray[adjNode.node] > costArray[currentNode.node] + adjNode.cost) {
                        costArray[adjNode.node] = costArray[currentNode.node] + adjNode.cost;
                        queue.offer(adjNode);
                    }
                }
            }
        }
    }

    class Node {
        public int node;
        public int cost;

        public Node(int node, int cost) {
            this.node = node;
            this.cost = cost;
        }
    }
}
```  

---
## Reference
[1238-파티](https://www.acmicpc.net/problem/1238)  
