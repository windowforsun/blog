--- 
layout: single
classes: wide
title: "[풀이] 백준 1708 볼록 껍질"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '평면상의 좌표들로 복록 껍질을 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Geometry
  - Convex Hull
  - Graham's Scan
---  

# 문제
- 아래 그림과 같은 두 꼭짓점을 연결하는 선분이 항상 다격형 내부에 존재하는 다각형을 볼록 다각형이라고 한다.

![볼록 다각형]({{site.baseurl}}/img/algorithm/concept-convexhull-1.png)

- 볼록 다각형은 다각형의 모든 내각이 180도 이하일 때 성립된다.
- 2차원 평면에 N개의 점이 주어졌을 때, 이들 중 몇개의 점을 골라 볼록 다각형을 만드는데, 나머지 모든 점을 내부에 포함하도록 할 수 있다.
- 위와 같은 것을 볼록 껍질(Convex Hull) 이라고 한다.
- 점이 주어졌을 때 볼록 껍지리을 이루는 점의 개수를 구하라.

## 입력
첫째 줄에 점의 개수 N(3 ≤ N ≤ 100,000)이 주어진다. 둘째 줄부터 N개의 줄에 걸쳐 각 점의 x좌표와 y좌표가 빈 칸을 사이에 두고 주어진다. 주어지는 모든 점의 좌표는 다르다고 가정해도 좋다. x좌표와 y좌표의 범위는 절댓값 40,000을 넘지 않는다. 입력으로 주어지는 다각형의 모든 점이 일직선을 이루는 경우는 없다.

## 출력
첫째 줄에 볼록 껍질을 이루는 점의 개수를 출력한다.

볼록 껍질의 변에 점이 여러 개 있는 경우에는 가장 양 끝 점만 개수에 포함한다.

## 예제 입력

```
8
1 1
1 2
1 3
2 1
2 2
2 3
3 1
3 2
```  

## 예제 출력

```
5
```  

## 풀이
- 문제을 해결하기 위해서 아래의 알고리즘을 활용한다.
	- [CCW]({{site.baseurl}}{% link _posts/2019-05-27-algorithm-concept-geometry.md %}) : 한 선분에서 다른 점까지의 각도가 180도 이하인지 를 판별한다.
	- [Graham's Scan]({{site.baseurl}}{% link _posts/2019-06-01-algorithm-concept-convexhull.md %}) : ccw 를 활용해서 볼록 껍질을 구한다.

```java
public class Main {
    private int result;
    private int count;
    private Vertex[] vertexArray;

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

        try{
            this.count = Integer.parseInt(reader.readLine());
            this.vertexArray = new Vertex[this.count];

            StringTokenizer token;

            for(int i = 0; i < this.count; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                this.vertexArray[i] = new Vertex(Integer.parseInt(token.nextToken()), Integer.parseInt(token.nextToken()));
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        LinkedList<Vertex> stack = new LinkedList<>();
        Vertex start, end, next = null;
        int nextIndex;

        // 좌표로 정렬
        Arrays.sort(this.vertexArray);

        for(int i = 1; i < this.count; i++) {
            this.vertexArray[i].setRelativeValue(this.vertexArray[0]);
        }

        // 기준점을 기준으로 한 기울기로 정렬
        Arrays.sort(this.vertexArray);

        stack.push(this.vertexArray[0]);
        stack.push(this.vertexArray[1]);
        nextIndex = 2;

        while(nextIndex < this.count) {
            // 스택에 2개이상의 점이 있거나 start-end 선과 next 점이 반시계(좌회전) 일때까지 반복한다.
            while(stack.size() >= 2) {
                end = stack.pop();
                start = stack.peek();
                next = this.vertexArray[nextIndex];

                // start-end 선과 next 점이 같은 직선에 있지 않거나, 좌회전 일 경우
                if(this.ccw(start, end, next) > 0) {
                    stack.push(end);
                    break;
                }
            }

            nextIndex++;
            stack.push(next);
        }

        this.result = stack.size();
    }

    public double ccw(Vertex p, Vertex v1, Vertex v2) {
        return this.ccw(minusVector(v1, p), minusVector(v2, p));
    }

    public static double ccw(Vertex v1, Vertex v2) {
        return (1d * v1.x * v2.y) - (1d * v1.y * v2.x);
    }

    public Vertex minusVector(Vertex v1, Vertex v2) {
        return new Vertex(v1.x - v2.x, v1.y - v2.y);
    }


    class Vertex implements Comparable<Vertex>{
        public int x;
        public int y;
        // 기준점으로 부터 상대 위치
        public int p;
        public int q;

        public Vertex(int x, int y) {
            this.x = x;
            this.y = y;
            this.p = 1;
            this.q = 0;
        }

        public void setRelativeValue(Vertex vertex) {
            this.p = this.x - vertex.x;
            this.q = this.y - vertex.y;
        }

        @Override
        public int compareTo(Vertex o) {
            int result = 0;
            long angleValue1 = 1l * this.q * o.p;
            long angleValue2 = 1l * this.p * o.q;

            if(angleValue1 != angleValue2) {
                if(angleValue1 < angleValue2) {
                    result = -1;
                } else {
                    result = 1;
                }
            } else {
                if(this.y != o.y) {
                    result = this.y - o.y;
                } else {
                    result = this.x - o.x;
                }
            }

            return result;
        }
    }
}
```  

---
## Reference
[1708-볼록 껍질](https://www.acmicpc.net/problem/1708)  
