--- 
layout: single
classes: wide
title: "[풀이] 백준 1110 더하기 사이클"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '숫자에 대해 특정 사이클로 연산을 반복할 경우 다시 처음 숫자로 돌아오는 사이클의 길이를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- 0 ~ 99 까지의 숫자가 있다.
- 주어진 숫자에 대해 다음 연산을 반복한다.
	- 10보다 작을 경우 앞에 0을 붙여 두자리 수로 만든다.
	- 2자리 수를 더한다.
	- 주어진 숫자의 가장 오른쪽 값과 더해서 나온 값의 가장 오른쪽 값을 이어 붙인다.
- 주어진 숫자가 26일 경우
	- 2 + 6 = 8 -> 68
	- 6 + 8 = 14 -> 84
	- 8 + 4 = 12 -> 42
	- 4 + 2 = 6 -> 26
- 연산을 반복했을 때 주어진 숫자(원래 수)가 나오는 사이클의 길이를 구하라.

## 입력
첫째 줄에 N이 주어진다. N은 0보다 크거나 같고, 99보다 작거나 같은 정수이다.

## 출력
첫째 줄에 N의 사이클 길이를 출력한다.

## 예제 입력

```
26
```  

## 예제 출력

```
4
```  

## 예제 입력

```
55
```  

## 예제 출력

```
3
```  

## 예제 입력

```
1
```  

## 예제 출력

```
60
```  

# 풀이
- 주어진 수 N의 2자리를 더하는 방법은 아래와 같다.
	- 첫째 자리 = N / 10
	- 둘째 자리 = N % 10
- 새로운 수를 만드는 방법은 아래와 같다.
	- 새로운 수 = (주어진 수 % 10) * 10
	- 새로운 수 += 주어진 수 덧셈 % 10

```java
public class Main {
    public TestHelper testHelper;
    private int result;
    private int num;

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
            this.num = Integer.parseInt(reader.readLine());
        } catch(Exception e) {
            e.printStackTrace();
        }

    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int sum = 0, tmp = this.num, front, back, next = -1;

        // 다음 숫자가 주어진 숫자와 다를 때까지 반복
        while(this.num != next) {
            // 첫번째 수
            front = tmp / 10;
            // 두번째 수
            back = tmp % 10;

            // 두 수의 합산
            sum = front + back;

            // 주어진 수의 오른쪽 수
            next = (tmp % 10) * 10;
            // 합산된 수의 오른쪽 수
            next += sum % 10;

            this.result++;
            
            tmp = next;
        }
    }
}
```  

---
## Reference
[1110-더하기 사이클](https://www.acmicpc.net/problem/1110)  
