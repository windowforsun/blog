--- 
layout: single
classes: wide
title: "[풀이] 백준 1026 보물"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '두 배열을 정렬해서 최소 합산 값을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Sorting
---  

# 문제
- 길이가 N 인 배열 A 와 B 가 있다.
- 함수 S 를 아래와 같이 정의한다.
	- `S = A[0]*B[0] + ... + A[N-1]*B[N-1]`
- S 값을 가장 작게 만들기 위해 A 배열은 재배열 할 수 있지만, B 배열은 재배열 하지 못한다.
- S 의 최솟값을 구하라.

## 입력
첫째 줄에 N이 주어진다. 
둘째 줄에는 A에 있는 N개의 수가 순서대로 주어지고, 셋째 줄에는 B에 있는 수가 순서대로 주어진다. 
N은 50보다 작거나 같은 자연수이고, A와 B의 각 원소는 100보다 작거나 같은 음이 아닌 정수이다.

## 출력
첫째 줄에 S의 최솟값을 출력한다.

## 예제 입력

```
5
1 1 1 6 0
2 7 8 3 1
```  

## 예제 출력

```
18
```  

# 풀이
- 문제에서 재정렬 가능한 배열은 A 배열 뿐이지만, 실제 코드에서는 A, B 두 배열 모두 정렬을 통해 문제를 해결한다.
- A 배열 (1, 1, 1, 6, 0), B 배열 (2, 7, 8, 3, 1)
	- 두 배열을 오름차순으로 정렬 한다.
		- A : 0, 1, 1, 1, 6
		- B : 1, 2, 3, 7, 8
	- 함수 S 의 결과값을 구하는데 A 배열은 첫 번째 원소 부터, B 배열은 마지막 원소 부터 시작하고 인덱스 순으로 배열하면 아래와 같다.
		- A : 0, 1, 1, 1, 6
		- B : 8, 7, 3, 2, 1
		- `S = 0*8 +  1*7 + 1*3 + 1*2 + 6*1 = 18`
	- 최종적으로 재배열된 A 배열과 재배열 되지 않은 B 배열은 아래와 같다.
		- A : 1, 1, 0, 1, 6
		- B : 2, 8, 8, 3, 1

```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 배열 사이즈
    private int size;
    // A 배열
    private int[] aArray;
    // B 배열
    private int[] bArray;

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

        try{
            this.size = Integer.parseInt(reader.readLine());
            this.aArray = new int[this.size];
            this.bArray = new int[this.size];

            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            for(int i = 0; i < this.size; i++) {
                this.aArray[i] = Integer.parseInt(token.nextToken());
            }

            token = new StringTokenizer(reader.readLine(), " ");

            for(int i = 0; i < this.size; i++) {
                this.bArray[i] = Integer.parseInt(token.nextToken());
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // S 함수 합산 결과
        int resultSum = 0;

        // A 배열을 오름차순으로 정렬
        Arrays.sort(this.aArray);
        // B 배열을 오름차순으로 정렬
        Arrays.sort(this.bArray);

        // 배열 크기만큼 Loop
        for(int i = 0, j = this.size - 1; i < this.size; i++, j--) {
            // 오름차순으로 정렬된 두 배열에서 A 배열은 0번째 부터, B 배열은 size -1 번째 부터 시작함
            resultSum += (this.aArray[i] * this.bArray[j]);
        }

        this.result.append(resultSum);
    }
}
```  

---
## Reference
[1026-보물](https://www.acmicpc.net/problem/1026)  
