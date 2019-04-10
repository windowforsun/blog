--- 
layout: single
classes: wide
title: "[풀이] 백준 1212 8진수 2진수"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '8진수를 2진수로 바꿔 출력하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Numeral System
---  

# 문제
- 8진수가 주어졌을 때 이를 2진수로 변환하여 출력한다.

## 입력
첫째 줄에 8진수가 주어진다. 주어지는 수의 길이는 333,334을 넘지 않는다.

## 출력
첫째 줄에 주어진 수를 2진수로 변환하여 출력한다. 수가 0인 경우를 제외하고는 반드시 1로 시작해야 한다.

## 예제 입력

```
314
```  

## 예제 출력

```
11001100
```  

## 풀이
- 111 이라는 8진수가 있을 때 아래와 같이 생각해 볼 수 있다.
	
	.|1|1|1|
	---|---|---|---|
	10진수|64|8|1
	2진수|1000000|1000|1

- 최종적으로 2진수는 1001001 이나온다.
- 위의 경우에서 알 수 있듯이 8진수에서 자리수가 증가할 때마다 2진수는 3개의 숫자가 추가된다.
- 그리고 우리는 이미 8진수의 각 자리수에 오게되는 0 ~ 7에 대한 2진수를 알고 있다.
	
```java
public class Main {
    private StringBuilder result;
    private String octaNum;

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
            this.octaNum = reader.readLine();
            this.result = new StringBuilder();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int len = this.octaNum.length(), tmp;
        String tmpStr;

        // 0 ~ 7 까지 이진수
        String[] octaToBinary = new String[]{
                "000", "001", "010", "011", "100", "101", "110", "111"
        };

        for (int i = 0; i < len; i++) {
            // 8진수의 숫자 한자리씩
            tmp = this.octaNum.charAt(i) - '0';
            // 2진수로 변경함
            tmpStr = octaToBinary[tmp];

            // 첫째 자리일 경우
            if(i == 0 && tmpStr.charAt(0) == '0') {
                // 앞에 0을 제거한다.
                tmpStr = (Integer.parseInt(tmpStr)) + "";
            }

            this.result.append(tmpStr);
        }
    }
}
```  

---
## Reference
[1212-8진수 2진수](https://www.acmicpc.net/problem/1212)  
