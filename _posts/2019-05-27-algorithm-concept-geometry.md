--- 
layout: single
classes: wide
title: "[Algorithm 개념] 기하 알고리즘(Geometry Algorithm)"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '기하 알고리즘을 통해 선분들의 방향, 교차점, 내부, 외부를 판별해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Geometry
  - Vector
  - Polygon
  - CCW
use_math : true
---  

## 기하 알고리즘이란
- 기하는 공간에 있는 점, 선, 면에 대해 치수, 모양, 위치 등에 대한 수학의 한 분야이다.
- 기하 알고리즘은 이런 수학의 분야를 컴퓨터에서 처리할 수 있도록 원리를 알고리즘화 시킨 것이다.


## 기하 알고리즘의 종류
- 두 선분이 교차하는지 확인
- 여러 개의 점들을 꼭지점으로 하는 다각형 만들기
- 주어진 점이 다각형 내부에 존재하는지 확인하기
- 주어진 점들을 둘러싸는 가장 작은 다각형 찾기
- 주어진 점들 중에서 최단 경로 찾기


## 두 벡터의 상대적인 방향 판별하기

- 두 벡터의 상대적인 방향 판별이란 아래와 같다.
	- a는 b로 부터 반시계 방향에 위치하고 있다.
	- a는 c로 부터 시계 방향에 위치하고 있다.

![두 선번 방향1]({{site.baseurl}}/img/algorithm/concept-geometry-1.jpg)

- 이렇게 벡터들에 대해서 상대적인으로 시계, 반시계 방향인지 몇도인지 파악하는 것이 방향 판별이다.

### 벡터의 외적
- 3차원에서 벡터의 외적
	- $\vec{a} = (a_x, a_y, a_z)$, $\vec{b} =(b_x, b_y, b_z)$ 두 벡터가 있을 때
	- 두 벡터의 외적은 $a \times b = ({a_y}{b_z}-{a_z}{b_y}, {a_x}{b_z}-{a_z}{b_x}, {a_x}{b_y}-{a_y}{b_x})$ 로 나타낼 수 있다.
- 2차원에서 벡터의 외적
	- $\vec{a} = (a_x, a_y, 0)$, $\vec{b} =(b_x, b_y, 0)$ 두 벡터가 있을 때
	- 두 벡터의 외적은 $a \times b = (0, 0, {a_x}{b_y}-{a_y}{b_x})$ 로 나타낼 수 있다.
- 위의 결과(${a_x}{b_y}-{a_y}{b_x}$)의 값에 따라 2차원 벡터의 방향이 결정된다.
	- 양수이면 $\vec{b}$는 $\vec{a}$ 로 부터 반시계 방향에 있다.
	- 음수이면 $\vec{b}$는 $\vec{a}$ 로 부터 시계 방향에 있다.
- 이를 코드로 풀면 아래와 같다.

```java
public class Main {
    public static void main(String[] args) {
        Vector a = new Vector(0, 2);
        Vector b = new Vector(2, 0);
        Vector c = new Vector(-2, -2);
        Vector d = new Vector(0, 3);

        // 반시계
        System.out.println(ccw(a, b));
        // 시계
        System.out.println(ccw(a, c));
        // 평행
        System.out.println(ccw(a, d));
    }

    /**
     * 원점을 기준점으로 v1, v2의 상대적인 방향을 구한다.
     * 양수 이면 v2는 v1를 기준으로 반시계 방향에 위치한다.
     * 음수이면 v2는 v1를 기준으로 시계 방향에 위치한다.
     * v2가 v1과 평행이면 0
     * @param v1
     * @param v2
     * @return
     */
    public static double ccw(Vector v1, Vector v2) {
        return (v1.x * v2.y) - (v1.y * v2.x);
    }

}

class Vector{
    public double x;
    public double y;

    public Vector(double x, double y) {
        this.x = x;
        this.y = y;
    }
}
```  

- ccw 는 counter clock wise 로 두 벡터의 상대적인 방향의 결과값을 도출해 낸다.
	- $\vec{a} \times \vec{b} < 0$ 이므로 $\vec{b}$는 $\vec{a}$로 부터 반시계 방향에 있다.
	- $\vec{a} \times \vec{c} > 0$ 이므로 $\vec{c}$는 $\vec{a}$로 부터 시계 방향에 있다.
	- $\vec{a} \times \vec{d} = 0$ 이므로 $\vec{d}$는 $\vec{a}$로 부터 평행 한다.


## 두 직선의 교차점 구하기
- 먼저 무한한 선인 직선의 교차점을 구해본다.
- 이를 구하기 위해서는 위에서 구했던 벡터의 외적을 사용한다.
- 2개의 직선을 벡터로 표현하기 위해 4개의 벡터를 사용한다.
	- $\vec{a}$와 $\vec{b}$ 를 지나는 직선 x
	- $\vec{c}$와 $\vec{d}$ 를 지나는 직선 y
- 직선 x와 y를 벡터로 표현하면 아래와 같다.
	- $x = {\vec{a}+p\vec{b}}$
	- $y = {\vec{c}+p\vec{d}}$
- p 의 값을 구하는 공식은 아래와 같다.

$$
	p = {({\vec{c}-\vec{a}) \times \vec{d}} \over {\vec{b} \times \vec{d}}} (\vec{b} \times \vec{d} \neq 0)
$$
	
- 위 수식들을 이용해서 x, y 두 직선에서 교차점을 반환하는 코드는 아래와 같다.

```java
public class Main {
    public static void main(String[] args) {
        // 직선 x
        Vector a = new Vector(1, 0);
        Vector b = new Vector(10, 0);
        // 직선 y
        Vector c = new Vector(2, 2);
        Vector d = new Vector(2, -2);

        // (2, 0) 에서 교차한다.
        System.out.println(lineCross(a, b, c, d));

        // 직선 x
        a = new Vector(1, 0);
        b = new Vector(2, 0);

        // 직선 y
        c = new Vector(1, 1);
        d = new Vector(2, 1);

        // 두 직선은 교차하지 않는다.
        System.out.println(lineCross(a, b, c, d));
    }

    /**
     * 원점을 기준점으로 v1, v2의 상대적인 방향을 구한다.
     * 양수 이면 v2는 v1를 기준으로 반시계 방향에 위치한다.
     * 음수이면 v2는 v1를 기준으로 시계 방향에 위치한다.
     * v2가 v1과 평행이면 0
     * @param v1
     * @param v2
     * @return
     */
    public static double ccw(Vector v1, Vector v2) {
        return (v1.x * v2.y) - (v1.y * v2.x);
    }


    public static Vector minusVector(Vector v1, Vector v2) {
        return new Vector(v1.x - v2.x, v1.y - v2.y);
    }

    public static Vector plusVector(Vector v1, Vector v2) {
        return new Vector(v1.x + v2.x, v1.y + v2.y);
    }

    public static Vector multipleVector(Vector v1, double d) {
        return new Vector(v1.x * d, v1.y * d);
    }

    public static Vector lineCross(Vector a, Vector b, Vector c, Vector d) {
        Vector bMinusA = minusVector(b, a);
        Vector dMinusC = minusVector(d, c);
        Vector cMinusA = minusVector(c, a);
        double det = ccw(bMinusA, dMinusC);

        if (Math.abs(det) < 0.00001d) {
            return null;
        }

        double crossProject = ccw(cMinusA, dMinusC) / det;
        Vector crossVector = multipleVector(bMinusA, crossProject);

        crossVector = plusVector(a, crossVector);

        return crossVector;
    }
}

class Vector {
    public double x;
    public double y;

    public Vector(double x, double y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public String toString() {
        StringBuilder str = new StringBuilder();
        str.append("x : " + this.x);
        str.append(", ");
        str.append("y : " + this.y);

        return str.toString();
    }
}
```  

- 두 직선의 교차점을 구하는 `lineCross` 함수는 교차점이 있으면 교차점에 해당되는 벡터를 반환하고, 교차 점이 없다면 null 값을 반환한다.


## 두 선분의 교차점 구하기
- 한 직선위의 두 선분의 교차점 구하기에서는 아래와 같은 경우로 나눠 생각한다.

	![두 선번 방향2]({{site.baseurl}}/img/algorithm/concept-geometry-2.jpg)

	- 선분이 한점에서 겹친다.
	- 선분이 완전히 겹친다.
	- 선분이 다른 선분에 포함된다.
	- 선분이 서로 겹치지 않는다.
	
- 두 직선의 교차점을 구하는 방식과 위의 조건을 통해 코드로 구현하면 아래와 같다.

```java
public class Main {
    public static void main(String[] args) {
        // 선분 x
        Vector a = new Vector(1, 0);
        Vector b = new Vector(5, 0);
        // 선분 y
        Vector c = new Vector(5, 0);
        Vector d = new Vector(10, 0);

        // 한 점에서 겹친다.
        System.out.println(segmentCross(a, b, c, d));

        // 선분 x
        a = new Vector(1, 0);
        b = new Vector(10, 0);
        // 선분 y
        c = new Vector(1, 0);
        d = new Vector(10, 0);

        // 두 선분이 완전히 겹친다.
        System.out.println(segmentCross(a, b, c, d));

        // 선분 x
        a = new Vector(1, 0);
        b = new Vector(10, 0);
        // 선분 y
        c = new Vector(2, 0);
        d = new Vector(5, 0);

        // 한 선분이 다른 선분에 완전히 포함된다.
        System.out.println(segmentCross(a, b, c, d));

        // 선분 x
        a = new Vector(1, 0);
        b = new Vector(2, 0);
        // 선분 y
        c = new Vector(1, 1);
        d = new Vector(2, 1);

        // 선분이 서로 겹치지 않는다.
        System.out.println(segmentCross(a, b, c, d));
    }

    /**
     * 원점을 기준점으로 v1, v2의 상대적인 방향을 구한다.
     * 양수 이면 v2는 v1를 기준으로 반시계 방향에 위치한다.
     * 음수이면 v2는 v1를 기준으로 시계 방향에 위치한다.
     * v2가 v1과 평행이면 0
     * @param v1
     * @param v2
     * @return
     */
    public static double ccw(Vector v1, Vector v2) {
        return (v1.x * v2.y) - (v1.y * v2.x);
    }

    /**
     * p 를 기준점으로 v1, v2의 상대적인 방향을 구한다.
     * 양수 이면 v2는 v1를 기준으로 반시계 방향에 위치한다.
     * 음수 이면 v2는 v1를 기준으로 시계 방향에 위치한다.
     * v2가 v1과 평행이면 0
     * @param p
     * @param v1
     * @param v2
     * @return
     */
    public static double ccw(Vector p, Vector v1, Vector v2) {
        return ccw(minusVector(v1, p), minusVector(v2, p));
    }


    public static Vector minusVector(Vector v1, Vector v2) {
        return new Vector(v1.x - v2.x, v1.y - v2.y);
    }

    public static Vector plusVector(Vector v1, Vector v2) {
        return new Vector(v1.x + v2.x, v1.y + v2.y);
    }

    public static Vector multipleVector(Vector v1, double d) {
        return new Vector(v1.x * d, v1.y * d);
    }

    public static boolean isVectorEqual(Vector v1, Vector v2) {
        return v1.x == v2.x && v1.y == v2.y;
    }

    public static boolean isVectorGT(Vector small, Vector big) {
        return small.x < big.x && small.y < big.y;
    }

    /**
     * a-b 직선과 c-d 직선의 교차점이 있으면 교차점을 반환
     * 없으면 null 을 반환
     * @param a
     * @param b
     * @param c
     * @param d
     * @return
     */
    public static Vector lineCross(Vector a, Vector b, Vector c, Vector d) {
        Vector bMinusA = minusVector(b, a);
        Vector dMinusC = minusVector(d, c);
        Vector cMinusA = minusVector(c, a);
        double det = ccw(bMinusA, dMinusC);

        if(Math.abs(det) < 0.00001d) {
            return null;
        }

        double crossProject = ccw(cMinusA, dMinusC) / det;
        Vector crossVector = multipleVector(bMinusA, crossProject);

        crossVector = plusVector(a, crossVector);

        return crossVector;
    }

    public static void swapVector (Vector v1, Vector v2) {
        Vector tmp = new Vector(v1.x, v1.y);
        v1.x = v2.x;
        v1.y = v2.y;

        v2.x = tmp.x;
        v2.y = tmp.y;
    }

    /**
     * 두 선분이 만나지 않으면 null 반환
     * 교차점이 있다면 교차점 중 하나를 반환
     * @param start1
     * @param end1
     * @param start2
     * @param end2
     * @return
     */
    public static Vector isParallelLine(Vector start1, Vector end1, Vector start2, Vector end2) {
        if(!isVectorGT(start1, end1)) {
            swapVector(start1, end1);
        }

        if(!isVectorGT(start2, end2)) {
            swapVector(start2, end2);
        }

        if(ccw(start1, end1, start2) != 0
                || isVectorGT(end1, start2)
                || isVectorGT(end2, start1)) {
            return null;
        }

        Vector p;

        // 두 선분이 겹칠 경우
        if(isVectorGT(start1, start2)) {
            p = start2;
        } else {
            p = start1;
        }

        return p;
    }

    /**
     * 점 p 가 start, end 사이에 존재하는지 검사한다.
     * p, start, end 는 한 직선상에 있어야 한다.
     * @param p
     * @param start
     * @param end
     * @return
     */
    public static boolean isMiddleVector(Vector p, Vector start, Vector end) {
        if(!isVectorGT(start, end)) {
            swapVector(start, end);
        }
        return isVectorEqual(p, start)
                || isVectorEqual(p, end)
                || (isVectorGT(start, p) && isVectorGT(p, end));
    }

    /**
     * start1-end1 선분, start2-end2 선분을 동시에 지나는(교차점)을 반환한다.
     * 존재 하지 않을 경우 null 을 반환
     * 여러개일 경우 조건에 맞는 하나의 점을 반환한다.
     * @param start1
     * @param end1
     * @param start2
     * @param end2
     * @return
     */
    public static Vector segmentCross(Vector start1, Vector end1, Vector start2, Vector end2) {
        Vector lineCross = lineCross(start1, end1, start2, end2);

        if(lineCross == null) {
            return isParallelLine(start1, end1, start2, end2);
        }

        if(isMiddleVector(lineCross, start1, end1)
                && isMiddleVector(lineCross, start2, end2)) {
            return lineCross;
        } else {
            return null;
        }
    }
}

class Vector{
    public double x;
    public double y;

    public Vector(double x, double y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public String toString() {
        StringBuilder str = new StringBuilder();
        str.append("x : " + this.x);
        str.append(", ");
        str.append("y : " + this.y);

        return str.toString();
    }
}
```  

- 두 선분의 교차점을 구하는 `segmentCross` 는 교차점이 있을 경우 교차점을 반환하고 없을 경우 null 을 반환한다.


## 두 선분의 교차 여부 구하기
- 위에서 사용했던 `ccw` 함수 중 원점이 아닌 특정 벡터를 기준으로 방향을 구하는 함수를 사용해서 교차 여부를 구한다.
- 두 선분이 한점에서 교차한다면 아래와 같은 모양이다.
	- $\vec{A}$와 $\vec{B}$ 를 지나는 선분 x
	- $\vec{C}$와 $\vec{D}$ 를 지나는 선분 y

	![두 선번 방향3]({{site.baseurl}}/img/algorithm/concept-geometry-3.jpg)
	
	- $\vec{A}$ 를 기준으로 $\vec{C}$는 시계방향이고, $\vec{D}$는 반시계 방향이다.
	- 즉 선분의 한 벡터를 기준으로 교차하는지 구하는 선분의 두 벡터가 각각 시계방향, 반시계방향이면 이는 교차함을 알수 있다.
- 위 성질을 코드로 구현하면 아래와 같다.

```java
public class Main {
    public static void main(String[] args) {
        // 선분 x
        Vector a = new Vector(1, 0);
        Vector b = new Vector(5, 0);
        // 선분 y
        Vector c = new Vector(2, 2);
        Vector d = new Vector(2, -2);

        // 한 점에서 겹친다.
        System.out.println(isSegmentCross(a, b, c, d));

        // 선분 x
        a = new Vector(1, 0);
        b = new Vector(5, 0);
        // 선분 y
        c = new Vector(5, 0);
        d = new Vector(10, 0);

        // 한 점(끝점)에서 겹친다.
        System.out.println(isSegmentCross(a, b, c, d));

        // 선분 x
        a = new Vector(1, 0);
        b = new Vector(10, 0);
        // 선분 y
        c = new Vector(1, 0);
        d = new Vector(10, 0);

        // 두 선분이 완전히 겹친다.
        System.out.println(isSegmentCross(a, b, c, d));

        // 선분 x
        a = new Vector(1, 0);
        b = new Vector(10, 0);
        // 선분 y
        c = new Vector(2, 0);
        d = new Vector(5, 0);

        // 한 선분이 다른 선분에 완전히 포함된다.
        System.out.println(isSegmentCross(a, b, c, d));

        // 선분 x
        a = new Vector(1, 0);
        b = new Vector(2, 0);
        // 선분 y
        c = new Vector(1, 1);
        d = new Vector(2, 1);

        // 선분이 서로 겹치지 않는다.
        System.out.println(isSegmentCross(a, b, c, d));
    }

    /**
     * 원점을 기준점으로 v1, v2의 상대적인 방향을 구한다.
     * 양수 이면 v2는 v1를 기준으로 반시계 방향에 위치한다.
     * 음수이면 v2는 v1를 기준으로 시계 방향에 위치한다.
     * v2가 v1과 평행이면 0
     * @param v1
     * @param v2
     * @return
     */
    public static double ccw(Vector v1, Vector v2) {
        return (v1.x * v2.y) - (v1.y * v2.x);
    }

    /**
     * p 를 기준점으로 v1, v2의 상대적인 방향을 구한다.
     * 양수 이면 v2는 v1를 기준으로 반시계 방향에 위치한다.
     * 음수 이면 v2는 v1를 기준으로 시계 방향에 위치한다.
     * v2가 v1과 평행이면 0
     * @param p
     * @param v1
     * @param v2
     * @return
     */
    public static double ccw(Vector p, Vector v1, Vector v2) {
        return ccw(minusVector(v1, p), minusVector(v2, p));
    }


    public static Vector minusVector(Vector v1, Vector v2) {
        return new Vector(v1.x - v2.x, v1.y - v2.y);
    }

    public static Vector plusVector(Vector v1, Vector v2) {
        return new Vector(v1.x + v2.x, v1.y + v2.y);
    }

    public static Vector multipleVector(Vector v1, double d) {
        return new Vector(v1.x * d, v1.y * d);
    }

    public static boolean isVectorEqual(Vector v1, Vector v2) {
        return v1.x == v2.x && v1.y == v2.y;
    }

    public static boolean isVectorGT(Vector small, Vector big) {
        return small.x < big.x && small.y < big.y;
    }

    public static void swapVector (Vector v1, Vector v2) {
        Vector tmp = new Vector(v1.x, v1.y);
        v1.x = v2.x;
        v1.y = v2.y;

        v2.x = tmp.x;
        v2.y = tmp.y;
    }

    /**
     * 두 선분이 한점에서 교차하는지 여부를 판별한다.
     * @param start1
     * @param end1
     * @param start2
     * @param end2
     * @return
     */
    public static boolean isSegmentCross(Vector start1, Vector end1, Vector start2, Vector end2) {
        boolean result = false;
        double start1End1 = ccw(start1, end1, start2) * ccw(start1, end1, end2);
        double start2End2 = ccw(start2, end2, start1) * ccw(start2, end2, end1);

        // 두 선분이 한 직선 위에 있거나, 끝점이 겹침
        if(start1End1 == 0 && start2End2 == 0) {
            if(!isVectorGT(start1, end1)) {
                swapVector(start1, end1);
            }

            if(!isVectorGT(start2, end2)) {
                swapVector(start2, end2);
            }

            result = !(isVectorGT(end1, start2) || isVectorGT(end2, start1));
        } else {
            result = start1End1 <= 0 && start2End2 <= 0;
        }

        return result;
    }
}

class Vector{
    public double x;
    public double y;

    public Vector(double x, double y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public String toString() {
        StringBuilder str = new StringBuilder();
        str.append("x : " + this.x);
        str.append(", ");
        str.append("y : " + this.y);

        return str.toString();
    }
}
```  


## 다각형에서 내부 외부 판별하기

![두 선번 방향3]({{site.baseurl}}/img/algorithm/concept-geometry-4.jpg)

- 위 그림에서 K의 점과 L점이 주어진 다각형에 내부에 위치하는지 외부에 위치하는지 판별해보자.
- 각 두 점에서 x축과 평행하는 직선을 그어 준다.

![두 선번 방향3]({{site.baseurl}}/img/algorithm/concept-geometry-5.jpg)

- 다각형의 외부에 있는 L은 오른쪽 방향으로 보았을 때 다각형의 면과 교차되는 점의 개수가 짝수개이다.
- 다각형의 내부에 있는 K는 오른쪽 방향으로 보았을 때 다각형의 면과 교차되는 점의 개수가 홀수개이다.
- 판별하고자 하는 점에 아래와 같은 조건을 통해 판별을 할수 있다.
	- 점의 y 좌표는 다각형의 꼭지점의 y 좌표 사이에 있어야 한다.
	- 점의 x 좌표는 다각형의 꼭지점의 x 좌표 보다 작아야 한다. (점이 다각형의 꼭지점보다 왼쪽에 위치해야 한다.)
	- 위를 만족 시키는 면의 개수가 홀수개면 내부, 짝수개이면 외부에 있다.
- 위 조건을 코드로 구현하면 아래와 같다.

```java
public class Main {

    public static void main(String[] args) {
        // 사각형
        ArrayList<Vector> polygon = new ArrayList<Vector>();
        polygon.add(new Vector(0, 0));
        polygon.add(new Vector(4, 0));
        polygon.add(new Vector(4, 4));
        polygon.add(new Vector(0, 4));

        Vector innerP = new Vector(2, 3);
        Vector outerP = new Vector(-1, 2);

        System.out.println(isInner(innerP, polygon));
        System.out.println(isInner(outerP, polygon));

        // 다각형
        polygon = new ArrayList<Vector>();
        polygon.add(new Vector(0, 0));
        polygon.add(new Vector(5, 5));
        polygon.add(new Vector(-2, 0));
        polygon.add(new Vector(-2, 4));
        polygon.add(new Vector(-5, 4));
        polygon.add(new Vector(-5, 0));

        innerP = new Vector(-4, 3);
        outerP = new Vector(-6, 3);

        System.out.println(isInner(innerP, polygon));
        System.out.println(isInner(outerP, polygon));
    }

    /**
     * 원점을 기준점으로 v1, v2의 상대적인 방향을 구한다.
     * 양수 이면 v2는 v1를 기준으로 반시계 방향에 위치한다.
     * 음수이면 v2는 v1를 기준으로 시계 방향에 위치한다.
     * v2가 v1과 평행이면 0
     * @param v1
     * @param v2
     * @return
     */
    public static double ccw(Vector v1, Vector v2) {
        return (v1.x * v2.y) - (v1.y * v2.x);
    }


    public static Vector minusVector(Vector v1, Vector v2) {
        return new Vector(v1.x - v2.x, v1.y - v2.y);
    }

    public static Vector plusVector(Vector v1, Vector v2) {
        return new Vector(v1.x + v2.x, v1.y + v2.y);
    }

    public static Vector multipleVector(Vector v1, double d) {
        return new Vector(v1.x * d, v1.y * d);
    }

    public static Vector lineCross(Vector a, Vector b, Vector c, Vector d) {
        Vector bMinusA = minusVector(b, a);
        Vector dMinusC = minusVector(d, c);
        Vector cMinusA = minusVector(c, a);
        double det = ccw(bMinusA, dMinusC);

        if (Math.abs(det) < 0.00001d) {
            return null;
        }

        double crossProject = ccw(cMinusA, dMinusC) / det;
        Vector crossVector = multipleVector(bMinusA, crossProject);

        crossVector = plusVector(a, crossVector);

        return crossVector;
    }

    public static boolean isInner(Vector p, ArrayList<Vector> polygon) {
        int crossCount = 0;
        int size = polygon.size();
        Vector start, end, pEnd = new Vector(1000000, p.y);

        for(int i = 0, j; i < size; i++) {
            j = (i + 1) % size;
            start = polygon.get(i);
            end = polygon.get(j);

            if(start.y > p.y != end.y > p.y) {
                Vector cross = lineCross(p, pEnd, start, end);

                if(p.x <cross.x) {
                    crossCount++;
                }
            }
        }

        return crossCount % 2 != 0;
    }
}

class Vector {
    public double x;
    public double y;

    public Vector(double x, double y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public String toString() {
        StringBuilder str = new StringBuilder();
        str.append("x : " + this.x);
        str.append(", ");
        str.append("y : " + this.y);

        return str.toString();
    }
}
```

---
## Reference
[기하 알고리즘이란](https://wondangcom.com/453)  
[[기하] 다각형의 내부 외부 판별](https://bowbowbow.tistory.com/24?category=159621)  

