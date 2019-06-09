--- 
layout: single
classes: wide
title: "[풀이] 백준 1652 누울 자리를 찾아라"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 이차원 배열에서 누울 수 있는 자리를 찾아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- 누울 수 있는 자리는 2차원 배열에서 최소 2개의 자리가 연속되어서 비여있는 경우이다.
	- 가로, 세로 모두 가능
- 2차원 배열의 정보가 주어질 때 위 조건에 맞는 누울 수 있는 자리의 개수를 구하라.

## 입력
첫째 줄에 방의 크기 N이 주어진다. N은 1이상 100이하의 정수이다. 그 다음 N줄에 걸쳐 N개의 문자가 들어오는데 '.'은 아무것도 없는 곳을 의미하고, 'X'는 짐이 있는 곳을 의미한다.

## 출력
첫째 줄에 가로로 누울 수 있는 자리와 세로로 누울 수 있는 자리의 개수를 출력한다.

## 예제 입력

```
5
....X
..XX.
.....
.XX..
X....
```  

## 예제 출력

```
5 4
```  

## 풀이
- 주어지는 입력의 데이터를 아래와 같이 가로, 세로로 분리해서 입력을 받는다.
	- 가로
	
		```
		12340
		12001
		12345
		10012
		01234
		``` 
	
	- 세로
	
		```
		11110
		22001
		33112
		40023
		01134
		``` 

- 분리 해서 입력 받은 2개의 2차원배열을 가로, 세로에 맞게 탐색하며 0값 일때 이전의 값이 2보다 클경우 각 카운트 결과값을 증가 시켜주면 된다.

```java
public class Main {
    private StringBuilder result;
    private int size;
    private int[][][] matrix;
    private final int VER = 0;
    private final int HOR = 1;

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
            this.size = Integer.parseInt(reader.readLine());
            this.matrix = new int[2][this.size][this.size];
            this.result = new StringBuilder();

            String str;
            int countVer, countHor;
            char ch;

            for(int i = 0; i < this.size; i++) {
                str = reader.readLine();

                for(int j = 0; j < this.size; j++) {
                    ch = str.charAt(j);

                    if(j == 0) {
                        countVer = 1;
                    } else {
                        countVer = this.matrix[VER][i][j - 1] + 1;
                    }

                    if(i == 0) {
                        countHor = 1;
                    } else {
                        countHor = this.matrix[HOR][i - 1][j] + 1;
                    }

                    if(ch == 'X') {
                        countVer = countHor = 0;
                    }

                    this.matrix[VER][i][j] = countVer;
                    this.matrix[HOR][i][j] = countHor;
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
        int horCount = 0, verCount = 0, ver, hor;

        for(int i = 0; i < this.size; i++) {
            for(int j = 1; j < this.size; j++) {
                ver = this.matrix[VER][i][j - 1];
                hor = this.matrix[HOR][j - 1][i];

                if(this.matrix[VER][i][j] == 0 && ver >= 2) {
                    verCount++;
                }

                if(this.matrix[HOR][j][i] == 0 && hor >= 2) {
                    horCount++;
                }
            }

            if(this.matrix[VER][i][this.size - 1] >= 2) {
                verCount++;
            }

            if(this.matrix[HOR][this.size - 1][i] >= 2) {
                horCount++;
            }
        }

        this.result.append(verCount)
                .append(" ")
                .append(horCount);
    }
}
```  

---
## Reference
[1652-누울 자리를 찾아라](https://www.acmicpc.net/problem/1652)  
