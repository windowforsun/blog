--- 
layout: single
classes: wide
title: "[풀이] 백준 1149 RGB 거리"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '인접하는 배열에 다른 색으로 칠할 때 최소 비용을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - DP
---  

# 문제
- N 개의 일렬로 나열된 집이 있다.
- N 개의 집에 빨강, 초록, 파랑 중 하나의 색으로 칠하는데 다음과 같은 조건이 있다.
	- i 번째 집과 이웃한 집은 i - 1, i + 1 이다.
	- 이웃한 집끼리는 같은 색을 칠할 수 없다.
	- 각 집마다 3가지 색을 칠할 때 드는 비용이 다르게 주어진다.
- 위의 조건이 있을 때 모든 집을 최소 비용을 칠하는 비용을 구하라.

## 입력
첫째 줄에 집의 수 N이 주어진다. N은 1,000보다 작거나 같다. 둘째 줄부터 N개의 줄에 각 집을 빨강으로 칠할 때, 초록으로 칠할 때, 파랑으로 칠할 때 드는 비용이 주어진다. 비용은 1,000보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 모든 집을 칠할 때 드는 비용의 최솟값을 출력한다.

## 예제 입력

```
3
26 40 83
49 60 57
13 89 99
```  

## 예제 출력

```
96
```  

## 풀이
- 현재 칠해야 하는 집을 i 번째 집이라고 한다.
- 현채 칠해야 하는 색을 j 번째 색이라고 한다.
- i 번째 집에 j 색을 칠하는 최소 비용 = j 번째를 제외한 색중 i - 1 번째 집까지 칠할 때 최소 비용 + i 집의 j 색의 비용이 된다.
- 이웃한 이전의 집에서 필요한 최소 비용을 통해 현재 집의 최소 비용을 도출해 내야하기 때문에 DP 를 사용한다.
- 3 개의 집에 아래와 같은 비용이 있다.

	집 | R | G | B
	---|---|---|---
	0|26|40|83
	1|49|60|57
	2|13|89|99

- 지금 부터 0번째 집부터 차례대로 위에서 설명한 수식을 바탕으로 동작을 수행한다.

1. 집 | R | G | B
   ---|---|---|---
   0|26|40|83
   1|0|0|0
   2|0|0|0
   
   - 0번째 집의 경우 전에 칠한 경우가 없기 때문에 해당 집의 비용을 그대로 넣어준다.
   
1. 집 | R | G | B
   ---|---|---|---
   0|26|40|83
   1|40+49|26+60|26+57
   2|0|0|0
   
   - 1번째 집에서 R, G, B를 0번째 집의 비용에서 칠하려는 색을 제외한 비용 중 최소 비용과 더해 비용을 갱신한다.
   
1. 집 | R | G | B
  ---|---|---|---
  0|26|40|83
  1|89|86|83
  2|83+13|83+89|86+99
  
   - 1번째 집의 최소 비용을 구할 때의 연산을 반복한다.

- 최종적으로 도출된 비용은 아래와 같고, 최종적인 최소 비용은 96이 된다.

	집 | R | G | B
	---|---|---|---
	0|26|40|83
	1|89|86|83
	2|96|172|185
  
```java
public class Main {
    // RGB 의 개수
    private final int RGB_COUNT = 3;
    // 출력 결과 저장
    private int result;
    // 총 집의 수
    private int totalCount;
    // 각 집마다 RGB 비용 저장
    private int[][] rgbCost;

    public Main() {
        this.result = Integer.MAX_VALUE;
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
            this.totalCount = Integer.parseInt(reader.readLine());
            this.rgbCost = new int[totalCount][RGB_COUNT];
            StringTokenizer token;

            for(int i = 0; i < this.totalCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");

                for(int j = 0; j < RGB_COUNT; j++) {
                    this.rgbCost[i][j] = Integer.parseInt(token.nextToken());
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // Memorization 을 사용한 이차원 배열
        int[][] dp = new int[this.totalCount][RGB_COUNT];
        int min, cost1, cost2;

        // 첫 번째 집에 대한 RGB 비용을 넣어준다.
        dp[0][0] = this.rgbCost[0][0];
        dp[0][1] = this.rgbCost[0][1];
        dp[0][2] = this.rgbCost[0][2];

        // 두 번째 집부터 마지막 집까지 반복
        for(int i = 1; i < this.totalCount; i++) {

            // 현재 집에 RGB 색을 하나씩 칠하는 반복문
            for(int j = 0; j < RGB_COUNT; j++) {
                // 지금까지 칠한(i - 1 번까지) 비용 중 j 번째 색을 제외한 2가지 색에 대한 비용
                cost1 = dp[i - 1][(j + 1) % RGB_COUNT];
                cost2 = dp[i - 1][(j + 2) % RGB_COUNT];

                // 2가지 색의 비용중 적은 비용을 선택
                if(cost1 > cost2) {
                    min = cost2;
                } else {
                    min = cost1;
                }

                // 현재 색을 칠하는 i 번째의 집의 j 번째 색의 비용과 i - 1 까지 칠한 색중 적은 비용을 더해 준다.
                dp[i][j] = this.rgbCost[i][j] + min;
            }
        }

        // 가장 적은 비용 선택
        for(int i = 0; i < RGB_COUNT; i++) {
            if(this.result > dp[this.totalCount - 1][i]) {
                this.result = dp[this.totalCount - 1][i];
            }
        }
    }
}
```  

---
## Reference
[1149-RGB 거리](https://www.acmicpc.net/problem/1149)  
