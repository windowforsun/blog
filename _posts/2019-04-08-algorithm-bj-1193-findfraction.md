--- 
layout: single
classes: wide
title: "[풀이] 백준 1193 분수찾기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '이차원 배열을 특정 방식으로 탐색할 때 순서에 해당하는 원소 값을 찾자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
---  

# 문제
- 무한히 큰 아래와 같은 배열이 있다.

	 |||||||
	 ---|---|---|---|---|---|
	1/1|	1/2|	1/3|	1/4|	1/5|	…
	2/1|	2/2|	2/3|	2/4|	…|	…
	3/1|	3/2|	3/3|	…|	…|	…
	4/1|	4/2|	…|	…|	…|	…
	5/1|	…|	…|	…|	…|	…
	…|	…|	…|	…|	…|	…
	
- 위 이차원 배열을 탐색시에 아래와 같은 대각선으로 탐색한다.
	- 1/1 > 1/2 > 2/1 > 3/1 > 2/2 > 1/3 > 1/4 > 2/3 ..
- X 의 순서가 주어질 때 X 번째의 분수를 구하라.

## 입력
첫째 줄에 X(1 ≤ X ≤ 10,000,000)가 주어진다.

## 출력
첫째 줄에 분수를 출력한다.

## 예제 입력

```
14
```  

## 예제 출력

```
2/4
```  

## 풀이
- 문제에서 주어진 2차원 배열을 아래와 같이 count 라는 대각선의 새로은 열값으로 생각해본다.

	count|1|2|3|4|5|6|
	---|---|---|---|---|---|---|
	-|1|2|3|4|5|6|
	-|2|3|4|5|6|7|
	-|3|4|5|6|7|8|
	-|4|5|6|7|8|9|
	-|5|6|7|8|9|10|

- 위의 표를 보면 6번까지의 count 가 있고 각 카운트 값은 2차원 배열에서 대각선의 열 값을 나타낸다.
	- count 2번 아래에 대각선으로 해당 열을 나타내는 2번 값이 들어가 있는 것을 확인 할 수 있다.
- 위 count 대각선 열에 문제에서 제시된 분수 값을 넣으면 아래와 같다.


	count|1|2|3|4|5|6|
	---|---|---|---|---|---|---|
	-|1(1/1)|2(1/2)|3(1/3)|4(1/4)|5(1/5)|6(1/6)|
	-|2(2/1)|3(2/2)|4(2/3)|5(2/4)|6(2/5)|7(2/6)|
	-|3(3/1)|4(3/2)|5(3/3)|6(3/4)|7(3/5)|8(3/6)|
	-|4(4/1)|5(4/2)|6(4/3)|7(4/4)|8(4/5)|9(4/6)|
	-|5(5/1)|6(5/2)|7(5/3)|8(5/4)|9(5/5)|10((5/6)|
	
- count 값에서 확인 할 수 있는 것은 아래와 같다.
	1. count = 1 일때, 열의 개수는 1개이다. 2일 때는 2개, ...
	1. count 값이 짝수 일때는 대각선 위에서부터, 홀수 일때는 대각선 아래에서부터 해당 열이 시작된다.
	1. 하나의 count 대각선 열에서 분모, 분자 두 가지의 수가 있다. 이 두가지의 수중 하나는 계속해서 감소하고, 하나는 계속해서 증가한다.
- X 가 14 일때 분수를 아래와 같은 방식으로 도출해 낼 수 있다.
	1. 1 ~ N 번 count 를 더하면서 14 보다 크지만 가장 작은 수를 찾는다.
	1. max = 1 + 2 + 3 + 4 + 5 = 15 이고, 이때 count 열의 값은 5가 된다.
	1. 5번 열이 시작하는 순서의 값은 start = 15(max) - 5(count) + 1 = 10 이된다.
	1. 해당 열에서 사용하는 2가지 값은 plus = 1, minus = 5(count) 이다.
	1. 11(start) ~ 14(X) 값 까지 1(plus)값은 증가, 5(minus)값은 감소 시킨다.
	1. plus 값은 4가 되고, minus 값은 2가 된다.
	1. count 값이 홀수 이므로 minus 값이 분자, plus 값이 분모가 되어 분수 값은 2/4 가 된다.

```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 입력 받은 수
    private int num;

    public Main() {
        this.result = new StringBuilder();
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input(){
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
        // max 는 현재 count 에서 최대 순서 값
        // count 는 대각선 열
        int max = 1, count = 1, plus, minus, start;

        // 입력 받은 num 의 count 값을 찾기 위해 max 값과 count 값을 증가시킨다.
        while(max < this.num) {
            count++;
            max += count;
        }

        // 증가 할 수
        plus = 1;
        // 감소할 수
        minus = count;
        // 현재 count 대각선 열이 시작하는 순서
        start = max - count + 1;

        // start 값 부터 입력 받은 num 까지 증가, 감소를 수행한다.
        for(int i = start; i < this.num; i++) {
            plus += 1;
            minus -= 1;
        }

        if(count % 2 == 0) {
            // 대각선 열이 짝수 일 경우 plus 가 분자 minus 를 분모로 만든다.
            this.result.append(plus).append("/").append(minus);
        } else {
            // 대각선 열이 홀수 일 경우 minus 가 분자 plus 를 분모로 만든다.
            this.result.append(minus).append("/").append(plus);
        }
    }
}
```  

---
## Reference
[1193-분수찾기](https://www.acmicpc.net/problem/1193)  
