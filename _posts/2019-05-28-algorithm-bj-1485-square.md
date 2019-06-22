--- 
layout: single
classes: wide
title: "[풀이] 백준 1485 정사각형"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 좌표들을 통해 정사각형을 만들수 있는지 판별하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Geometry
---  

# 문제
- 네 점이 주어졌을 때, 네 점을 통해 정사각형을 만들 수 있는지 구하라.

## 입력
첫째 줄에 테스트 케이스의 개수 T가 주어진다. 각 테스트 케이스는 네 줄로 이루어져 있으며, 점의 좌표가 한 줄에 하나씩 주어진다. 점의 좌표는 -100,000보다 크거나 같고, 100,000보다 작거나 같은 정수이다. 같은 점이 두 번 이상 주어지지 않는다.

## 출력
각 테스트 케이스마다 입력으로 주어진 네 점을 이용해서 정사각형을 만들 수 있으면 1을, 없으면 0을 출력한다.

## 예제 입력

```
2
1 1
1 2
2 1
2 2
2 2
3 3
4 4
5 5
```  

## 예제 출력

```
1
0
```  

## 풀이
- 네 점을 통해 사각형임을 판별 하는 방법은 아래와 같다.
	- 사각형의 모든 변의 길이는 같다.
	- 사각형의 내부 대각선의 길이는 모두 같다. (각도가 모두 같다.)
- 한 점에 대한 다른 세 점의 길이를 모두 구하고 이를 정렬하고 이를 비교하여 정사각형임을 판별한다.

```java
public class Main {
    private StringBuilder result;
    private int count;

    public Main() {
        this.input();
        this.output();
    }

    public static void main(String[] args){
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.count = Integer.parseInt(reader.readLine());
            this.result = new StringBuilder();

            int[][] vectorArray;
            StringTokenizer token;

            for(int i = 0; i < this.count; i++) {
                vectorArray = new int[4][2];

                for(int j = 0; j < 4; j++) {
                    token = new StringTokenizer(reader.readLine(), " ");
                    vectorArray[j][0] = Integer.parseInt(token.nextToken());
                    vectorArray[j][1] = Integer.parseInt(token.nextToken());

                }
                this.result.append(this.solution(vectorArray)).append("\n");
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result.toString());
    }

    public int solution(int[][] vectorArray) {
        double[][] distanceArray = new double[4][3];
        int index;

        for(int i = 0; i < 4; i++) {
            index = 0;
            for(int j = 0; j < 4; j++) {
                if(i != j) {
                    double distance = this.getDistance(vectorArray[i], vectorArray[j]);
                    distanceArray[i][index] = distance;
                    index++;
                }
            }
            Arrays.sort(distanceArray[i]);

            // 사각형에서 한점에서 다른 세점에 대한 거리를 정렬하게 되면 앞의 두 거리는(변) 같고, 앞의 두거리는 세번째 거리(대각선)보다 작다.
            if(distanceArray[i][0] != distanceArray[i][1] || distanceArray[i][1] >= distanceArray[i][2]) {
                return 0;
            }
        }

        // 네개의 점에 대해서 각각 다른 세점에 대한 거리를 구하고 이를 정렬했을 때, 모든 대각선의 거리와 모든 변의 거리는 같아야 한다.
        for(int i = 1; i < 4; i++) {
            for(int j = 0; j < 3; j++) {
                if(distanceArray[0][j] != distanceArray[i][j]) {
                    return 0;
                }
            }
        }

        return 1;
    }

    // 두 점 사이의 거리를 구한다.
    public double getDistance(int[] vector1, int[] vector2) {
        double x = vector1[0] - vector2[0];
        double y = vector1[1] - vector2[1];
        return Math.sqrt((x * x) + (y * y));
    }
}
```  

---
## Reference
[1485-정사각형](https://www.acmicpc.net/problem/1485)  
