--- 
layout: single
classes: wide
title: "[풀이] 백준 1194 달이 차오른다, 가자."
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '미로를 탐색하여 출구로 나갈 수 있는 최소 이동 횟수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - BFS
---  

# 문제
- 직사각형의 미로를 탈출하려 하는데 미로에는 아래와 같은 특징이 있다.
	1. 빈곳(.) : 언제나 이동 가능하다.
	1. 벽(#) : 절대 이동 할 수 없다.
	1. 열쇠(a~f) : 언제나 이동 할 수 있다. 이곳에 방문하면 열쇠를 획득한다.
	1. 문(A~F) : 대응하는 열쇠가 있어야 이동 가능하다.
	1. 출발지(0) : 빈곳이고, 출발 지점이다.
	1. 출구(1) : 미로를 탈출 할 수 있는 출구이다.
- 이동 할 수 있는 방향은 상, 하, 좌, 우이고, 한번에 한칸 씩만 이동 가능하다.
- 위의 미로에서 조건을 만족시키는 미로를 탈출하는 최소 이동 횟수를 구하라.

## 입력
첫째 줄에 미로의 세로 크기 N과 가로 크기 M이 주어진다. (N,M <= 50) 둘째 줄부터 N개의 줄에 미로의 모양이 주어진다. 같은 타입의 열쇠가 여러 개 있을 수 있고, 문도 마찬가지이다. 그리고, 영식이가 열쇠를 숨겨놓는 다면 문에 대응하는 열쇠가 없을 수도 있다. 0은 한 개, 1은 적어도 한 개 있다. 그리고, 열쇠는 여러 번 사용할 수 있다.

## 출력
첫째 줄에 민식이가 미로를 탈출하는데 드는 이동 횟수의 최솟값을 출력한다. 만약 민식이가 미로를 탈출 할 수 없으면, -1을 출력한다.

## 예제 입력

```
1 7
f0.F..1
```  

## 예제 출력

```
7
```  

## 풀이
- 문제 해결을 위해 BFS 탐색을 사용하는데 아래와 같은 특징이 있다.
	- BFS 탐색 시 while 문 첫 부분에서 queue 에 들어가 있는 위치의 개수(queue.size()) 만큼 for 문을 돌린다.
	- 위 for 문에서는 현재 queue 들어가 있는 위치에서 이동 가능한 위치들을 다시 queue 에 넣는다.
	- for 문이 끝나면 이동 횟수를 증가 시키고, 다시 queue 에 들어가 있는 위치의 개수 만큼 for 문을 돌린다.
	- 즉 queue 에 들어가 있는 위치가 현재 방문 하는 위치가 되고, queue.size() 만큼 반복문에서 현재 방문 하는 위치에서 방문가능한 위치(다음에 방문할)를 넣으면서 최소 이동 횟수인 result 값을 증가 시켜주는 방식이다.
- 5 개의(a~f) 키가 있고 문의 경우 대응하는 키가 있어야 방문이 가능하다. 이는 BFS 에서 방문을 할때 아래와 같은 경우의 수가 있을 수 있다.
	- 아무런 키가 없을 경우에는 A 문을 방문하지 못한다.
	- a 키를 얻은 후에는 같은 위치에 있는 A 문에 방문할 수 있다.
- 위의 경우를 생각해 보면 BFS 에서 방문한 위치를 단순하게 이차원 배열로 표현 할 경우 키의 상황에 따른 정답을 구할 수 없다.
	- 5개의(a~f) 키에서 아무런 키도 없는 경우를 더하면 총 6가지의 경우가 된다.
	- 이 6가지의 경우에서는 각 키마다 획득한 경우와 획득하지 않는 경우가 있다.
	- 이를 비트마스크를 사용해서 총 2^6 으로 64 개의 경우의 수를 관리해야 한다.왜 64 개의 경우의 수가 필요한지는 아래의 표에서 확인 가능하다.
	
	얻은 키|숫자|Binary|
	---|---|:---|
	none|0|0|
	a|1|1|
	b|2|10|
	a,b|3|11|
	c|4|100|
	a,b,c|7|111
	d|8|1000|
	a,b,c,d|15|1111
	e|16|10000|
	a,b,c,d,e|31|11111
	f|32|100000|
	a,b,c,d,e,f|63|111111
	
	- 비트마스크를 사용할 때 a~f 까지의 모든 키를 획득하면 64가 되므로 64 개의 경우의 수로 키 상황을 관리하게 된다.
- BFS 에서 사용하는 방문 배열은 보통 2차원 배열로 사용한다. 하지만 키 획득에 따른 방문의 경우의 수가 달라지기 때문에 3차원 배열을 사용한다.
	- 방문 배열을 isVisited 라고 하고, 미로의 행수를 row, 열 수를 col, 키 상황에 대한 경우의 수를 key 로하면 방문 배열 `isVisited = new boolean[row][col][key]` 가된다.
- 키 획득에 대한 연산은 `mazeKey | 1 << (keyCh - 'a')` 가 된다.
- 키 확인에 대한 연산은 `mzaeKey & 1 << (keyCh = 'A')` 가 된다.
	
	
```java
public class Main {
    // 키의 개수
    private final int KEY_MAX = 'f' - 'a';
    // 출력 결과 저장
    private int result;
    // 미로의 row 수
    private int row;
    // 미로의 col 수
    private int col;
    // 미로의 시작 인덱스
    private int[] start;
    // 전체 미로 이차원 배열
    private char[][] mazeArray;
    // 키 획득 상황에 따른 방문 삼차원 배열
    private boolean[][][] isVisited;
    // 이동할 수 있는 방향
    private int[][] adjIndex = new int[][] {
            {1, 0}, {0, 1}, {-1, 0}, {0, -1}
    };

    public Main() {
        this.result = 0;
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
            String tmp;
            this.row = Integer.parseInt(token.nextToken());
            this.col = Integer.parseInt(token.nextToken());

            this.mazeArray = new char[this.row][this.col];
            this.isVisited = new boolean[this.row][this.col][(1 << KEY_MAX + 1)];

            for(int i = 0; i < this.row; i++) {
                tmp = reader.readLine();

                for(int j = 0; j < this.col; j++) {
                    this.mazeArray[i][j] = tmp.charAt(j);
                    // 미로 시작 위치 인덱스 설정
                    if(this.mazeArray[i][j] - '0' == 0) {
                        this.start = new int[]{i, j};
                    }
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
        this.bfs(this.start[0], this.start[1]);
    }

    // 너비 우선 탐색으로 미로 탐색
    public void bfs(int r, int c) {
        // 현재 미로 인덱스 및 키 상황 (현재 R, 현재 C, 키상황을 binary 로 나타낸 수)
        int[] current;
        int nextR, nextC, nextMazeKey, currentR, currentC, currentMazeKey, queueSize;
        char nextCh, currentCh;
        // 갈 수 있는 곳인지 아닌지
        boolean isGo;

        // 미로 시작 인덱스 및 키상황 방문 설정
        this.isVisited[r][c][0] = true;

        // BFS 에서 사용할 방문 queue
        LinkedList<int[]> queue = new LinkedList<>();
        // 시작 인덱스 및 키 상황 큐에 넣어주기
        queue.addLast(new int[]{r, c, 0});

        // 큐가 비어 있을 때 까지 반복
        while(!queue.isEmpty()) {
            // DFS 탐색을 수행할 때 현재 큐에 있는 크기 만큼 반복함
            // 다시 아래의 for 문에서 queue 넣어진 크기 만큼 반복하고 result 값 증가시켜줌
            // 즉 result 의 카운트 값은 아래 for 문이 몇번 호출 되었는지에 의존됨
            queueSize = queue.size();

            for(int k = 0; k < queueSize; k++) {
                // 현재 방문한 위치에 대한 정보
                current = queue.removeFirst();
                currentR = current[0];
                currentC = current[1];
                currentCh = this.mazeArray[currentR][currentC];
                currentMazeKey = current[2];

                // 출구일 경우 탐색 종료
                if(currentCh == '1') {
                    return;
                }

                // 방문 가능한 4곳 모두 방문
                for(int i = 0; i < 4; i++) {
                    isGo = true;
                    nextR = currentR + this.adjIndex[i][0];
                    nextC = currentC + this.adjIndex[i][1];
                    nextMazeKey = currentMazeKey;

                    // 방분 가능한 인덱스가 유효하면서, # 벽이아니고, 키 상황에도 방문한 적이 없는 경우
                    if(nextR >= 0 && nextR < this.row && nextC >= 0 && nextC < this.col && this.mazeArray[nextR][nextC] != '#' && !this.isVisited[nextR][nextC][currentMazeKey]) {
                        nextCh = this.mazeArray[nextR][nextC];

                        if(nextCh >= 'a' && nextCh <= 'z') {
                            // 키륵 획득 하기 위해 현재 키상황을 변경
                            nextMazeKey = currentMazeKey | 1 << (nextCh - 'a');
                        } else if(nextCh >= 'A' && nextCh <= 'Z') {
                            // 현재 해당하는 키가 있는지 검사
                            int check = nextMazeKey & 1 << (nextCh - 'A');
                            // 키가 없을 경우 isGo 값 갱신
                            if(check <= 0) {
                                isGo = false;
                            }
                        }

                        // 인접한 위치에 방문이 가능하면 방문으로 설정하고 queue 에 넣어줌
                        if(isGo) {
                            isVisited[nextR][nextC][nextMazeKey] = true;
                            queue.addLast(new int[]{nextR, nextC, nextMazeKey});
                        }
                    }
                }
            }

            // 한 뎁스에 대한 방문이 모두 끝났으면 result 값 증가
            this.result++;
        }

        // 모든 미로를 탐색하였지만 출구를 만나지 못한 경우이므로 -1 값으로 설정
        this.result = -1;
    }
}
```  

---
## Reference
[1194-달이 차오른다, 가자.](https://www.acmicpc.net/problem/1194)  
