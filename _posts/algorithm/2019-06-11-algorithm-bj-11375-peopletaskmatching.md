--- 
layout: single
classes: wide
title: "[풀이] 백준 11375 열혈강호"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '직원과 일 목록이 있을때 최대 몇개의 일을 처리할 수 있는지 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Bipartite Matching
---  

# 문제
- N명의 직원과 M개의 일이 있다.
- 한 개의 일을 담당하는 사람은 한 명이다.
- 각 직원들은 할 수 있는 일의 목록이 있다.
- 이때 최대 몇개의 일을 할 수 있는지 구하라.

## 입력
첫째 줄에 직원의 수 N과 일의 개수 M이 주어진다. (1 ≤ N, M ≤ 1,000)

둘째 줄부터 N개의 줄의 i번째 줄에는 i번 직원이 할 수 있는 일의 개수와 할 수 있는 일의 번호가 주어진다.

## 출력
첫째 줄에 강호네 회사에서 할 수 있는 일의 개수를 출력한다.

## 예제 입력

```
5 5
2 1 2
1 1
2 2 3
3 3 4 5
1 1
```  

## 예제 출력

```
4
```  

## 풀이
- 이분 매칭의 기본적인 문제이므로 [이분 매칭 알고리즘]({{site.baseurl}}{% link _posts/algorithm/2019-06-10-algorithm-concept-bipartitematching.md %}) 을 활용해서 직원과 담당해야 하는 일을 매칭 시킬 수 있다.

```java
public class Main {
    private int result;
    private int peopleCount;
    private int taskCount;
    private ArrayList<Integer>[] isAdjListArray;
    private boolean[] isVisited;
    private int[] matched;


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
            int count;

            this.peopleCount = Integer.parseInt(token.nextToken());
            this.taskCount = Integer.parseInt(token.nextToken());
            this.isAdjListArray = new ArrayList[this.peopleCount + 1];
            this.isVisited = new boolean[this.peopleCount + this.taskCount + 1];
            this.matched = new int[this.taskCount + 1];

            for(int i = 1; i <= this.peopleCount; i++) {
                this.isAdjListArray[i] = new ArrayList<>();
                token = new StringTokenizer(reader.readLine(), " ");
                count = Integer.parseInt(token.nextToken());

                for(int j = 0; j < count; j++) {
                    this.isAdjListArray[i].add(Integer.parseInt(token.nextToken()) + this.peopleCount);
                }
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 각직원 들이 할 수 있는 일 중 하나와 매칭 시킨다.
        for(int i = 1; i <= this.peopleCount; i++) {
            Arrays.fill(this.isVisited, false);

            // 처리 가능한 일이 있을 경우 결과 값 증가
            if(this.dfs(i)) {
                this.result++;
            }
        }
    }

    public boolean dfs(int start) {
        boolean result = false;

        // 아직 방문하지 않은 노드일 경우에만
        if(!this.isVisited[start]) {
            this.isVisited[start] = true;

            // 현재 직원이 할 수 있는 일의 목록
            ArrayList<Integer> adjList = this.isAdjListArray[start];
            int adj, adjListSize = adjList.size();

            for(int i = 0; i < adjListSize; i++) {
                adj = adjList.get(i) - this.peopleCount;

                // 할 수 있는 일을 담당하는 사람이 없거나
                // 담당하는 사람이 있지만, 그 사람이 다른일 담당이 가능 할 경우
                if(this.matched[adj] == 0 || this.dfs(this.matched[adj])) {
                    // 현재 일을 현재 직원이 담당
                    this.matched[adj] = start;
                    result = true;
                    break;
                }
            }
        }

        return result;
    }
}
```  

---
## Reference
[11375-열혈강호](https://www.acmicpc.net/problem/11375)  
