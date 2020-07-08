--- 
layout: single
classes: wide
title: "[풀이] 백준 1120 문자열"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '두 문자열의 길이가 같을 때까지 특정 연산을 반복하여 최소 차이를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - String
  - Simulation
  - Brute Force
  - Grid Algorithm
---  

# 문제
- A, B 두 문자열이 있다.
- A, B 문자열에 대해 아래와 같은 동작을 수행한다.
	1. A, B 문자열의 길이가 같다면 다른 문자의 개수를 카운트 한다.
	1. A, B 문자열의 길이가 다를 경우는 아래 동작을 같을 때 까지 반복 수행한다.
		1. 길이가 짧은 문자열 앞에 아무 문자나 추가한다.
		1. 길이가 짧은 문자열 뒤에 아무 문자나 추가한다.
- 위와 같은 동작을 반복 수행하게 될때 A, B 문자열의 길이가 같으면서 차이가 최소를 구하라.

## 입력
첫째 줄에 A와 B가 주어진다. A와 B의 길이는 최대 50이고, A의 길이는 B의 길이보다 작거나 같고, 알파벳 소문자로만 이루어져 있다.

## 출력
A와 B의 길이가 같으면서, A와 B의 차이를 최소가 되도록 했을 때, 그 차이를 출력하시오.

## 예제 입력

```
adaabc aababbc
```  

## 예제 출력

```
2
```  

## 풀이
- 두 문자열이 같을 때 까지 반복 수행하는 동작은 앞에 아무 문자나 추가, 뒤에 아무 문자나 추가이다.
- 이를 다르게 생각하면 짧은 문자열이 긴 문자열을 기준으로 어느 인덱스에 위치하는 것이 최적이냐로 생각할 수 있다.
	- 긴 문자열의 맨 첫번째 인덱스 부터 시작해서 짧은 문자열을 우측으로 옮겨가며 문자열을 비교한다.
- "aabaa", "ab" 라는 문자열이 주어진다 가정하자.

1. |0|1|2|3|4|
	|---|---|---|---|---|
	|a|a|b|a|a|
	|a|b||||
	- 두 문자열의 차이는 1이 된다.
1. |0|1|2|3|4|
	|---|---|---|---|---|
	|a|a|b|a|a|
	||a|b|||
	- 두 문자열의 차이는 0이 된다.
1. |0|1|2|3|4|
	|---|---|---|---|---|
	|a|a|b|a|a|
	|||a|b||
	- 두 문자열의 차이는 2가 된다.
1. |0|1|2|3|4|
	|---|---|---|---|---|
	|a|a|b|a|a|
	||||a|b|
	- 두 문자열의 차이는 1이 된다.
	
- 최종적으로 결과 값은 0이 되고, 긴 문자열의 인덱스 1부터 짧은 문자열이 위치하는 경우가 정답이 된다.

```java
public class Main {
    // 출력 결과값 저장
    private int result;
    // 첫번째 문자열
    private String str1;
    // 두번째 문자열
    private String str2;

    public Main() {
        this.result = Integer.MAX_VALUE;
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

            this.str1 = token.nextToken();
            this.str2 = token.nextToken();
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int len1 = this.str1.length(), len2 = this.str2.length();
        String loggerStr, shorterStr;
        int diffLen = Math.abs(len1 - len2), loggerLen, shorterLen;

        // 입력 받은 두 문자열의 길이가 같다면
        if(diffLen == 0) {
            // 두 문자열의 차이값을 결과값으로 설정
            this.result = this.diffStr(this.str1, this.str2, len1);
        } else {
            // 긴 문자열과 짧은 문자열 설정
            if(len1 > len2) {
                loggerStr = this.str1;
                shorterStr = this.str2;
            } else {
                loggerStr = this.str2;
                shorterStr = this.str1;
            }

            loggerLen = loggerStr.length();
            shorterLen = shorterStr.length();

            // 짧은 문자열의 길이에서 부터 긴 문자열의 길이까지 검사
            for(int i = shorterLen, j = 0; i <= loggerLen; i++, j++) {
                // 긴 문자열을 베이스로 짧은 문자열을 긴 문자열의 0 번째 문자부터 (긴문자열길이 - 짧은문자열길이) 인덱스 까지 우측으로 옮기며 비교값 도출
                diffLen = this.diffStr(shorterStr, loggerStr.substring(j, shorterLen + j), shorterLen);

                // 문자 비교값이 결과값보다 작으면 갱신
                if(this.result > diffLen) {
                    this.result = diffLen;
                }
            }
        }
    }

    // 두 문자열의 차이값을 리턴
    public int diffStr(String a, String b, int len) {
        int result = 0;

        for(int i = 0; i < len; i++) {
            if(a.charAt(i) != b.charAt(i)) {
                result++;
            }
        }

        return result;
    }
}
```  

---
## Reference
[1120-문자열](https://www.acmicpc.net/problem/1120)  
