--- 
layout: single
classes: wide
title: "[풀이] 백준 1181 단여 정렬"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '특정 조건에 맞게 단어를 정렬해 보자'
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

## 입력
첫째 줄에 단어의 개수 N이 주어진다. (1≤N≤20,000) 둘째 줄부터 N개의 줄에 걸쳐 알파벳 소문자로 이루어진 단어가 한 줄에 하나씩 주어진다. 주어지는 문자열의 길이는 50을 넘지 않는다.

## 출력
조건에 따라 정렬하여 단어들을 출력한다. 단, 같은 단어가 여러 번 입력된 경우에는 한 번씩만 출력한다.

## 예제 입력

```
13
but
i
wont
hesitate
no
more
no
more
it
cannot
wait
im
yours
```  

## 예제 출력

```
i
im
it
no
but
more
wait
wont
yours
cannot
hesitate
```  

## 풀이
  
```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 입력 받는 문자열의 수
    private int count;
    // 입력 문자열 저장
    private String[] strArray;

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
            this.strArray = new String[this.count];

            for(int i = 0; i < this.count; i++) {
                this.strArray[i] = reader.readLine();
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.print(this.result);
    }

    public void solution() {
        // 중복 문자열 제거를 위해 Set 에 저장
        HashSet<String> set = new HashSet<>(Arrays.asList(this.strArray));
        // 중복 문자열 제거후 문자열의 개수
        int size = set.size();

        // 정렬을 위해 다시 배열으로
        this.strArray = set.toArray(new String[size]);

        // 정렬
        Arrays.sort(this.strArray, new Comparator<String>() {
            @Override
            public int compare(String o1, String o2) {
                int len1 = o1.length(), len2 = o2.length();
                // 두 문자열의 길이의 차
                int result = len1 - len2;

                // 두 문자열의 길이가 같다면
                if(result == 0) {
                    // 사전 순으로 정렬
                    result = o1.compareTo(o2);
                }

                return result;
            }
        });

        // 정렬 값 출력 결과값에 저장
        for(int i = 0; i < size; i++) {
            this.result.append(this.strArray[i]).append("\n");
        }
    }
}
```  

---
## Reference
[1181-단여 정렬](https://www.acmicpc.net/problem/1181)  
