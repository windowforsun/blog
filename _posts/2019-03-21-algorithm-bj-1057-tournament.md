--- 
layout: single
classes: wide
title: "[풀이] 백준 1057 토너먼트"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '토너먼트에서 두 사람이 만나는 라운드를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Simulation
  - Math
---  

# 문제
- N명이 토너먼트를 진행한다.
- N명의 참가자들은 1 ~ N번 까지 번호를 배정받게 된다.
- 토너먼트는 인접한 번호끼리 수행한다.
	- 1:2, 3:4, 5:6, ..
- 승리한 사람은 다음 라운드에 진출하고, 홀수의 경우 마지막 참가번호는 부전승으로 자동 진출된다.
- 다음 라운드에서도 동일하게 다시 1번 부터 번호를 매기는데, 전 라운드에서의 번호를 유지하면서 번호를 매긴다.
	- 1:2, 3:4 대결 후에 1, 4번이 승리했을 경우 다음 라운드에서 1번은 1번, 4번은 2번으로 번호가 매겨진다.
- 라운드는 최종우승 1명이 나올때까지 반복된다.
- 특정 두 사람이 토너먼트에 참가할 경우, 이 두 사람이 몇번째 라운드에서 대결하는지 알아내보자.
	- 특정 두 사람은 만나기 전까지 항상 승리한다고 가정한다.

## 입력
첫째 줄에 참가자의 수 N과 1 라운드에서 김지민의 번호와 임한수의 번호가 순서대로 주어진다. 
N은 100,000보다 작거나 같은 자연수이고, 김지민의 번호와 임한수의 번호는 N보다 작거나 같은 자연수이고, 서로 다르다.

## 출력
첫째 줄에 김지민과 임한수가 대결하는 라운드 번호를 출력한다. 
만약 서로 대결하지 않을 때는 -1을 출력한다.

## 예제 입력

```
16 8 9
```  

## 예제 출력

```
4
```  

# 풀이
- 특정 두 사람은 항상 승리하기 때문에 패배의 경우는 고려하지 않아도 된다.
- 4명의 사람이 있다고 가정했을 때, 모든 번호가 다음 라운드에서 가질수 있는 번호는 아래와 같다.

	라운드 | | | | |
	---|---|---|---|---|
	0|1|2|3|4|
	1|1|1|2|2|
	2|1|1|1|1|
	
- 5명의 사람이 있다고 가정한다면 아래와 같다.

	라운드 | | | | | |
	---|---|---|---|---|---|
	0|1|2|3|4|5
	1|1|1|2|2|3
	2|1|1|1|1|2
	3|1|1|1|1|1
	
- 특정 라운드에서 번호가 같다면 두 번호는 해당 라운드에서 만날 수 있다.
	- 1:4 번이 만나는 라운드는 2라운드이다.
	- 1:5 번이 만나는 라운드는 3라운드이다.
- 여기서 주목해야 할점은 다음 N명이 주어졌을 때 다음 라운드에 부여 받는 번호를 알아내는 방법이다.
- 다음 라운드에서 부여 받을 수 있는 번호들의 패턴을 보면 다음과 같은 수식을 도출해 낼 수 있다.
	- 다음 라운드 때 번호 = 반올림(지금 라운드 번호 / 2)
	- 반올림 대신 지금 라운드 번호 + 1을 해도 동일한 결과값이 나온다.
- 입력에서 주어지는 두 사람의 번호를 위의 수식을 반복해서 돌리면서 같은 수가 나오는 라운드가 문제의 정답이다.
- 문제 본문(링크)의 조건 중에 `만나지 못하는 경우에는 -1을 출력 한다.` 라는 조건 있는데, 두 사람은 항상 승리하기 때문에 만날 수 없는 경우는 없다.

```java
public class Main {
    private StringBuilder result;
    private int count;
    private int a;
    private int b;

    public Main() {
        this.result = new StringBuilder();
    }

    public static void main(String[] args) {
        Main main = new Main();
        main.input();
        main.solution();
        main.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            this.count = Integer.parseInt(token.nextToken());
            this.a = Integer.parseInt(token.nextToken());
            this.b = Integer.parseInt(token.nextToken());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int resultCount = 0;
        float newA = this.a, newB = this.b;

        while(newA != newB) {
            newA /= 2;
            newA = Math.round(newA);

            newB /= 2;
            newB = Math.round(newB);

            resultCount++;
        }

        this.result.append(resultCount);
    }
}
```

---
## Reference
[1026-보물](https://www.acmicpc.net/problem/1026)  
