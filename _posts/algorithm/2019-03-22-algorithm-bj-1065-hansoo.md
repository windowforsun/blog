--- 
layout: single
classes: wide
title: "[풀이] 백준 1065 한수"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '한 수의 개수를 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Brute Force
  - Search
---  

# 문제
- X 자리의수가 등차수열을 이룬다면, 해당 수를 한수라고 한다.
- 등차수열은 연속된 두 개의 수의 차가 일정한 수열을 말한다.
- N이 주어졌을 때, 1보다 크거나 N 보다 같거나 작은 한수의 개수를 구하라

## 입력
첫째 줄에 1,000보다 작거나 같은 자연수 N이 주어진다.

## 출력
첫째 줄에 1보다 크거나 같고, N보다 작거나 같은 한수의 개수를 출력한다.

## 예제 입력

```
110
```  

## 예제 출력

```
99
```  

# 풀이
- 1 ~ N 까지의 숫자 중 한수의 개수를 구하면 된다.
- 한수는 X 자리를 가진 정수의 자리수들이 등차수열을 이룰 때 한수라고 할 수 있다.
- 문제에는 명시되어 있지 않지만, 예제 입력과 예제 출력으로 유추할 수 있는 것은 한 자리수도 한수에 해당된다.
	- 1 ~ 99 까지는 모든 수가 한수에 해당된다
- 100 미만 까지는 입력 받은 수를 그대로 출력 결과값으로 사용하고, 같거나 큰 값들에서 부터 반복문을 통해 한수를 판별하면 된다.

```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 한수를 구하는 최대값
    private int count;

    public Main() {
        this.result = new StringBuilder();
    }

    public static void main(String[] args) {
        Main main = new Main();
        main.input();
        main.solution();
        main.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            this.count = Integer.parseInt(reader.readLine());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int resultCount = 0, len, gap;
        boolean is;
        String tmp;

        // 100보다 작을 경우 한수는 최대값과 같음
        if (this.count < 100) {
            this.result.append(this.count);
        } else {
            // 100 보다 같거나 클 경우 한수의 개수는 99개
            resultCount = 99;

            // 100 부터 카운트 값까지 Loop
            for (int i = 100; i <= this.count; i++) {
                // 현재 숫자를 문자열로 변환
                tmp = i + "";
                // 현재 숫자가 한수인지 아닌지 플래그
                is = true;
                // 숫자의 문자 개수
                len = tmp.length();

                // 숫자 문자열에서 0번째와 1번째의 숫자 차이를 구함
                gap = this.chToInt(tmp.charAt(0)) - this.chToInt(tmp.charAt(1));

                // 1번째 부터 len - 1 까지 반복하며 다음 j번째 숫자가 gap 의 차이를 갖는지 판별
                for (int j = 1; j < len - 1; j++) {
                    // 한수가 아니라면 플래그를 갱신하고 break
                    if (this.chToInt(tmp.charAt(j)) - gap != this.chToInt(tmp.charAt(j + 1))) {
                        is = false;
                        break;
                    }
                }
                
                // 한수일 경우에만 결과 카운트 값 증가
                if (is) {
                    resultCount++;
                }
            }

            this.result.append(resultCount);
        }
    }

    public int chToInt(char c) {
        return c - '0';
    }
}
```  

---
## Reference
[1065-한수](https://www.acmicpc.net/problem/1065)  
