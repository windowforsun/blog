--- 
layout: single
classes: wide
title: "[풀이] 백준 1316 그룹 단어 체커"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 문자열 중에서 그룹 단어의 개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Searching
  - String
---  

# 문제
- 그룹 단어란 단어를 구성하는 문자열에 대해, 각 문자가 연속해서 나타나는 경우만을 뜻한다.
	- ccccaddddeg 는 c, a, d, e, g 로 구성된 그룹단어이다.
	- ccccacdddeg 는 c가 연속되어서 나타나지 않기 때문에 그룹단어가 아니다.
- 문자열들이 주어질 때 그룹 단어의 개수를 구하라

## 입력
첫째 줄에 단어의 개수 N이 들어온다. N은 100보다 작거나 같은 자연수이다. 둘째 줄부터 N개의 줄에 단어가 들어온다. 단어는 알파벳 소문자로만 되어있고 중복되지 않으며, 길이는 최대 100이다.

## 출력
첫째 줄에 그룹 단어의 개수를 출력한다.

## 예제 입력

```
3
happy
new
year
```  

## 예제 출력

```
3
```  

## 예제 입력

```
4
aba
abab
abcabc
a
```  

## 예제 출력

```
1
```  

## 풀이
- 알파벳을 인덱스로 매핑 시켜 배열에 해당 알파벳이 위치한 마지막 위치를 넣어 둔다.
- 알파벳의 마지막 위치를 꺼냈을 때 0이 아니면서, 자신의 인덱스에서 -1 한 인덱스와 같지 않으면 그룹 단어가 아니게 된다.

```java
public class Main {
    private int result;
    private int count;
    private String[] strArray;

    public Main() {
        this.input();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.count = Integer.parseInt(reader.readLine());
            this.strArray = new String[this.count];
            this.result = 0;

            for(int i = 0; i < this.count; i++) {
                this.solution(reader.readLine());
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution(String str) {
        // 알파벳의 총 개수
        final int ALPHABET = 'z' - 'a' + 1;
        // 문자열에서 알파벳의 마지막 위치
        int[] alphabetLastIndexArray = new int[ALPHABET];
        int len = str.length(), alphabetIndex;
        boolean isGroupWord = true;

        for(int i = 0; i < len; i++) {
            // 알파벳 인덱스
            alphabetIndex = str.charAt(i) - 'a';

            // 전에 인덱스 설정이 된 경우
            if(alphabetLastIndexArray[alphabetIndex] != 0) {
                // 바로전 인덱스가 아니라면 그룹 단어가 아님
                if(alphabetLastIndexArray[alphabetIndex] + 1 != i + 1) {
                    isGroupWord = false;
                }
            }

            // 마지막 위치 설정
            alphabetLastIndexArray[alphabetIndex] = i + 1;
        }

        if(isGroupWord) {
            this.result++;
        }
    }
}
```  

---
## Reference
[1316-그룹 단어 체커](https://www.acmicpc.net/problem/1316)  
