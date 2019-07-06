--- 
layout: single
classes: wide
title: "[풀이] 백준 7578 공장"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '교차 간선의 개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
  - Fenwick Tree
---  

# 문제
- 두 개의 열에 번호의 쌍을 이루고 있는 기계가 배치되어 있다.
	
	.|0|1|2|3
	---|---|---|---|---
	A|123|111|222|333
	B|111|222|333|123

- 번호는 각 기계를 구분짓는 식별 번호이기 때문에 하나의 열에 식별번호는 고유하다.
- A열의 식별번호는 B열에 같은 식별번호를 가진 기계와 케이블로 연결된다.
- A열과 B열에 같은 식별번호 끼리 연결된 케이블로 인해 케이블이 엉켜있다.
- 배치된 기계를 연결하기 위한 교차하는 케이블 쌍의 개수를 구하라.

## 입력
입력은 세 줄로 이루어져 있다. 첫 줄에는 정수 N이 주어지며, 두 번째 줄에는 A열에 위치한 N개 기계의 서로 다른 식별번호가 순서대로 공백문자로 구분되어 주어진다. 세 번째 줄에는 B열에 위치한 N개의 기계의 식별번호가 순서대로 공백문자로 구분되어 주어진다.

단, 1 ≤ N ≤ 500,000이며, 기계의 식별번호는 모두 0 이상 1,000,000 이하의 정수로 주어진다

## 출력
여러분은 읽어 들인 2N개의 기계의 배치로부터 서로 교차하는 케이블 쌍의 개수를 정수 형태로 한 줄에 출력해야 한다.

## 예제 입력

```
5
132 392 311 351 231
392 351 132 311 231
```  

## 예제 출력

```
3
```  

## 풀이
- 교차 간선을 판별하는 방법은 아래와 같다.
	- A 그룹을 순차적으로 탐색한다.
	- 순차 탐색하며 연결된 B 그룹의 인덱스에 연결 정보를 설정한다.
	- A 그룹과 연결된 B 그룹에서 왼쪽(인덱스가 큰)쪽으로 봤을 때 이미 연결되어 있는(연결 정보가 설정된) 간선의 수가 교차 간선의 수이다.
	- 이 때 이미 설정된 간선의 수는 인덱스~배열의 크기의 구간 합이다.
	- 구간 합을 구하기 위해 [펜윅 트리]({{site.baseurl}}{% link _posts/2019-06-20-algorithm-concept-fenwicktreebinaryindextree.md %}) 를 사용한다.
	
- 아래와 같은 간선에 대한 정보가 있다.

	![그림1]({{site.baseurl}}/img/algorithm/bj-7578-1.png)

- 펜윅 트리(F) 배열 정보에는 A 그룹을 순차 탐색하며 연결된 B 그룹의 인덱스에 간선 연결 정보를 설정한다.

	index|1|2|3|4
	---|---|---|---|---
	info|0|0|0|0
	
- A 그룹 111 식별 번호와 연결된 B 그룹노드의 인덱스(i)보다 큰 인덱스에서 연결된 간선의 수를 구하고, 간선의 연결 정보를 F 배열에 설정한다.
	- 교차간선 수 = 0 (F 배열에서 4~4 구간의 합)
	- result = 0

	![그림2]({{site.baseurl}}/img/algorithm/bj-7578-2.png)
	
	index|1|2|3|4
	---|---|---|---|---
	info|0|0|0|1

- A 그룹 222 식별 번호와 연결된 B 그룹노드의 인덱스(i)보다 큰 인덱스에서 연결된 간선의 수를 구하고, 간선의 연결 정보를 F 배열에 설정한다.
	- 교차간선 수 = 1 (F 배열에서 2~4 구간의 합)
	- 교차간선 식별번호 = 111
	- result = 1

	![그림3]({{site.baseurl}}/img/algorithm/bj-7578-3.png)
	
	index|1|2|3|4
	---|---|---|---|---
	info|0|1|0|1
	
- A 그룹 333 식별 번호와 연결된 B 그룹노드의 인덱스(i)보다 큰 인덱스에서 연결된 간선의 수를 구하고, 간선의 연결 정보를 F 배열에 설정한다.
	- 교차간선 수 = 1 (F 배열에서 3~4 구간의 합)
	- 교차간선 식별번호 = 111
	- result = 2

	![그림4]({{site.baseurl}}/img/algorithm/bj-7578-4.png)
	
	index|1|2|3|4
	---|---|---|---|---
	info|0|1|1|1

- A 그룹 444 식별 번호와 연결된 B 그룹노드의 인덱스(i)보다 큰 인덱스에서 연결된 간선의 수를 구하고, 간선의 연결 정보를 F 배열에 설정한다.
	- 교차간선 수 = 3 (F 배열에서 1~4 구간의 합)
	- 교차간선 식별번호 = 111, 222, 333
	- result = 5

	![그림5]({{site.baseurl}}/img/algorithm/bj-7578-5.png)
	
	index|1|2|3|4
	---|---|---|---|---
	info|1|1|1|1	
	
```java
public class Main {
    private long result;
    private int dataCount;
    private int[] fenwickTree;
    private int[] aDataArray;
    private int[] bDataIndexArray;
    private int treeSize;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            this.dataCount = Integer.parseInt(reader.readLine());
            this.treeSize = this.dataCount + 1;
            this.fenwickTree = new int[this.treeSize];
            this.aDataArray = new int[this.dataCount + 1];
            this.bDataIndexArray = new int[this.MAX_NUM + 1];

            StringTokenizer aToken = new StringTokenizer(reader.readLine(), " ");
            StringTokenizer bToken = new StringTokenizer(reader.readLine(), " ");
            int number;

            for(int i = 1; i <= this.dataCount; i++) {
                number = Integer.parseInt(aToken.nextToken());
                // A 그룹 정보 배열에 A그룹 식별번호 설정
                this.aDataArray[i] = number;
                number = Integer.parseInt(bToken.nextToken());
                // B 그룹 정보 배열에 B그룹 식별번호와 같은 A그룹 정보배열의 인덱스 설정
                this.bDataIndexArray[number] = i;
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int bIndex;
        long tmp;

        // A 그룹 식별번호를 차례대로 방문
        for(int i = 1; i <= this.dataCount; i++) {
            // A 그룹의 식별번호와 같은(연결 된) B 그룹의 인덱스
            bIndex = this.bDataIndexArray[this.aDataArray[i]];
            // 현재 식별번호의 간선과 교차하는 간선의 수
            // B 그룹의 인덱스(A 그룹의 식별번호와 연결 된) ~ dataCount 까지에서 설정된 간선의 수
            tmp = this.sum(this.dataCount) - this.sum(bIndex - 1);
            this.result += tmp;
            // B 그룹의 인덱스에(A 그룹의 식별번호와 연결 된) 간선 정보 설정
            this.update(bIndex, 1);
        }
    }

    public void update(int index, int diff) {
        while(index < this.treeSize) {
            this.fenwickTree[index] += diff;
            index += (index & -index);
        }
    }

    public long sum(int index) {
        long sum = 0;

        while(index > 0) {
            sum += this.fenwickTree[index];
            index -= (index & -index);
        }

        return sum;
    }
}
```  

---
## Reference
[7578-공장](https://www.acmicpc.net/problem/7578)  
