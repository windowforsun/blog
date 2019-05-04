--- 
layout: single
classes: wide
title: "[풀이] 백준 1292 쉽게 푸는 문제"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '특정 수열이 주어질 때 구간의 합을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
---  

# 문제
- 1은 1번, 2는 2번, 3은 3번 ... 순서로 진행되는 수열이 있다.
	- 1, 2, 2, 3, 3, 3, 4, 4, 4, 4 ...
- 위 수열에서 특정 구간의 합을 구하라.

## 입력
첫째 줄에 구간의 시작과 끝을 나타내는 정수 A, B(1≤A≤B≤1,000)가 주어진다. 즉, 수열에서 A번째 숫자부터 B번째 숫자까지 합을 구하면 된다.

## 출력
첫 줄에 구간에 속하는 숫자의 합을 출력한다.

## 예제 입력

```
3 7
```  

## 예제 출력

```
15
```  

## 풀이
- 주어지는 최대 범위는 1 ~ 1000 이 될 수 있다.
- 끝나는 정수 End 값의 크기 만큼 해당 N번째 까지의 합을 저장하는 세그먼트 배열을 선언해 사용한다.
- 1~10 이 있을때 세그먼트 배열 S 을 만들면 아래와 같다.
	
	S|1|2|3|4|5|6|7|8|9|10
	---|---|---|---|---|---|---|---|---|---|---
	 |1|3|6|10|15|21|28|36|45|55

- 위 배열을 가지고 2~4 까지의 합을 구하고 싶다면 아래와 같은 수식을 세울 수 있다.
 	- `S[4] - S[2 - 1] = (4+3+2+1) - (1) = 4+3+2 = 9` 가 된다.
	
```java
public class Main {
    // 출력 결과
    private int result;
    // 시작 순서
    private int start;
    // 끝 순서
    private int end;
    // 세그먼트배열
    private int[] segments;

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
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            this.start = Integer.parseInt(token.nextToken());
            this.end = Integer.parseInt(token.nextToken());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        this.segments = new int[this.end + 1];

        int count = 1, num = 1;
        // 현재 순서를 나타내는 count 부터 end 값까지
        while(count <= this.end) {

            // 순서 count 를 증가하면서 숫자 num 과 전 세그먼트 값을 더해줌
            for(int i = 0; i < num; i++) {
                if(count > this.end) {
                    break;
                }
                this.segments[count] += segments[count - 1] + num;
                count++;
            }

            num++;
        }

        // start, end 의 사이의 합은 end 까지의 합 - (start - 1) 까지의 합
        this.result = this.segments[this.end] - this.segments[this.start - 1];
    }
}
```  

---
## Reference
[1292-쉽게 푸는 문제](https://www.acmicpc.net/problem/1292)  
