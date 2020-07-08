--- 
layout: single
classes: wide
title: "[풀이] 백준 2188 축사 배정"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '소들이 원하는 방으로 배정할 수 있는 최대개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Network Flow
  - Bipartite Matching
---  

# 문제
- 여러 마리의 소와 여러 개의 축사가 있다.
- 한 칸에는 한마리의 소만 들어가고, 소들은 의망하는 축사 번호가 있을때 최대한으로 축사에 들어갈 수 있는 소의 수를 구하라.

## 입력
첫째 줄에 소의 마릿수 N과 축사의 개수 M이 주어진다. (1<=N<=200, 1<=M<=200)

둘째 줄부터 N개의 줄에는 N마리의 소들이 각자 들어가기 원하는 축사에 대한 정보가 주어진다. i번째 소가 들어가기 원하는 축사의 수 Si(0<=Si<=M)가 먼저 주어지고, 그 이후에 Si개의 축사 번호가 주어진다. 한 축사가 2번 이상 입력되는 경우는 없다.

## 출력
첫째 줄에 축사에 들어갈 수 있는 소 마릿수의 최댓값을 출력한다.

## 예제 입력

```
5 5
2 2 5
3 2 3 4
2 1 5
3 1 2 5
1 2
```  

## 예제 출력

```
4
```  

## 풀이
- 이분 매칭과 네트워크 유량을 이용해서 문제를 해결 할 수 있다.
- [Network Flow 알고리즘]({{site.baseurl}}{% link _posts/algorithm/2019-06-07-algorithm-concept-networkflow.md %}) 을 활용하여 문제를 해결하였다.
- 주어지는 데이터 소에 대한 정보, 축사에 대한 정보에서 추가적으로 시작점(source), 도착지(sink) 노드를 둔다.
- 시작점에서는 모든 소들로 갈 수 있고, 소들은 자신이 원하는 축사로 갈 수 있고, 모든 축사는 도착지로 갈 수 있다.
- 위 그래프에서 네트워크 유량의 용량을 모두 1로 해두면 문제를 해결 할 수 있다.

```java
public class Main {
   private int result;
   private int max;
   private int cowCount;
   private int houseCount;
   private int source;
   private int sink;
   private ArrayList<Integer>[] isAdjListArray;
   private int[][] capacity;
   private int[][] flow;
   private int[] prev;

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

	   try {
		   StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
		   int size, house;

		   this.cowCount = Integer.parseInt(token.nextToken());
		   this.houseCount = Integer.parseInt(token.nextToken());
		   this.max = this.cowCount + this.houseCount + 2;
		   this.source = this.cowCount + this.houseCount + 1;
		   this.sink = this.source + 1;
		   this.isAdjListArray = new ArrayList[this.max + 1];
		   this.capacity = new int[this.max + 1][this.max + 1];
		   this.flow = new int[this.max + 1][this.max + 1];

		   for(int i = 0; i <= this.max; i++) {
			   this.isAdjListArray[i] = new ArrayList<>();
		   }

		   for(int i = 1; i <= this.cowCount; i++) {
			   token = new StringTokenizer(reader.readLine(), " ");
			   size = Integer.parseInt(token.nextToken());

			   for(int j = 0; j < size; j++) {
				   house = Integer.parseInt(token.nextToken()) + this.cowCount;

				   this.isAdjListArray[i].add(house);
				   this.isAdjListArray[house].add(i);
				   this.capacity[i][house] += 1;
			   }
		   }

		   for(int i = 1; i <= this.cowCount; i++) {
			   this.isAdjListArray[this.source].add(i);
			   this.isAdjListArray[i].add(this.source);
			   this.capacity[this.source][i] += 1;
		   }

		   for(int i = this.cowCount + 1; i <= this.cowCount + this.houseCount; i++) {
			   this.isAdjListArray[i].add(this.sink);
			   this.isAdjListArray[this.sink].add(i);
			   this.capacity[i][this.sink] += 1;
		   }
	   } catch (Exception e) {
		   e.printStackTrace();
	   }
   }

   public void output() {
	   System.out.println(this.result);
   }

   public void solution() {
	   int minFlow;
	   this.prev = new int[this.max + 1];

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
		   this.prev = new int[this.max + 1];
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
[2188-축사 배정](https://www.acmicpc.net/problem/2188)  
