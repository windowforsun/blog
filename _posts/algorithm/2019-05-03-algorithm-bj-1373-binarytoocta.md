--- 
layout: single
classes: wide
title: "[풀이] 백준 1373 2진수 8진수"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '2진수를 8진수로 변환하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
---  

# 문제
- 2진수가 주어지면 8진수로 변환한다.

## 입력
첫째 줄에 2진수가 주어진다. 주어지는 수의 길이는 1,000,000을 넘지 않는다.

## 출력
첫째 줄에 주어진 수를 8진수로 변환하여 출력한다.

## 예제 입력

```
11001100
```  

## 예제 출력

```
314
```  

## 풀이
- 2진수는 0,1로 이루어져 있고, 8진수는 0,1,2,3,4,5,6,7로 이루어져 있다.
- 2진수를 8진수로 변환하는 방법은 2진수는 2개의 수, 8진수는 8개의 수이기 때문에 2^3 = 8 이므로 2진수를 3개씩 짤라 이를 8진수로 계산해 주면 된다.

```java
public class Main {
    private StringBuilder result;
    private String binaryString;

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
            this.binaryString = reader.readLine();
            this.result = new StringBuilder();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int tmp, len = this.binaryString.length(), treeCount = 3;
        int quotient = len / 3;
        int remainder = len % 3;
        String tmpStr = "";
        int[] pow = new int[]{
                1, 2, 4
        };

        // 입력 받은 바이너리 문자열을 8진수 단위로 맞춤
        if (remainder != 0) {
            tmp = ((quotient + 1) * 3) - len;
            for (int i = 0; i < tmp; i++) {
                tmpStr += "0";
            }

            this.binaryString = tmpStr + this.binaryString;
            len = this.binaryString.length();
        }

        tmp = 0;
        // 앞에서부터 3비트 씩 짤라서 8진수로 변환
        for (int i = 0; i < len; i++) {
            treeCount--;
            tmp += ((this.binaryString.charAt(i) - '0') * pow[treeCount]);
            if (treeCount == 0) {
                treeCount = 3;
                this.result.append(tmp);
                tmp = 0;
            }
        }
    }
}
```  

---
## Reference
[1373-2진수 8진수](https://www.acmicpc.net/problem/1373)  
