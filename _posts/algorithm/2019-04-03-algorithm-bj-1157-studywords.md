--- 
layout: single
classes: wide
title: "[풀이] 백준 1157 단어 공부"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '문자열이 주어질 때 가장 많이 사용한 알파벳을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - String
---  

# 문제
- 알파벳 대소문자로 구성된 문자열이 주어진다.
- 이 문자열에서 가장 많이 상요된 알파벳이 무엇인지 구하라.
- 대소문자를 구분하지 않는다.

## 입력
첫째 줄에 알파벳 대소문자로 이루어진 단어가 주어진다. 주어지는 단어의 길이는 1,000,000을 넘지 않는다.

## 출력
첫째 줄에 이 단어에서 가장 많이 사용된 알파벳을 대문자로 출력한다. 단, 가장 많이 사용된 알파벳이 여러 개 존재하는 경우에는 ?를 출력한다.

## 예제 입력

```
Mississipi
```  

## 예제 출력

```
?
```  

## 예제 입력

```
zZa
```  

## 예제 출력

```
Z
```  

## 예제 입력

```
z
```  

## 예제 출력

```
Z
```  

## 예제 입력

```
baaa
```  

## 예제 출력

```
A
```  

## 풀이
- 주어진 문자열에서 어느 알파벳이 가장 많이 사용하였는지 구해내야 한다.
- 해결하기 위해서는 주어진 문자열에 있는 모든 알파벳이 몇개 사용하였는지 카운트 해주면 된다.
- 모든 알파벳을 카운트 하기 위해 알파벳 크기 만큼의 배열을 선언해 두고 문자열을 순차적으로 탐색하며 이를 배열의 인덱스에 매핑 시켜 카운트 한다.
- 가장 큰 알파벳 카운트 값이 중복된다면 ?를 출력해야 하기 때문에 이를 위해 알파벳 배열을 한번더 탐색하며 검사해주면 된다.
  
```java
public class Main {
    // 알파벳 개수 상수
    private final int ALPHABET_MAX = 'z' - 'a' + 1;
    // 출력 결과 저장
    private String result;
    // 입력 받은 문자열
    private String inputStr;
    // 알파벳 개수 카운트 배열
    private int[] alphabetCounts;

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
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int len = this.inputStr.length(), alphabetIndex, maxCount = -1, maxIndex = 0;

        this.alphabetCounts = new int[ALPHABET_MAX];
        // 입력 받은 문자열을 모두 소문자로 처리한다.
        this.inputStr = this.inputStr.toLowerCase();

        // 입력 받은 문자열 개수만큼 반복
        for (int i = 0; i < len; i++) {
            // 문자의 인덱스 0 ~ 25
            alphabetIndex = this.inputStr.charAt(i) - 'a';

            // 문자 인덱스에 해당하는 카운트 증가
            this.alphabetCounts[alphabetIndex]++;

            // 현재 최대 카운트 값 및 인덱스 저장 변수 갱신
            if (this.alphabetCounts[alphabetIndex] > maxCount) {
                maxCount = this.alphabetCounts[alphabetIndex];
                maxIndex = alphabetIndex;
            }

        }

        // 최대 카운트 값이 다른 알파벳 카운트과 동일한지 검사
        for (int i = 0; i < ALPHABET_MAX; i++) {
            if (i != maxIndex && maxCount == this.alphabetCounts[i]) {
                maxIndex = -1;
                break;
            }
        }

        if(maxIndex < 0) {
            // 동일한 카운트 값이 있으면 ? 로 설정
            this.result = "?";
        } else {
            // 최대 카운트에 해당하는 인덱스를 알파벳 대문자로 변환
            this.result = String.valueOf((char)(maxIndex + 'A'));
        }
    }
}
```  

---
## Reference
[1157-단어 공부](https://www.acmicpc.net/problem/1157)  
