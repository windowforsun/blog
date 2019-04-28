--- 
layout: single
classes: wide
title: "[풀이] 백준 1254 팰린드롬 만들기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 문자열에서 가장 짧은 팰린드롬 문자열을 만들어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
---  

# 문제
- 하나의 문자열이 주어진다.
- 주어진 문자열의 뒤에 문자열을 더해 팰린드롬을 만든다.
- 만들수 있는 팰린드롬 중 가장 짧은 길이의 팰린드롬을 구하라.

## 입력
첫째 줄에 문자열 S가 주어진다. S의 길이는 최대 1000이다.

## 출력
첫째 줄에 동호가 만들 수 있는 가장 짧은 팰린드롬의 길이를 출력한다.

## 예제 입력

```
abab
```  

## 예제 출력

```
5
```  

## 풀이
- 팰린드롬의 정의는 앞으로 읽으나, 뒤로 읽으나 같은 문자열이 되는 것을 뜻한다.
- abab 라는 문자열이 있을 때 가장 짧은 팰린드롬 문자열은 ababa 가 된다.
	- 원본 문자열을 뒤집어 이를 원본 문자열과 비교 했을 때 모든 문자열이 일치하면 팰린드롬이 만족된다.
	
		1|2|3|4
		---|---|---|---
		a|b|a|b
		b|a|b|a
	
	- 비교 했을 때 비교한 문자열이 만족되지 않았을 때는 뒤집은 문자열을 한칸 뒤로 미뤄 비교를 반복한다.
	
		1|2|3|4|5
		---|---|---|---|---
		a|b|a|b
		 |b|a|b|a
	
	- 모든 비교한 문자열이 만족될 때 그 문자열의 길이가 만들 수 있는 팰린드롬의 최소 길이가 된다.

```java
public class Main {
    private int result;
    private String str;

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
            this.str = reader.readLine();
            this.result = 0;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int len = this.str.length(), sameCount;

        // 문자열을 뒤집은 문자열과 원래의 문자열을 하나씩 뒤로 밀려 몇개가 같은지 탐색
        for (int i = 0; i < len; i++) {

            // 뒤집은 문자열과 같은 개수
            sameCount = 0;

            // 문자열과 뒤집은 문자열 비교
            for (int index = i, reverseIndex = len - 1; index < len; index++, reverseIndex--) {
                if (this.str.charAt(index) == this.str.charAt(reverseIndex)) {
                    // 같으면 값 증가
                    sameCount++;
                } else {
                    break;
                }
            }

            // 뒤집은 문자열과 같은 개수 == 문자열 길이 - i 라면 팰린드롬을 만들 수 있음
            if(sameCount == len - i) {
                // 만들어진 팰린드롬의 길이는 문자열 길이 + (문자열길이 - 뒤집은 문자열과 같은 개수)
                this.result = len + (len - sameCount);
                break;
            }
        }
    }
}
```  

---
## Reference
[1254-팰린드롬 만들기](https://www.acmicpc.net/problem/1254)  
