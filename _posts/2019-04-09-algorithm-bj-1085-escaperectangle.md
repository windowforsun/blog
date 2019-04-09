--- 
layout: single
classes: wide
title: "[풀이] 백준 1085 직사각형에서 탈출"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '직사각형 내의 한점이 주어 질때 가장 가까운 면과의 거리를 구하라'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Math
---  

# 문제
- (0,0) 왼쪽 아래 ~ (w,h) 오른쪽 위로 구성된 직사각형이 있다.
- (x,y) 점이 주어 질때 직사각형의 경계선까지 가는 최소 거리를 구하라.

## 입력
첫째 줄에 x y w h가 주어진다. w와 h는 1,000보다 작거나 같은 자연수이고, x는 1보다 크거나 같고, w-1보다 작거나 같은 자연수이고, y는 1보다 크거나 같고, h-1보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 문제의 정답을 출력한다.

## 예제 입력

```
6 2 10 3
```  

## 예제 출력

```
1
```  

## 풀이
- (0,0) ~ (10,3) 의 직사각형에서 현재 위치가 (6,2) 일경우 가장 가까운 경계선과 그 점의 좌표는 아래와 같다.

	![풀이]({{site.baseurl}}/img/algorithm/bj-1085-1.png)
- 현재 위치 좌표에서 4개의 경계선과 직선으로 만나는 4개의 좌표 중 가장 가까운 거리를 구하면 된다.

```java
public class Main {
    public TestHelper testHelper = new TestHelper();

    class TestHelper {
        private ByteArrayOutputStream out;

        public TestHelper() {
            this.out = new ByteArrayOutputStream();

            System.setOut(new PrintStream(this.out));
        }

        public String getOutput() {
            return this.out.toString().trim();
        }
    }
    // 출력 결과 저장
    private int result;
    // 현재 위치한 x 축 좌표
    private int currentX;
    // 현재 위치한 y 축 좌표
    private int currentY;
    // 총 너비
    private int w;
    // 총 높이
    private int h;

    public Main() {
        this.result = Integer.MAX_VALUE;
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
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            this.currentX = Integer.parseInt(token.nextToken());
            this.currentY = Integer.parseInt(token.nextToken());
            this.w = Integer.parseInt(token.nextToken());
            this.h = Integer.parseInt(token.nextToken());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // x, y 좌표에서 직사각형과 직선으로 만나는 4가지 점의 좌표
        int[][] sideWhArray = new int[][]{
                {this.currentX, 0}, {0, this.currentY},
                {this.currentX, this.h}, {this.w, this.currentY}
        };

        int size = sideWhArray.length, dist, tmpW, tmpH;

        // (x1, y1), (x2, y2) 두 점사이의 거리 dist = sqrt((x1 - x2)^2 + (y1 - y2)^2)
        for(int i = 0; i < size; i++) {
            tmpW = this.currentX - sideWhArray[i][0];
            tmpH = this.currentY - sideWhArray[i][1];

            tmpW *= tmpW;
            tmpH *= tmpH;

            dist = (int)Math.sqrt(tmpW + tmpH);
            if(this.result > dist) {
                this.result = dist;
            }
        }
    }
}
```  

---
## Reference
[1085-직사각형에서 탈출](https://www.acmicpc.net/problem/1085)  
