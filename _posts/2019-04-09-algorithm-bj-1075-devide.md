--- 
layout: single
classes: wide
title: "[풀이] 백준 1075 나누기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: 'K로 나누어 떨어지는 특정조건의 최소 값을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- 두 수 N, F 가 주어 질때 N 이 F 로 나누어 떨어지는(N % F == 0) N 의 아래의 조건을 만족시키는 최소 값을 구하라.
	1. N 이 주어 질때 뒤의 2자리만 변경 가능하다.
- N = 275, F = 5 일경우 N = 200 답은 00 이 된다.
- N = 1021, F = 11 일 경우 N = 1001 답은 01 이 된다.

## 입력
첫째 줄에 N, 둘째 줄에 F가 주어진다. N은 100보다 크거나 같고, 2,000,000,000보다 작거나 같은 자연수이다. F는 100보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 마지막 두 자리를 모두 출력한다. 한자리이면 앞에 0을 추가해서 두 자리로 만들어야 한다.

## 예제 입력

```
1000
3
```  

## 예제 출력

```
02
```  

## 풀이
- 답을 구하는 가장 간단한 방법은 Brute Force 로 뒤에 2자리를 00 ~ 99 까지 변경해가며 모든 경우의 수에 대한 고려를 하는 것이다.
	- 총 100가지의 경우의 수만 있기 때문에 현재 문제에서는 나쁘지 않은 방법이다.
	- 하지만 변경 가능한 자리수가 3자리, 4자리 등으로 증가 될 경우 이 방법으로는 문제해결이 어렵다.
- 두 수 1021 과 11 이 있다.
	- 1021 / 11 을 했을 때 몫은 92, 나머지는 9가 된다.
	- 즉 이는 1021 의 수와 가장 가까운 수중 11로 나누어 지는 수는 1021 + 9 혹은 1021 - 9 가 된다는 의미이다.
- 구해야 하는 숫자는 1021 에서 뒤에 두자리를 변경한 수중 11 로 나누어 떨어지는 가장 작은 수이다.
	- 1021 에서 가능하나 수중 가장 작은 수는 1000 이다.
	- 1000 에서 가장 인접한 수중 11 로 나누어 떨어지는 수는 1001 이다.

```java
public class Main {
    // 출력 결과
    private StringBuilder result;
    private long N;
    private long K;

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

        try {
            this.N = Long.parseLong(reader.readLine());
            this.K = Long.parseLong(reader.readLine());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // N 이 변할 수 있는 최소값
        long min = this.N - (this.N % 100);
        // 최소값을 K 로 나누었을 때 나머지
        long mod = min % this.K;
        // 최소값 이 K 로 나누어 지기 위해 필요한 숫자
        long gap = this.K - mod;
        // 최소값에 K 로 나누어지기 위한 숫자를 더하면 K로 나누어 떨어진다.
        long num = min + gap;

        // 나머지가 0일 경우엔 0으로 설정
        if(mod == 0) {
            num = 0;
        }

        // 뒤에 2자리
        num %= 100;

        if (num < 10) {
            this.result.append("0");
        }

        this.result.append(num);
    }
}
```  

---
## Reference
[1075-나누기](https://www.acmicpc.net/problem/1075)  
