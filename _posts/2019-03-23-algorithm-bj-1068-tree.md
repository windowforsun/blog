--- 
layout: single
classes: wide
title: "[풀이] 백준 1068 트리"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '트리의 리프노드 개수를 알아내자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - DFS
  - Tree
  - Graph
---  

# 문제
- 특정 트리가 주어졌을 때, 트리의 노드중 하나를 제거한다.
- 남은 트리에서 리프노드의 개수를 구하라.
	- 리프 노드는 자식의 개수가 0인 노드를 뜻한다.

## 입력
첫째 줄에 트리의 노드의 개수 N이 주어진다. 
N은 50보다 작거나 같은 자연수이다. 
둘째 줄에는 0번 노드부터 N-1번 노드까지, 각 노드의 부모가 주어진다. 
만약 부모가 없다면 (루트) -1이 주어진다. 
셋째 줄에는 지울 노드의 번호가 주어진다.

## 출력
첫째 줄에 입력으로 주어진 트리에서 입력으로 주어진 노드를 지웠을 때, 리프 노드의 개수를 출력한다.

## 예제 입력

```
5
-1 0 0 1 1
2
```  

## 예제 출력

```
2
```  

# 풀이
- 문제에서 제시되는 트리는 이진트리가 아닐수 있다.
- 루트 노드가 항상 0이 아닐 수도 있다.
- 2개의 배열을 통해 입력받은 데이터를 사용할 수 있도록 구성했다.
	- parentArray : 1차원 배열로 부모의 노드 번호를 저장한다.
	- isAdjacent : 2차원 배열로 두 노드가 연결되었는지 저장하며, 부모노드에 연결된 자식 노드를 저장한다.
- 삭제되는 노드를 입력 받으면 2개의 배열을 업데이트 해준다.
	- parentArray : 부모의 노드 번호를 현재 문제에서 유효하지 않는(-100) 값으로 수정한다.
	- isAdjacent : 삭제되는 노드가 자식으로 가지는 값, 부모에 등록된 값을 모두 false 로 갱신한다.
- 입력 받고 갱신한 2개의 배열을 기반으로 DFS 탐색을 수행한다.
	- parentArray 을 기준으로 해당 노드가 루트 노드일 경우 DFS 탐색을 수행한다.
	- DFS 탐색에서는 isAdjacent 배열을 기준으로 자식노드를 반복해서 방문한다.
	- 방문하던 노드에서 자식노드가 없을 경우 결과값을 증가 시켜준다.

```java
public class Main {
    // 결과 값 저장
    private int result;
    // 총 노드의 수
    private int nodeCount;
    // 삭제할 노드
    private int removeNode;
    // 인접 노드를 저장하는 이차원 배열
    private boolean[][] isAdjacent;
    // 부모 노드 저장
    private int[] parentArray;

    public Main() {
        this.result = 0;
    }

    public static void main(String[] args) {
        Main main = new Main();
        main.input();
        main.solution();
        main.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.nodeCount = Integer.parseInt(reader.readLine());
            this.isAdjacent = new boolean[this.nodeCount][this.nodeCount];
            this.parentArray = new int[this.nodeCount];
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int num;

            for(int i = 0; i < this.nodeCount; i++) {
                num = Integer.parseInt(token.nextToken());
                // 부모 노드값 저장
                this.parentArray[i] = num;

                // 부모 노드가 있을 경우
                if(num != -1) {
                    // 부모노드에 자식노드가 연결됬음을 저장
                    this.isAdjacent[num][i] = true;
                }
            }

            this.removeNode = Integer.parseInt(reader.readLine());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 삭제할 노드 처리
        for(int i = 0; i < this.nodeCount; i++) {
            // 연결된 정보를 모드 false 로 수정
            this.isAdjacent[i][this.removeNode] = false;
            this.isAdjacent[this.removeNode][i] = false;

            // 루트노드일 경우 현재 문제에서 존재하지 않는 값으로 변경
            this.parentArray[this.removeNode] = -100;
        }

        // 노드의 수만큼 반복
        for(int i = 0; i < this.nodeCount; i++) {
            // 해당 노드가 루트노드이면 dfs 실행
            if(this.parentArray[i] == -1) {
                this.dfs(i);
                break;
            }
        }
    }

    public void dfs(int node) {
        // 리프 노드인지 판별하는 플래그
        boolean isLeaf = true;

        // 총 노드의 수만큼 반복하며
        for(int i = 0; i < this.nodeCount; i++) {
            // 해당 노드에 자식 노드가 있을 경우에
            if(this.isAdjacent[node][i] == true) {
                // 리프 노드가 아님
                isLeaf = false;
                // 연결된 자식노드로 dfs 실행
                dfs(i);
            }
        }

        // 자식노드일 경우 카운트 값 증가
        if(isLeaf) {
            this.result++;
        }
    }
}
```  

---
## Reference
[1068-트리](https://www.acmicpc.net/problem/1068)  
