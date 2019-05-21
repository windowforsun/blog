--- 
layout: single
classes: wide
title: "[풀이] 백준 1145 적어도 대부분의 배수"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 숫자에서 최소가 되는 대부분의 배수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
  - Searching
---  

# 문제
- 다섯개의 수가 주어진다.
- 주어진 수 중 적어도 3개로 나누어 지는 가장 작은 자연수를 구하라.

## 입력
첫째 줄에 다섯 개의 자연수가 주어진다. 100보다 작거나 같은 자연수이고, 서로 다른 수이다.

## 출력
첫째 줄에 적어도 대부분의 배수를 출력한다.

## 예제 입력

```
30 42 70 35 90
```  

## 예제 출력

```
210
```  

## 풀이
- 주어진 5개의 수에서 3개의 최소공배수 중 가장 작은 값을 구하면 된다.
- 중첩 for문을 통해 주어진 수 중 최소공배수 연산에서 제외 2개의 시킬 인덱스를 순차적으로 정한다.
- 2개의 인덱스를 제외시키고 3개에 대해서 최소공배수를 구하면서 결과값이 되는 최소값을 구한다.

```java
public class Main {
    final int COUNT = 5;
    // 출력 결과
    private int result;
    // 입력 받은 수
    private int[] numArray;

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
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            this.numArray = new int[COUNT];
            this.result = Integer.MAX_VALUE;

            for (int i = 0; i < COUNT; i++) {
                this.numArray[i] = Integer.parseInt(token.nextToken());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int lcm;

        // 오름차순으로 정렬
        Arrays.sort(this.numArray);

        // 최소공배수 연산에서 제외 시킬 i, j 번째 인덱스
        for(int i = 0; i < COUNT; i++) {
            for(int j = i + 1; j < COUNT; j++) {
                ArrayList<Integer> indexs = new ArrayList<>(Arrays.asList(new Integer[]{0,1,2,3,4}));

                // 2개 인덱스 제외
                indexs.remove(j);
                indexs.remove(i);

                // 남은 3개 인덱스에 해당되는 수의 최소공배수를 구함
                lcm = this.getLcm(this.numArray[indexs.get(0)], this.numArray[indexs.get(1)]);
                lcm = this.getLcm(lcm, this.numArray[indexs.get(2)]);

                this.result = Integer.min(lcm, this.result);
            }
        }
    }

    public int getGcd(int a, int b) {
        int tmp, small, big;

        if (a > b) {
            big = a;
            small = b;
        } else {
            big = b;
            small = a;
        }

        while (small != 0) {
            tmp = big % small;
            big = small;
            small = tmp;
        }

        return big;
    }

    public int getLcm(int a, int b) {
        return (a * b) / this.getGcd(a, b);
    }
}
```  

---
## Reference
[1145-적어도 대부분의 배수](https://www.acmicpc.net/problem/1145)  
