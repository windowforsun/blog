--- 
layout: single
classes: wide
title: "[풀이] 백준 1021 회전하는 큐"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: 'Queue 를 사용해서 최소 회전 동작을 통해 순서대로 원소를 빼내어 보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Simulation
  - Queue
---  

# 문제
- DeQueue(양방향큐) 가 하나 있다.
- 큐에서 몇개의 원소를 뽑는 동작을 수행한다.
- 동작 3가지가 있다.
	1. 원소를 뽑아낸다.
		- a1, ... , ak 에서 a2, ... , ak 와 같이 된다.
	1. 왼쪽으로 한 칸 이동한다.
		- a1, ... , ak 에서 a2, ... , ak, a1 이 된다.
	1. 오른쪽으로 한 칸 이동한다.
		- a1, ... , ak 에서 ak, a1, ... , ak-1 이된다.
- N 개의 원소에서 M 개의 원소를 순서대로 뽑아 낼때 2, 3번 연산의 최솟값을 출력하라.

## 입력
첫째 줄에 큐의 크기 N과 뽑아내려고 하는 수의 개수 M이 주어진다. 
N은 50보다 작거나 같은 자연수이고, M은 N보다 작거나 같은 자연수이다. 
둘째 줄에는 지민이가 뽑아내려고 하는 수의 위치가 순서대로 주어진다. 
위치는 1보다 크거나 같고, N보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 문제의 정답을 출력한다.

## 예제 입력

```
10 3
1 2 3
```  

## 예제 출력

```
0
```  

## 예제 입력

```
10 3
2 9 5
```  

## 예제 출력

```
8
```  

## 예제 입력

```
32 6
27 16 30 11 6 23
```  

## 예제 출력

```
59
```  

## 예제 입력

```
10 10
1 6 3 2 7 9 8 4 10 5
```  

## 예제 출력

```
14
```  

# 풀이
- 빼내어야 하는 원소의 위치가 주어졌을 때 오른쪽, 왼쪽으로 이동할지는 Queue 의 Head 를 기준으로 결정한다.
	- 여기서 Queue Head 는 특정 상태의 Queue 에서 맨 앞에 위치한 원소이다.
- Head 를 기준으로 오른쪽 방향에서 빼내야 하는 위치의 위치 차이를 구한다.
	- Queue 에 1, 2, 3, 4, 5 가 있을 경우 Head 는 1 이되고, 찾는 위치가 4 가 될경우 오른쪽 위치 차이는 3 이 된다.
- Head 를 기준으로 오른쪽 방향에서 빼내야 하는 위치의 위치 차이를 구한다.
	- Queue 에 1, 2, 3, 4, 5 가 있을 때 Head 는 1 이 되고, 찾는 위치가 4 가 일 경우 왼쪽 위치 차이는 2 가 된다.
- 오른쪽, 왼쪽 위치 차이에서 적은 쪽의 위치 차이 값만큼 움직이면 된다.
	- 오른쪽 위치 차이값이 더 적을 경우 방향은 왼쪽으로 움직인다.
	- 왼쪽 위치 차이값이 더 적을 경우 방향은 오른쪽으로 움직인다.
- 위치 차이 값만큼 움직이면서 결과 카운트를 카운트해준다.
- 이동이 끝났으면 첫번째(빼내여야 하는 위치) 원소를 큐에서 삭제해 준다.

```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 큐의 크기
    private int queueSize;
    // 뽑아 내는 숫자의 개수
    private int popCount;
    // 사용할 큐
    private LinkedList<Integer> queue;
    // 뽑아내야 하는 인덱스 배열
    private ArrayList<Integer> popIndexArray;

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
            this.queueSize = Integer.parseInt(token.nextToken());
            this.popCount = Integer.parseInt(token.nextToken());

            this.queue = new LinkedList<Integer>();
            this.popIndexArray = new ArrayList<>(this.popCount);

            for (int i = 1; i <= this.queueSize; i++) {
                this.queue.addLast(i);
            }

            token = new StringTokenizer(reader.readLine(), " ");

            for (int i = 0; i < this.popCount; i++) {
                popIndexArray.add(Integer.parseInt(token.nextToken()));
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.print(this.result);
    }

    public void solution() {
        int currentQueueSize, rightGap, leftGap, moveCount, moveDir, tmp, resultCount = 0;

        // 움직이는 방향 결정
        final int RIGHT = 1;
        final int LEFT = 2;

        // 뽑아내는 위치의 개수만큼 loop
        for (int i = 0; i < this.popCount; i++) {
            // 현재 큐의 크기
            currentQueueSize = this.queue.size();

            // 큐 head 에서부터 오른쪽으로 찾고자 하는 인덱스의 위치 차이
            rightGap = this.queue.indexOf(this.popIndexArray.get(i));
            // 큐 head 에서부터 왼족으로 찾고자 하는 인덱스의 위치 차이
            leftGap = currentQueueSize - rightGap;

            // 둘중 작은 쪽을 선택
            if (rightGap < leftGap) {
                moveCount = rightGap;
                // 오른쪽 차이가 작을 경우 왼쪽으로 큐 이동
                moveDir = LEFT;
            } else {
                moveCount = leftGap;
                // 왼쪽 차이가 작을 경우 오른쪽으로 큐 이동
                moveDir = RIGHT;
            }

            // 움직이는(2,3 번 동작) 카운트 만큼 반복
            for (int j = 0; j < moveCount; j++) {
                // 각 방향에 따라 큐 원소 이동
                if (moveDir == LEFT) {
                    tmp = this.queue.removeFirst();
                    this.queue.addLast(tmp);
                } else {
                    tmp = this.queue.removeLast();
                    this.queue.addFirst(tmp);
                }

                // 결과 카운트 값 증가
                resultCount++;
            }

            // 첫번째(찾고자 하는 위치) 원소 삭제
            this.queue.removeFirst();
        }

        this.result.append(resultCount).append("\n");
    }
}
```  


---
## Reference
[1021-회전하는 큐](https://www.acmicpc.net/problem/1021)  
