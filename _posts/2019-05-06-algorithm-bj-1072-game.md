--- 
layout: single
classes: wide
title: "[풀이] 백준 1072 게임"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '게임 승률이 변할때를 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- 승패가 있는 게임을 할때 매번 승리한다고 가정한다.
- 게임 횟수(X), 승리한 게임(Y) 이 주어질 때 확률 Z% 를 구할 수 있다.
- 게임을 몇판을 더해야 확률 Z% 가 변하는지 구하라.
	- 확률 Z에서 소수점 자리는 버린다.

## 입력
각 줄에 X와 Y가 주어진다. X는 1,000,000,000보다 작거나 같은 자연수이고, Y는 0보다 크거나 같고, X보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 형택이가 게임을 몇 판 해야하는지 출력한다. 만약 Z가 절대 변하지 않는다면 -1을 출력한다.

## 예제 입력

```
10 8
```  

## 예제 출력

```
1
```  

## 예제 입력

```
100 80
```  

## 예제 출력

```
6
```  
## 풀이
- X = 10, Y = 8 일 때 8/10 * 100 = Z = 80 이 된다.
- 변하는 최소 승률은 81 이므로 Z 가 81이 되기 위해서 몇판의(a) 게임을 더해야 하는지 구하면 된다.
- 이를 수식으로 세워보면 아래와 같다.
	1. (8 + a)/(10 + a) * 100 = 81
	1. 800 + 100a = 8100 + 81a
	1. a = 100/19 = 6

```java
public class Main {
    // 출력 결과
    private StringBuilder result;
    // 플레이한 게임 수, 승리한 게임 수
    private long gameCount, winCount;

    public Main() {
        this.input();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            String str;
            StringTokenizer token;

            this.result = new StringBuilder();

            while((str = reader.readLine()) != null) {
                token = new StringTokenizer(str, " ");
                this.gameCount = Integer.parseInt(token.nextToken());
                this.winCount = Integer.parseInt(token.nextToken());
                this.solution();
            }


        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 승률이 변하기 위해서 해야하는 게임 수
        long count = -1;
        // 현재 게임 승률
        long percent = (this.winCount * 100) / this.gameCount;

        // 99퍼 이상이면 확률은 변하지 않음
        if(percent < 99) {
            // 변할 수 있는 최소 승률
            percent += 1;
            
            // count = ((총 게임 수 * 변할 최소 확률) - (100 * 승리 수)) / (100 - 변할 최소 확률)
            long remainPercent = 100 - percent;
            long remainCount = ((this.gameCount * percent) - (100 * this.winCount));

            count = remainCount / remainPercent;
            
            // 소수점 자리가 있을 경우 올림처리
            if(remainCount % remainPercent != 0) {
                count += 1;
            }
        }
        this.result.append(count).append("\n");
    }
}
```  

---
## Reference
[1072-게임](https://www.acmicpc.net/problem/1072)  
