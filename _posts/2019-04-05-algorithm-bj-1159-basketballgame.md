--- 
layout: single
classes: wide
title: "[풀이] 백준 1159 농구 경기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '문자열 배열에서 이름의 첫글자가 같은 알파벳들을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - String
  - Loop
---  

# 문제
- N 개의 선수 이름이 주어진다.
- 주어진 선수 이름의 첫글자가 같은 개수가 5개 이상인 알파벳을 구하라.

## 입력
첫째 줄에 선수의 수 N (1 ≤ N ≤ 150)이 주어진다. 다음 N개 줄에는 각 선수의 성이 주어진다. (성은 알파벳 소문자로만 이루어져 있고, 최대 30글자이다)

## 출력
상근이가 선수 다섯 명을 선발할 수 없는 경우에는 "PREDAJA" (따옴표 없이)를 출력한다. PREDAJA는 크로아티아어로 항복을 의미한다. 선발할 수 있는 경우에는 가능한 성의 첫 글자를 사전순으로 공백없이 모두 출력한다.

## 예제 입력

```
18
babic
keksic
boric
bukic
sarmic
balic
kruzic
hrenovkic
beslic
boksic
krafnic
pecivic
klavirkovic
kukumaric
sunkic
kolacic
kovacic
prijestolonasljednikovi
```  

## 예제 출력

```
bk
```  

## 풀이
  
```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 총 개수
    private int count;
    // 입력 받은 이름 배열
    private String[] nameArray;

    public Main() {
        this.result = new StringBuilder();
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
            this.count = Integer.parseInt(reader.readLine());
            this.nameArray = new String[this.count];

            for(int i = 0; i < this.count; i++) {
                this.nameArray[i] = reader.readLine();
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 알파벳 개수
        final int ALPHABET = 'z' - 'a' + 1;
        // 알파뱃 카운트 배열
        int[] alphabetCountArray = new int[ALPHABET];
        int alphabetIndex;

        for(int i = 0; i < this.count; i++) {
            // 현재 이름의 첫 글자의 알파벳 인덱스
            alphabetIndex = this.nameArray[i].charAt(0) - 'a';

            // 알파벳 카운트 배열에서 인덱스 부분 증가
            alphabetCountArray[alphabetIndex]++;
        }

        // 알파벳 카운트 배열에서 값이 5이상일 경우 출력결과에 추가
        for(int i = 0; i < ALPHABET; i++) {
            if(alphabetCountArray[i] >= 5) {
                this.result.append(String.valueOf((char)(i + 'a')));
            }
        }

        // 해당하는 알파벳이 하나도 없을 경우
        if(this.result.length() == 0) {
            this.result.append("PREDAJA");
        }
    }
}
```  

---
## Reference
[1159-농구 경기](https://www.acmicpc.net/problem/1159)  
