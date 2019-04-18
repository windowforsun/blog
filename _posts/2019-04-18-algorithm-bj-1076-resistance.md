--- 
layout: single
classes: wide
title: "[풀이] 백준 1076 저항"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '3가지 색으로 저항의 값을 구하보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제

색|값|곱
---|---|---
black|0|1
brown|1|10
red|2|100
orange|3|1000
yellow|4|10000
green|5|100000
blue|6|1000000
violet|7|10000000
grey|8|100000000
white|9|1000000000

- 3개의 색이 주어지는데, 첫 번째, 두 번째 색은 저항의 값이고, 마지막 색은 곱해야 하는 값이다.
- 이를 이용해서 3개 가지 색으로 저항의 값을 구하라


## 입력
첫째 줄에 첫 번째 색, 둘째 줄에 두 번째 색, 셋째 줄에 세 번째 색이 주어진다. 색은 모두 위의 표에 쓰여 있는 색만 주어진다.

## 출력
입력으로 주어진 저항의 저항값을 계산하여 첫째 줄에 출력한다.

## 예제 입력

```
yellow
violet
red
```  

## 예제 출력

```
4700
```  

## 풀이
- 저항의 색과 값을 표현하기 위해서 Resistance 라는 enum 을 사용하였다.
	- black 은 값이 0, brown 은 1 순으로 증가한다.
- 입력 받은 3개의 색을 순차적으로 탐색하며 enum 을 사용해서 두번째 까지는 값을 더해주고 세번째 에서는 곱해준다.

```java
public class Main {
    // 저항 색에 대한 enum
    enum Resistance {
        black, brown, red, orange, yellow, green, blue, violet, grey, white;
    }

    private long result;
    private String first;
    private String second;
    private String third;

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
            this.first = reader.readLine();
            this.second = reader.readLine();
            this.third = reader.readLine();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        String[] inputColors = new String[]{this.first, this.second, this.third};
        String inputColor;
        int value, pow;

        // 립력 받느 순서대로 탐색
        for (int i = 0; i < 3; i++) {
            inputColor = inputColors[i];

            // enum 에서 해당하는 색 탐색
            for (Object color : Resistance.values()) {

                // 같은 색을 찾으면
                if (inputColor.equals(color + "")) {
                    // enum 의 순서 값이 바로 해당 색의 값
                    value = Resistance.valueOf(color + "").ordinal();

                    if (i < 2) {
                        // 첫 번째, 두 번째 색은 값을 뒤에 이어서 붙여줌
                        pow = (int) Math.pow(10, i);
                        this.result *= pow;
                        this.result += value;
                    } else {
                        // 마지막 색은 해당 곱만큼 곱해줌
                        pow = (int) Math.pow(10, value);
                        this.result *= pow;

                    }
                    break;
                }
            }
        }
    }
}
```  

---
## Reference
[1076-저항](https://www.acmicpc.net/problem/1076)  
