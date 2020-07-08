--- 
layout: single
classes: wide
title: "[풀이] 백준 1100 햐얀 칸"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '체스 판에서 하얀 칸 위에 있는 말의 개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - String
  - Searching
---  

# 문제
- 8*8 체스판이 있다.
- 검정과 하얀 칸이 번걸아 가면서 칠해져 있다.
- 가장 왼쪽 위칸(0,0)은 햐얀색이다.
- 체스판의 상태가 주어졌을 때 햐얀색 위에 있는 말의 개수를 구하라.

## 입력
첫째 줄부터 8개의 줄에 체스판의 상태가 주어진다. ‘.’은 빈 칸이고, ‘F’는 위에 말이 있는 칸이다.

## 출력
첫째 줄에 문제의 정답을 출력한다.

## 예제 입력

```
.F.F...F
F...F.F.
...F.F.F
F.F...F.
.F...F..
F...F.F.
.F.F.F.F
..FF..F.
```  

## 예제 출력

```
1
```  

# 풀이
- 체스판의 행에서 흰색이 먼저 시작하는 인덱스는 모두 짝수 이다.
- 흰색이 먼저 시작하는 행의 인덱스가 짝수일때 열의 인덱스 또한 짝수이다.
- 다시 표현하면, (i, j) 의 인덱스가 체스판에서 흰색인지 알아내는 수식은 다음과 같다.
	- ((i % 2) + j % 2) == 0 이면 흰색

```java
public class Main {
    final int MAX = 8;
    // 출력 결과
    private int result;
    // 체스 판 문자열
    private String[] chessStrArray;

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
        this.chessStrArray = new String[MAX];

        try {
            for (int i = 0; i < MAX; i++) {
                this.chessStrArray[i] = reader.readLine();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int x;
        char ch;

        for (int i = 0; i < MAX; i++) {
            // x가 0인 행에서는 햐얀색이 첫 칸 부터 시작
            x = i % 2;

            for (int j = 0; j < MAX; j++) {
                ch = this.chessStrArray[i].charAt(j);

                // 열 인덱스에 x 값을 더했을 때 짝수 이면 하얀칸
                if ((j + x) % 2 == 0
                        && ch == 'F') {
                    // 하얀칸에 말이 있을 경우 결과값 저장
                    this.result++;
                }
            }
        }
    }
}
```  

---
## Reference
[1100-햐얀 칸](https://www.acmicpc.net/problem/1100)  
