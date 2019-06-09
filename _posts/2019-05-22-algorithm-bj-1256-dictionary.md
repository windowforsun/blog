--- 
layout: single
classes: wide
title: "[풀이] 백준 1256 사전"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '사전에서 순서에 해당되는 문자열을 찾아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - DP
  - Combination
  - Binomial Coefficient
---  

# 문제
- a와 z로만 이루어진 사전이 있다.
- a의 개수, z의 개수가 주어질 때 사전순으로 k 번째의 해당하는 문자열을 구하라.

## 입력
첫째 줄에 N, M, K가 순서대로 주어진다. N과 M은 100보다 작거나 같은 자연수이고, K는 1,000,000,000보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 규완이의 사전에서 K번째 문자열을 출력한다. 만약 규완이의 사전에 수록되어 있는 문자열의 개수가 K보다 작으면 -1을 출력한다.

## 예제 입력

```
2 2 2
```  

## 예제 출력

```
azaz
```  

## 풀이
- 조합 알고리즘을 이용해서 a의 개수 n과 z의 개수 m에 따른 문자열의 총 개수는 아래와 같다.
	- n+mCn = n+mCm
	- n+mCn = n-1+mCm + n+m-1Cn

	n\m|1|2|3|4|5|6
	---|---|---|---|---|---|---
	1|2|3|4|5|6|7
	2|3|3|10|15|21|28 
	3|4|10|20|35|56|84
	4|5|15|35|70|126|210 
	5|6|21|56|126|252|462
	6|7|28|84|210|462|904

- n+mCn = n-1+mCm + n+m-1Cn 식을 다른 표현으로 하면 n+mCn = (a로 시작하는 개수) + (z로 시작하는 개수)
- 위 식을 통해 n+mCn 에서 a로 시작하는 개수는 n-1+mCm 임을 알 수 있다.

```java
public class Main {
    public TestHelper testHelper = new TestHelper();

    class TestHelper {
        private ByteArrayOutputStream out;

        public TestHelper() {
            this.out = new ByteArrayOutputStream();

            System.setOut(new PrintStream(this.out));
        }

        public String getOutput() {
            return this.out.toString().trim();
        }
    }

    private final int MAX = 1000000000;
    private StringBuilder result;
    private int aCount;
    private int zCount;
    private int count;
    private int[][] totalCountMatrix;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            this.aCount = Integer.parseInt(token.nextToken());
            this.zCount = Integer.parseInt(token.nextToken());
            this.count = Integer.parseInt(token.nextToken());
            this.totalCountMatrix = new int[this.aCount + 1][this.zCount + 1];
            this.result = new StringBuilder();
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        this.totalCountMatrix[0][0] = 0;
        for(int i = 1; i <= this.aCount; i++) {
            this.totalCountMatrix[i][0] = 1;
        }

        for(int i = 1; i <= this.zCount; i++) {
            this.totalCountMatrix[0][i] = 1;
        }

        // DP 를 통해 문자열의 총 개수를 구함
        for(int i = 1; i <= this.aCount; i++) {
            for(int j = 1; j <= this.zCount; j++) {
            	// n+mCn = n-1+mCm + n+m-1Cn
                this.totalCountMatrix[i][j] = Integer.min(this.totalCountMatrix[i - 1][j] + this.totalCountMatrix[i][j - 1], MAX);
            }
        }

        if(this.count > this.totalCountMatrix[this.aCount][this.zCount]) {
            this.result.append("-1");
        } else {
            int n = this.aCount;
            int m = this.zCount;
            int k = this.count;
            long halfCount;
            char ch;

            while(n > 0 && m > 0) {
                // n,m 에서 a로 시작하는 문자열의 개수
                halfCount = this.totalCountMatrix[n - 1][m];

                // a로 시작하는 개수보다 작거나 같으면 a, 크면 z로 시작
                if(k > halfCount) {
                    // a로 시작하는 개수만큼 감소시켜준다.
                    k -= halfCount;
                    // z 개수를 감소
                    m -= 1;
                    ch = 'z';
                } else {
                    // a 개수를 감소
                    n -=1;
                    ch = 'a';
                }

                this.result.append(ch);
            }

            // a, z 중 남은 문자열을 이어서 추가
            int insertCount;
            if(n > 0) {
                insertCount = n;
                ch = 'a';
            } else {
                insertCount = m;
                ch = 'z';
            }

            for(int i = 0; i < insertCount; i++) {
                this.result.append(ch);
            }
        }
    }
}
```  

---
## Reference
[1256-사전](https://www.acmicpc.net/problem/1256)  
