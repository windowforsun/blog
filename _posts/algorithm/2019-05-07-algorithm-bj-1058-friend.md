--- 
layout: single
classes: wide
title: "[풀이] 백준 1058 친구"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '특정 조건의 가장 많은 친구 수를 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Bitmask
---  

# 문제
- 특정 조건의 친구 수를 카운트한다.
- 조건은 아래와 같다.
	- 나와 직접적으로 친구이다. (A-B)
	- 나와 한다리 걸쳐서 친구이다.(A-B-C)
- 위 조건으로 친구를 카운트 했을 때 가장 많은 친구 수를 구하라.

## 입력
첫째 줄에 사람의 수 N이 주어진다. N은 50보다 작거나 같은 자연수이다. 둘째 줄부터 N개의 줄에 각 사람이 친구이면 Y, 아니면 N이 주어진다. (예제를 참고)

## 출력
첫째 줄에 가장 유명한 사람의 2-친구의 수를 출력한다.

## 예제 입력

```
3
NYY
YNY
YYN
```  

## 예제 출력

```
2
```  

## 풀이
- 비트마스트를 이용해서 친구 정보를 저장한다.
- 최대 사람 수가 50명이기 때문에 비트마스크로 50명의 친구 정보를 저장하기 위해서는 2^50 까지 필요하므로 long 형을 사용한다.
- OR 연산을 통해 i 번째가 친구라면 비트 연산으로 친구를 추가한다.
- AND 연산을 통해 결과값이 0 이상이면 1명 이상 조건에 맞는 사람이 있음므로 친구이다.
- 비트마스크로 해당 문제를 풀면 비교적 빠른 성능을 보일 수 있지만 long 형이 자료형의 최대이기 때문에 최대 64 명까지의 친구정보 밖에 저장 하지 못한다.

```java
public class Main {
    // 출력 결과 저장
    private int result;
    // 총 인원 수
    private int count;
    // 친구 정보 저장
    private long[] friendMarkArray;
    private final long BIT = 1;

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
            this.count = Integer.parseInt(reader.readLine());
            this.friendMarkArray = new long[this.count + 1];

            String str;
            long mark;

            for(int i = 0; i < this.count; i++) {
                str = reader.readLine();
                mark = 0;
                for(int j = 0; j < this.count; j++) {
                    // 친구일 경우 비트마스트를 통해 친구 정보를 저장
                    if(str.charAt(j) == 'Y') {
                        mark = mark | BIT << j;
                    }
                }

                this.friendMarkArray[i] = mark;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        long value;
        int friendCount;

        for(int i = 0; i < this.count; i++) {
            friendCount = 0;

            // 친구 정보에서 자신까지 포함해서 비트마스트 저장
            value = this.friendMarkArray[i] | BIT << i;
            for(int j = 0; j < this.count; j++) {
                // i번째의 친구와 과 j번째 친구 중 겹치는 친구가 1명이상 있으면 카운트 값 증가
                if(i != j && (this.friendMarkArray[j] & value) > 0) {
                    friendCount++;
                }
            }

            // 최대 값으로 설정
            if(friendCount > this.result) {
                this.result = friendCount;
            }
        }
    }
}
```  

---
## Reference
[1058-친구](https://www.acmicpc.net/problem/1058)  
