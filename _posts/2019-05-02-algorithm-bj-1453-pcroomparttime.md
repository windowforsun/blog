--- 
layout: single
classes: wide
title: "[풀이] 백준 1453 피시방 알바"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '피시방에서 거절 당하는 손님의 수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
---  

# 문제
- 어느 피시방에서 100번 까지의 좌석이 있다.
- 손님들은 자신이 원하는 자리에만 앉고 싶어 한다.
- 피시방을 들어올 때 한명씩 자신이 앉고 싶어하는 좌석 번호를 말할때 비어있으면 사용가능하고, 비어있지 않으면 거절당한다.
- 손님 중 거절당하는 손님의 수를 구하라.

## 입력
첫째 줄에 손님의 수 N이 주어진다. N은 100보다 작거나 같다. 둘째 줄에 손님이 들어오는 순서대로 각 손님이 앉고 싶어하는 자리가 입력으로 주어진다.

## 출력
첫째 줄에 거절당하는 사람의 수를 출력한다.

## 예제 입력

```
3
1 2 3
```  

## 예제 출력

```
0
```  

## 풀이
- 번호에 해당하는 자리가 사용중인지 체크하는 불린 배열을 둬서 거절당하는 손님을 카운트 한다.

```java
public class Main {
	// 출력 결과
    private int result;
    // 입력 받을 개수
    private int numCount;
    // 입력받은 수 저장 배열
    private int[] array;

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args){
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.numCount = Integer.parseInt(reader.readLine());
            this.array = new int[this.numCount];
            this.result = 0;

            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int num;

            for(int i = 0; i < this.numCount; i++) {
                num = Integer.parseInt(token.nextToken());

                this.array[i] = num;
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 체크할 때 쓸 불린 배열
        boolean[] check = new boolean[101];

        for(int i = 0; i < this.numCount; i++) {
            if(check[this.array[i]]) {
                // 자리가 이미 사용중이면 결과값 카운트
               this.result++;
            } else {
                check[this.array[i]] = true;
            }
        }
    }
}
```  

---
## Reference
[1453-피시방 알바](https://www.acmicpc.net/problem/1453)  
