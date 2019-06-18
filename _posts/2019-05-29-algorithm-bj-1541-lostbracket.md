--- 
layout: single
classes: wide
title: "[풀이] 백준 1541 잃어버린 괄호"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 수식에서 괄호를 쳐서 최소값을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Greedy
---  

# 문제
- 주어지는 수식에는 괄호가 모두 지워진 상태의 수식이다.
- +, - 그리고 숫자(0~9)로 구성되어 있고, 50자로 구성되어 있다.
- 이러한 수식에 괄호를 쳐서 수식의 결과값이 최소값이 나오도록 구하라.

## 입력
첫째 줄에 식이 주어진다. 식은 ‘0’~‘9’, ‘+’, 그리고 ‘-’만으로 이루어져 있고, 가장 처음과 마지막 문자는 숫자이다. 그리고 연속해서 두 개 이상의 연산자가 나타나지 않고, 5자리보다 많이 연속되는 숫자는 없다. 수는 0으로 시작할 수 있다.

## 출력
첫째 줄에 정답을 출력한다.

## 예제 입력

```
55-50+40
```  

## 예제 출력

```
-35
```  

## 풀이
- 수식이 +, - 밖에 없는 상태에서 최소값을 구하는 방법은 - 부근에 있는 + 를 모아(괄호를 쳐) 연산해주는 것이다.
	- 55-50+40-10+100+20+10-10+10 = 55-(50+40)-(10+100+20+10)-(10+10)

```java
public class Main {
    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    private int result;
    private String input;

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.input = reader.readLine();
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        StringTokenizer minusToken = new StringTokenizer(this.input, "-");
        StringTokenizer plusToken;
        String minusTokenStr, plusTokenStr;
        int flag = 1, plusTokenSum;

        // - 를 기준으로 입력 받은 문자열을 나눈다.
        while(minusToken.hasMoreTokens()) {
            minusTokenStr = minusToken.nextToken();
            plusToken = new StringTokenizer(minusTokenStr, "+");

            plusTokenSum = 0;
            // + 를 기준으로 - 로 나눠진 문자열을 다시 나눠 합을 구한다.
            while(plusToken.hasMoreTokens()) {
                plusTokenStr = plusToken.nextToken();
                plusTokenSum += Integer.parseInt(plusTokenStr);
            }

            this.result += (flag * plusTokenSum);

            // - 로 나눈 문자열 중 첫번째는 덧셈을 해야 한다.
            if(flag == 1) {
                flag = -1;
            }
        }
    }
}
```  

---
## Reference
[1541-잃어버린 괄호](https://www.acmicpc.net/problem/1541)  
