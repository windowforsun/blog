--- 
layout: single
classes: wide
title: "[풀이] 백준 1009 분산처리"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '10개의 컴퓨터로 N 개의 Task 를 처리할 때 마지막 Task 를 처리하는 컴퓨터를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- 10 개의 컴퓨터로 많은 데이터를 분산 처리한다.
	- 1번 컴퓨터는 1번 데이터
	- 2번 컴퓨터는 2번 데이터
	- ...
	- 10번 컴퓨터는 10번 데이터
- a^b 개의 데이터가 주어질 때 마지막 데이터를 몇번 컴퓨터가 처리하는지 구하라.

## 입력
입력의 첫 줄에는 테스트 케이스의 개수 T가 주어진다. 그 다음 줄부터 각각의 테스트 케이스에 대해 정수 a와 b가 주어진다. (1 ≤ a < 100, 1 ≤ b < 1,000,000)

## 출력
각 테스트 케이스에 대해 마지막 데이터가 처리되는 컴퓨터의 번호를 출력한다.

## 예제 입력

```
5
1 6
3 7
6 2
7 100
9 635
```  

## 예제 출력

```
1
7
6
1
9
```  

## 풀이
- N 개의 데이터일 때 마지막 처리하는 컴퓨터 번호는 ((N - 1) % 10) + 1 번 컴퓨터가 된다.
- 문제를 해결하기 위해 필요한 값은 N 번의 수에서 마지막 자리이다.
- a^(1~4) 까지 했을 때의 마지막 자리수는 아래와 같다.

	a^b|N|마지막 자리|
	---|---|---|
	2^1|2|2|
	2^2|4|4|
	2^3|8|8|
	2^4|16|6|
	3^1|3|3|
	3^2|9|9|
	3^3|27|7|
	3^4|81|1|
	4^1|4|4|
	4^2|16|6|
	4^3|64|4|
	4^4|256|5

- a^b 에서 b=1~4 일때의 마지막 자리수가 4 개씩 반복되는 것을 확인 할 수 있다.
	
```java
public class Main {
    // 출력 결과
    private StringBuilder result;
    // 테스트 케이스 카운트
    private int caseCount;
    // 입력 받은 a, b 값
    private int[][] array;
    // a 값을 이미 처리한 적이 있는 지에 대한 여부
    private int[] check;
    // a 값을 지수승 했을 때의 마지막 자리 캐싱, 4개까지
    private int[][] cache;

    public Main() {
        this.result = new StringBuilder();
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
            this.caseCount = Integer.parseInt(reader.readLine());
            this.array = new int[this.caseCount][2];
            this.check = new int[101];
            this.cache = new int[101][4];
            StringTokenizer token;

            for(int i = 0; i < this.caseCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");

                this.array[i][0] = Integer.parseInt(token.nextToken());
                this.array[i][1] = Integer.parseInt(token.nextToken());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int a, b, num;

        for(int i = 0; i < this.caseCount; i++) {
            a = this.array[i][0];
            b = this.array[i][1];

            // a 값을 아직 처리 한 적이 없다면
            if(this.check[a] == 0) {
                num = 1;
                
                // a^(1~4) 까지 구하며 마지막 자리를 구해 cache 넣는다.
                for(int j = 0; j < 4; j++) {
                    num *= a;

                    this.cache[a][j] = num % 10;

                    // 0일 경우에는 10번째 컴퓨터
                    if(this.cache[a][j] == 0) {
                        this.cache[a][j] = 10;
                    }
                }

                // a 값 체크 갱신
                this.check[a] = a;
            }

            // a^b 승일 때 마지막 자리는 최대 4개 까지 반복된다. 
            // 10 개의 컴퓨터 이므로 반복되는 마지막 자리의 순서에 해당하는 cache 값을 결과값으로 설정한다.
            this.result.append(this.cache[a][(b - 1) % 4]).append("\n");
        }
    }
}
```  

---
## Reference
[1009-분산처리](https://www.acmicpc.net/problem/1009)  
