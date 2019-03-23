--- 
layout: single
classes: wide
title: "[풀이] 백준 1012 유기농 배추"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: 'BFS, DFS 를 사용해서 분리된 공간의 수를 알아내 보자'
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
- 2차원 배열에 배추가 심어져 있다.
- 지렁이는 배추를 옮겨다니며 해충을 먹는다.
- 지렁이는 인접(상하좌우)한 곳으로 옮겨 다닐 수 있다.
- 이때 최소 몇마리의 지렁이가 전체 배추의 해충을 먹기위해 필요한지 알아내자

## 입력
입력의 첫 줄에는 테스트 케이스의 개수 T가 주어진다. 
그 다음 줄부터 각각의 테스트 케이스에 대해 첫째 줄에는 배추를 심은 배추밭의 가로길이 M(1 ≤ M ≤ 50)과 세로길이 N(1 ≤ N ≤ 50), 
그리고 배추가 심어져 있는 위치의 개수 K(1 ≤ K ≤ 2500)이 주어진다. 
그 다음 K줄에는 배추의 위치 X(0 ≤ X ≤ M-1), Y(0 ≤ Y ≤ N-1)가 주어진다.

## 출력
각 테스트 케이스에 대해 필요한 최소의 배추흰지렁이 마리 수를 출력한다.

## 예제 입력

```
2
10 8 17
0 0
1 0
1 1
4 2
4 3
4 5
2 4
3 4
7 4
8 4
9 4
7 5
8 5
9 5
7 6
8 6
9 6
10 10 1
5 5
```  

## 예제 출력

```
5
1
```  

# 풀이
- 주어진 2차원 배열의 전체탐색을 한다.
- 탐색 중 배추있는 위치일 경우 DFS, BFS 탐색을 수행하고, 분리된 땅을 카운트하는 변수의 값을 증가시킨다.
- DFS, BFS 에서는 상하좌우에 배추가 있는지 검사하고 있다면 해당 위치로 가서 다시 탐색을 수행한다.
- 탐색을 한 부분에 대해서는 배추가 없는 것으로 2차원 배열을 수정해준다.
- 2차원 배열의 전체탐색이 끝나면 분리된 땅의 카운트 값을 출력해 준다.

```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 테스트 케이스 수
    private int testCaseCount;
    // 2차원 배열에서 컬럼의 수
    private int c;
    // 2차원 배열에서 로우의 수
    private int r;
    // 배추의 수
    private int cabbageCount;
    // 배추의 위치가 저장되는 매트릭스
    private int[][] matrix;
    // 지렁이가 다닐수 있는 인접 배열
    private final int[][] adjacent = new int[][]{
            {1, 0}, {0, 1}, {-1, 0}, {0, -1}
    };

    public Main() {
        this.result = new StringBuilder();
    }

    public static void main(String[] args) {
        Main main = new Main();
        main.input();
        main.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            this.testCaseCount = Integer.parseInt(reader.readLine());

            for (int i = 0; i < this.testCaseCount; i++) {
                StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
                this.c = Integer.parseInt(token.nextToken());
                this.r = Integer.parseInt(token.nextToken());
                this.matrix = new int[this.r][this.c];
                this.cabbageCount = Integer.parseInt(token.nextToken());

                int tmpC, tmpR;
                for (int j = 0; j < this.cabbageCount; j++) {
                    token = new StringTokenizer(reader.readLine(), " ");
                    tmpC = Integer.parseInt(token.nextToken());
                    tmpR = Integer.parseInt(token.nextToken());

                    this.matrix[tmpR][tmpC] = 1;
                }

                this.solution();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.print(this.result);
    }

    public void solution() {
        // 분리된 땅 카운트 변수
        int landCount = 0;

        // 탐색을 위해 matrix 전체 탐색
        for(int i = 0; i < this.r; i++){
            for(int j = 0; j < this.c; j++) {

                // matrix 에 배추가 있을 경우 땅 카운트 증가시켜주고, DFS 혹은 BFS 호출
                if(this.matrix[i][j] == 1) {
                    landCount++;
//                    this.bfs(i, j);
                    this.dfs(i,j);
                }
            }
        }

        this.result.append(landCount).append("\n");
    }

    // BFS 탐색 메서드
    public void bfs(int i, int j) {
        LinkedList<int[]> queue = new LinkedList<>();
        int[] current;
        queue.addLast(new int[]{i, j});
        int x, y, newX, newY;

        while(!queue.isEmpty()) {
            current = queue.removeFirst();
            x = current[0];
            y = current[1];

            // 탐색한 위치는 배추가 없는 것으로 설정해준다.
            this.matrix[x][y] = 0;

            // adjacent 인접 배열로 지렁이가 갈수 있는 모든 방향에 배추가 있는지 검사한다.
            for(int k = 0; k < 4; k++) {
                newX = x + this.adjacent[k][0];
                newY = y + this.adjacent[k][1];

                // 위치 유효성 검사와 배추가 있는 곳인지 검사
                if(newX >= 0 && newX < this.r && newY >= 0 && newY < this.c && this.matrix[newX][newY] == 1) {
                    queue.addLast(new int[]{newX, newY});
                }
            }
        }
    }

    // DFS 탐색 메서드
    public void dfs(int i, int j) {
        int newX, newY;

        // 탐색한 위치는 배추가 없는 것으로 설정해준다.
        this.matrix[i][j] = 0;

        // adjacent 인접 배열로 지렁이가 갈수 있는 모든 방향에 배추가 있는지 검사한다.
        for(int k = 0; k < 4; k++) {
            newX = i + this.adjacent[k][0];
            newY = j + this.adjacent[k][1];

            // 위치 유효성 검사와 배추가 있는 곳인지 검사
            if(newX >= 0 && newX < this.r && newY >= 0 && newY < this.c && this.matrix[newX][newY] == 1) {
                dfs(newX, newY);
            }
        }
    }
}
```  


---
## Reference
[1012-유기농 배추](https://www.acmicpc.net/problem/1012)  
