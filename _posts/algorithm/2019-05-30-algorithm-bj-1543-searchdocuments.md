--- 
layout: single
classes: wide
title: "[풀이] 백준 1543 문서 검색"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 문자열에서 중복되지 않는 문자열의 개수를 알아내자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - String
  - Greedy
  - Searching
---  

# 문제
- 영문으로만 구성된 문서 문자열이 주어진다.
- 문서에서 찾을 문자열도 함께 주어질 때, 중복을 제외한 찾는 문자열의 등장 횟수를 구하라.
	- 문서 :  abababa
	- 찾을 문자열 : ababa
	- 등장 횟수 : 1

## 입력
첫째 줄에 문서가 주어진다. 문서의 길이는 최대 2500이다. 둘째 줄에 검색하고 싶은 단어가 주어진다. 이 길이는 최대 50이다. 문서와 단어는 알파벳 소문자와 공백으로 이루어져 있다.

## 출력
첫째 줄에 중복되지 않게 최대 몇 번 등장하는지 출력한다.

## 예제 입력

```
ababababa
aba
```  

## 예제 출력

```
2
```  

## 풀이
- 주어진 문서 문자열의 앞부분부터 찾을 문자열을 찾고 카운트한다.
- 찾을 문자열을 포함해서 문자열을 자르고, 다시 반복한다.

```java
public class Main {
    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    private int result;
    private String input;
    private String target;

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.input = reader.readLine();
            this.target = reader.readLine();
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int index, targetLen = this.target.length();

        // 찾는 문자열이 있다면, 찾는 문자열을 포함해서 문자열을 자른다.
        while((index = this.input.indexOf(this.target)) != -1) {
            this.input = this.input.substring(index + targetLen, this.input.length());
            this.result++;
        }
    }
}
```  

---
## Reference
[1543-문서 검색](https://www.acmicpc.net/problem/1543)  
