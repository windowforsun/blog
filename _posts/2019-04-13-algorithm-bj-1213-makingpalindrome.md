--- 
layout: single
classes: wide
title: "[풀이] 백준 1213 팰린드롬 만들기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '입력받은 문자열로 사전순으로 가장 빠른 팰린드롬을 만들자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Brute Force
  - Sorting
---  

# 문제
- 특정 문자열이 입력으로 주어진다.
- 주어진 문자열로 팰린드롬을 만들어야 하는데, 사전순으로 가장 빠른 팰린드롬을 만들어라.

## 입력
첫째 줄에 임한수의 영어 이름이 있다. 알파벳 대문자로만 된 최대 50글자이다.

## 출력
첫째 줄에 문제의 정답을 출력한다. 만약 불가능할 때는 "I'm Sorry Hansoo"를 출력한다. 정답이 여러 개일 경우에는 사전순으로 앞서는 것을 출력한다.

## 예제 입력

```
AABB
```  

## 예제 출력

```
ABBA
```  

## 풀이
- 주어진 문자열의 최대 길이가 50이기 때문에 Brute Force 로 문제를 해결해도 무방하다.
- 하지만 팰린드롬의 특성을 이용해서 조건에 맞는 문자열을 넣어 문제를 해결 해보자.
- 팰린드롬을 만들기 위해서는 홀수인 문자의 개수가 1개이거나 없어야 한다.
	- aabc : 팰린드롬 X
	- aabccc : 팰린드롬 X
	- aab : 팰린드롬 O
	- aabb : 팰린드롬 O
- 사전순에서 가장 빠른 팰린드롬을 만들어야 하기 때문에 a 부터 시작해서 z 까지 먼저 문자를 넣어 준다.
- 문자의 개수가 홀수인것이 있다면 한개만 넣어준다.
- 마지막에는 남은 문자의 개수를 z 부터 시작해서 a 까지 문자를 넣어 준다.

```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 알파벳 개수
    final int ALPHABET = ('z' - 'a') + 1;
    // 입력 받은 문자열
    private String inputStr;
    // 입력 받은 문자열의 알파뱃 개수 저장
    private int[] alphabetCountArray;

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
            this.inputStr = reader.readLine();
            this.result = new StringBuilder();
            this.alphabetCountArray = new int[ALPHABET];
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int len = inputStr.length(), oddIndex = -1, addCount;

        // 입력 받은 문자열의 알파벳 개수를 카운트
        for (int i = 0; i < len; i++) {
            this.alphabetCountArray[this.inputStr.charAt(i) - 'A']++;
        }

        // a~z 까지 탐색
        for (int i = 0; i < ALPHABET; i++) {

            // 현재 알파뱃의 카운트
            addCount = this.alphabetCountArray[i];

            // 입력 받은 문자열 중 있었으면
            if(addCount > 0) {

                // 개수가 홀수 일 경우
                if (addCount % 2 != 0) {

                    // 2개 이상일면 팰린드롬을 만들 수 없다.
                    if (oddIndex != -1) {
                        this.result = new StringBuilder();
                        this.result.append("I'm Sorry Hansoo");
                        return;
                    }

                    // 카운트가 홀수인 인덱스 저장
                    oddIndex = i;
                }

                // 카운트 값의 절반 만 우선 결과 값에 넣어준다.
                addCount = addCount / 2;
                // 남은 개수 갱신
                this.alphabetCountArray[i] -= addCount;

                for (int j = 0; j < addCount; j++) {
                    this.result.append(String.valueOf((char) (i + 'A')));
                }
            }
        }

        // 홀수인 인덱스가 있으면 결과값에 한개 추가한다.
        if(oddIndex != -1) {
            this.result.append(String.valueOf((char)(oddIndex + 'A')));
            this.alphabetCountArray[oddIndex]--;
        }

        // z~a까지 탐색해서 나머지 카운트 값만큼 결과 값에 넣어준다.
        for (int i = ALPHABET - 1; i >= 0; i--) {
            addCount = this.alphabetCountArray[i];

            if(addCount > 0) {
                for(int j = 0; j < addCount; j++) {
                    this.result.append(String.valueOf((char)(i + 'A')));
                }
            }
        }
    }
}
```  

---
## Reference
[1213-팰린드롬 만들기](https://www.acmicpc.net/problem/1213)  
