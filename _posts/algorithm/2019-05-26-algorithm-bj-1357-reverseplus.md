--- 
layout: single
classes: wide
title: "[풀이] 백준 1357 뒤집힌 덧셈"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '덧셈의 결과를 뒤집어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- 두수 x, y 가 주어졌을때 그 두 수를 뒤집는다.
	- x = 123 -> 321
	- y = 100 -> 1
- 뒤집힌 두 수를 더한 수를 다시 뒤집는다.
	- x + y = 322 -> 223
- rev(rev(x) + rev(y)) = 223

## 입력
첫째 줄에 수 X와 Y가 주어진다. X와 Y는 1,000보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 문제의 정답을 출력한다.

## 예제 입력

```
123 100
```  

## 예제 출력

```
223
```  

## 풀이
- 입력 받은 숫자와 뒤집은 숫자의 덧셈의 결과를 스택을 이용해 뒤집어 결과를 도출해 낸다.

```java
public class Main {
    private int result;
    private int num1, num2;

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

        try {
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            this.num1 = Integer.parseInt(token.nextToken());
            this.num2 = Integer.parseInt(token.nextToken());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        this.result = this.rev(this.rev(this.num1) + this.rev(this.num2));
    }

    // 스택에 넣어 수를 reverse 시킨다.
    public int rev(int num) {
        LinkedList<Integer> stack = new LinkedList<>();
        int rev = 0;

        while (num > 0) {
            stack.addLast(num % 10);
            num /= 10;
        }

        while (!stack.isEmpty()) {
            rev += stack.removeFirst() * Math.pow(10, stack.size());
        }

        return rev;
    }
}
```  

---
## Reference
[1357-뒤집힌 덧셈](https://www.acmicpc.net/problem/1357)  
