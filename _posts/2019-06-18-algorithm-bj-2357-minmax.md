--- 
layout: single
classes: wide
title: "[풀이] 백준 2357 최솟값 최댓값"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 구간의 최솟값과 최댓값을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
use_math : true
---  

# 문제
- N 개의 정수가 주어지고, 구간의 시작 a, 구간의 끝 b 인덱스가 주어진다.
- 이때 a~b 인덱스 사이의 최솟값과 최댓값을 구하라.

## 입력
첫째 줄에 N, M이 주어진다. 다음 N개의 줄에는 N개의 정수가 주어진다. 다음 M개의 줄에는 a, b의 쌍이 주어진다.

## 출력
M개의 줄에 입력받은 순서대로 각 a, b에 대한 답을 최솟값, 최댓값 순서로 출력한다.

## 예제 입력

```
10 4
75
30
100
38
50
51
52
20
81
5
1 10
3 5
6 9
8 10
```  

## 예제 출력

```
5 100
38 100
20 81
5 81
```  

## 풀이
- 반복문을 이용해서 구간의 최솟값과 최댓값을 구할 경우 $O(MN)$ 의 시간복잡도로 구할 수 있다.
- [세그먼트 트리]({{site.baseurl}}{% link _posts/2019-06-12-algorithm-concept-segmenttree.md %}) 를 사용하면 이를 $O(M \log_2 N)$ 의 시간 복잡도로 줄일 수 있다.
- 최솟값과, 최댓값을 구해야 하기 때문에 세그먼트 트리도 각각 한개씩 만들면된다.

```java
public class Main {
    private StringBuilder result;
    private int dataCount;
    private int queryCount;
    private int dataStartIndex;
    private int treeSize;
    private int[] minSegmentTree;
    private int[] maxSegmentTree;

    public Main() {
        this.input();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int value;

            this.dataCount = Integer.parseInt(token.nextToken());
            this.queryCount = Integer.parseInt(token.nextToken());
            this.dataStartIndex = this.getDataStartIndex(dataCount);
            this.treeSize = this.dataStartIndex * 2;
            this.minSegmentTree = new int[this.treeSize];
            this.maxSegmentTree = new int[this.treeSize];
            this.result = new StringBuilder();

            for(int i = this.dataStartIndex; i < this.dataStartIndex + this.dataCount; i++) {
                value = Integer.parseInt(reader.readLine());
                this.minSegmentTree[i] = value;
                this.maxSegmentTree[i] = value;
            }

            this.initSegmentTree();

            for(int i = 0; i < this.queryCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                this.getMinMax(Integer.parseInt(token.nextToken()) - 1, Integer.parseInt(token.nextToken()) - 1);
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void getMinMax(int start, int end) {
        int min = Integer.MAX_VALUE;
        int max = Integer.MIN_VALUE;
        start += this.dataStartIndex;
        end += this.dataStartIndex;

        while(start < end) {
            if(start % 2 == 1) {
                min = Integer.min(min, this.minSegmentTree[start]);
                max = Integer.max(max, this.maxSegmentTree[start]);
                start++;
            }

            if(end % 2 == 0) {
                min = Integer.min(min, this.minSegmentTree[end]);
                max = Integer.max(max, this.maxSegmentTree[end]);
                end--;
            }

            start /= 2;
            end /= 2;
        }

        if(start == end) {
            min = Integer.min(min, this.minSegmentTree[start]);
            max = Integer.max(max, this.maxSegmentTree[start]);
        }

        this.result.append(min).append(" ").append(max).append("\n");
    }

    public void initSegmentTree() {
        for(int i = this.treeSize - 1; i > 1; i -= 2) {
            this.minSegmentTree[i / 2] = Integer.min(this.minSegmentTree[i], this.minSegmentTree[i - 1]);
            this.maxSegmentTree[i / 2] = Integer.max(this.maxSegmentTree[i], this.maxSegmentTree[i - 1]);
        }
    }

    public int getDataStartIndex(int dataCount) {
        int result = 1;

        while(result < dataCount) {
            result <<= 1;
        }

        return result;
    }
}
```  

---
## Reference
[2357-최솟값 최댓값](https://www.acmicpc.net/problem/2357)  
