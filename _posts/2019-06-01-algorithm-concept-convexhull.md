--- 
layout: single
classes: wide
title: "[Algorithm 개념] Convex Hull(볼록 껍질) Graham's Scan Algorithm"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '그라함 스캔 알고리즘으로 볼록 껍질을 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Convex Hull
  - Graham's Scan
  - CCW
use_math : true
---  

## Convex Hull(볼록 껍질) 이란
- 2차원 평면상에 여러개의 점이 있을 때 주어진 점들을 모두 포함하는 최소 크기의 다각형을 뜻한다.
- 볼록 껍질은 `Graham's Scan` 그라함 스캔 알고리즘을 통해 구할 수 있다.

![convex hull 1]({{site.baseurl}}/img/algorithm/concept-convexhull-1.png)

## Graham's Scan Algorithm 의 원리
- 알고리즘 구현은 스택과 정렬 그리고 [CCW]({{site.baseurl}}{% link _posts/2019-05-27-algorithm-concept-geometry.md %}) 를 통해 구현된다.
- 여러 점 중 y혹은 x가 가장 작은 좌표를 하나 선택한다.
- 위에서 선택한 점 제외한 점들을 반시계 방향으로 정렬한다.
- 정렬된 점들을 스택에 순서대로 넣는데 아래와 같은 과정이 있다.
	- 스택에 2개 이상의 점이 있으면, 그 두점을 이은 직선을 기준으로 왼쪽(반시계)에 추가할 점이 위치하는지 판별(ccw > 0)한다.
	- 왼쪽에 위치한다면 push 를 해준다.
	- 왼쪽에 위치하지 않는다면 pop 을 해준다.
- 최종적으로 스택에 남은 점들이 최소 크기의 다각형을 이루는 점이 된다.

## 예시
- 먼저 기준점(A)을 하나 잡는다.(빨간색)
	- 점들 중 y혹은 x가 가장 작은 점을 하나 선택한다.
	
![convex hull 2]({{site.baseurl}}/img/algorithm/concept-convexhull-2.png)

- 기준점에서 다른 모든 점들으르 반시계 방향으로 정렬한다.
	- 기준점으로 부터 각 점을 이었을때 각도가 가장 적은 순
	- `D-E-F-B-F-C-H-I` 순이 된다.
	
![convex hull 3]({{site.baseurl}}/img/algorithm/concept-convexhull-3.png)

- 기준점을 시작으로 각도가 가장 작은 순으로 스택에 push 해준다.
	- D점이 초록색인건 기분탓이다.
	- 점들 위에 오른쪽이 헤드가 되는 스택을 표현한다.

![convex hull 4]({{site.baseurl}}/img/algorithm/concept-convexhull-4.png)

- 스택에서 한개의 D점을 pop(end)하고 또 하나의 A점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 E점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다.
	- 이때 반시계(좌회전)이라면 껍질(외벽)이 될수 있다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 해준다.
	- 껍질이된 선분은 실선, 비교하는 선분은 점선으로 표시한다.

![convex hull 5]({{site.baseurl}}/img/algorithm/concept-convexhull-5.png)

start|end|next
---|---|---
A|D|E

- 스택에서 한개의 E점을 pop(end)하고 또 하나의 D점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 F점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 한다.
	
![convex hull 6]({{site.baseurl}}/img/algorithm/concept-convexhull-6.png)

start|end|next
---|---|---
D|E|F

- 스택에서 한개의 F점을 pop(end)하고 또 하나의 E점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 B점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 한다.
	
![convex hull 7]({{site.baseurl}}/img/algorithm/concept-convexhull-7.png)

start|end|next
---|---|---
E|F|B

- 스택에서 한개의 B점을 pop(end)하고 또 하나의 F점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 G점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이 아니기 때문에, pop 한 B점(end) 와 G점(next) 는 스택에 넣지 않는다.
	
![convex hull 8]({{site.baseurl}}/img/algorithm/concept-convexhull-8.png)

start|end|next
---|---|---
F|B|G

- 스택에서 한개의 F점을 pop(end)하고 또 하나의 E점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 G점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 한다.
	
![convex hull 9]({{site.baseurl}}/img/algorithm/concept-convexhull-9.png)

start|end|next
---|---|---
E|F|G

- 스택에서 한개의 G점을 pop(end)하고 또 하나의 F점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 C점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 한다.
	
![convex hull 10]({{site.baseurl}}/img/algorithm/concept-convexhull-10.png)

start|end|next
---|---|---
F|G|C

- 스택에서 한개의 C점을 pop(end)하고 또 하나의 G점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 H점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이 아니기 때문에, pop 한 C점(end) 와 H점(next) 는 스택에 넣지 않는다.
	
![convex hull 11]({{site.baseurl}}/img/algorithm/concept-convexhull-11.png)

start|end|next
---|---|---
G|C|H

- 스택에서 한개의 G점을 pop(end)하고 또 하나의 F점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 H점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 한다.
	
![convex hull 12]({{site.baseurl}}/img/algorithm/concept-convexhull-12.png)

start|end|next
---|---|---
F|G|H

- 스택에서 한개의 H점을 pop(end)하고 또 하나의 G점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 I점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 한다.
	
![convex hull 13]({{site.baseurl}}/img/algorithm/concept-convexhull-13.png)

start|end|next
---|---|---
G|H|I

- 스택에서 한개의 H점을 pop(end)하고 또 하나의 G점은 peek(start)해서 이를 선으로 잇고나서, 다음 각도가 작은 I점(next)과의 ccw 값이 반시계(>=0) 인지 확인한다. 
	- start-end, next 가 좌회전이기 때문에 end, next 를 스택에 push 한다.
	
![convex hull 13]({{site.baseurl}}/img/algorithm/concept-convexhull-13.png)

start|end|next
---|---|---
G|H|I

- 최종적으로 아래와 같은 Convex Hull 을 만들어 낼 수 있다.

![convex hull 14]({{site.baseurl}}/img/algorithm/concept-convexhull-14.png)

- 주의할 사항으로는 `ccw < 0` 일 경우에는 `start-end, next` 의 ccw 가 >= 0일 때까지 pop(end), peek(start), next 를 계속 반복해준다.
- `Graham Scan` 알고리즘은 2번의 정렬이 필요한데, 이로인해 시간복잡도는 정렬 알고리즘에 비례하는 O(nlogn) 이 된다.
- 소스코드

```java
public class Main {
    public Vertex[] vertexArray;

    public Main() {
        this.vertexArray = new Vertex[9];
        this.vertexArray[0] = new Vertex(0, 0); // A
        this.vertexArray[1] = new Vertex(3, 1); // D
        this.vertexArray[2] = new Vertex(4, 2); // E
        this.vertexArray[3] = new Vertex(3, 3); // F
        this.vertexArray[4] = new Vertex(2, 4); // G
        this.vertexArray[5] = new Vertex(-1, 3); // H
        this.vertexArray[6] = new Vertex(-2, 1); // I
        this.vertexArray[7] = new Vertex(0, 2); // C
        this.vertexArray[8] = new Vertex(2, 3); // B

        this.solution();
    }

    public static void main(String[] args) {
        Main main = new Main();
    }

    public void solution() {
        int size = this.vertexArray.length, nextIndex;
        Vertex start, end, next = null;
        LinkedList<Vertex> stack = new LinkedList<>();

        // 좌표 순으로 정렬한다. y값이 가장 작거나, x값이 가장 작은
        Arrays.sort(this.vertexArray);

        // 가장 작은 좌표가 기준점이 된다.(배열에서 0번째)
        // 기준점에 대한 각 좌표들의 상대 좌표를 구한다.
        for(int i = 1; i < size; i++) {
            this.vertexArray[i].p = this.vertexArray[i].x - this.vertexArray[0].x;
            this.vertexArray[i].q = this.vertexArray[i].y - this.vertexArray[0].y;
        }

        // 기준점 상대 좌표를 통한 각도가 작은 순으로 정렬
        Arrays.sort(this.vertexArray);

        stack.push(this.vertexArray[0]);
        stack.push(this.vertexArray[1]);
        nextIndex = 2;

        while(nextIndex < size) {

            // 스택에 2개이상의 점이 있거나 start-end 선과 next 점이 반시계(좌회전) 일때까지 반복한다.
            while(stack.size() >= 2) {
                end = stack.pop();
                start = stack.peek();
                next = this.vertexArray[nextIndex];

                // start-end 선과 next 점이 반시계(좌회전) 일 경우
                // >= 일경우에는 같은 직선상에서 가운데에 있는 점도 껌질로 포함하고
                // > 일경우에는 같은 직선상에서 가운데에 있는 점은 껍질로 포함하지 않는다.
                if(this.ccw(start, end, next) >= 0) {
                    stack.push(end);
                    break;
                }
            }

            // 각도로 정렬된 순서에서 다음 점으로 이동
            nextIndex++;
            stack.push(next);
        }

        System.out.println(Arrays.toString(stack.toArray()));
    }

    public double ccw(Vertex p, Vertex v1, Vertex v2) {
        return this.ccw(minusVector(v1, p), minusVector(v2, p));
    }

    public static double ccw(Vertex v1, Vertex v2) {
        return (v1.x * v2.y) - (v1.y * v2.x);
    }
    
    public Vertex minusVector(Vertex v1, Vertex v2) {
        return new Vertex(v1.x - v2.x, v1.y - v2.y);
    }

    class Vertex implements Comparable<Vertex> {
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

        @Override
        public int compareTo(Vertex o) {
            long angleValue1 = this.q * o.p;
            long angleValue2 = this.p * o.q;
            int result = 0;

            if(angleValue1 != angleValue2) {
                // 각도로 정렬할 때
                if(angleValue1 < angleValue2) {
                    result = -1;
                } else {
                    result = 1;
                }
            } else {
                // 좌표로 정렬할 때
                if(this.y != o.y) {
                    result = this.y - o.y;
                } else {
                    result = this.x - o.x;
                }
            }

            return result;
        }

        @Override
        public String toString() {
            return "x:" + this.x + ", y:" + this.y + ", p:" + this.p + ", p:" + this.q + "\n";
        }
    }
}
```  

---
## Reference
[컨벡스 헐 알고리즘(Convex Hull Algorithm)](https://www.crocus.co.kr/1288)  
[볼록껍질](https://hellogaon.tistory.com/39)  
[볼록 껍질(Convex Hull) (수정: 2019-01-22)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220857597424&parentCategoryNo=&categoryNo=299&viewDate=&isShowPopularPosts=true&from=search)  

