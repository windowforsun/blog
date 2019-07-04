--- 
layout: single
classes: wide
title: "[풀이] 백준 10868 최솟값"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '주어진 구간에서 최솟값을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
---  

# 문제
- N 개의 값이 주어지고, 그 값을 지칭가는 a, b 인덱스가 주어진다.
- a~b 사이의 값 중 최솟값을 구하라.

## 입력
첫째 줄에 N, M이 주어진다. 다음 N개의 줄에는 N개의 정수가 주어진다. 다음 M개의 줄에는 a, b의 쌍이 주어진다.

## 출력
M개의 줄에 입력받은 순서대로 각 a, b에 대한 답을 출력한다.

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
5
38
20
5
```  

## 풀이

```java
public class Main {
    private StringBuilder result;
    private int numCount;
    private int queryCount;
    private long[] dataArray;
    private long[] minSegmentArray;
    private int treeSize;

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

            this.numCount = Integer.parseInt(token.nextToken());
            this.queryCount = Integer.parseInt(token.nextToken());
            this.dataArray = new long[this.numCount];
            this.treeSize = this.getTreeSize(this.numCount);
            this.minSegmentArray = new long[this.treeSize];
            this.result = new StringBuilder();

            for(int i = 0; i < this.numCount; i++) {
                this.dataArray[i] = Integer.parseInt(reader.readLine());
            }

            this.initSegmentTree(1, 0, this.numCount - 1);

            for(int i = 0; i < this.queryCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                this.result.append(this.min(1, 0, this.numCount - 1, Integer.parseInt(token.nextToken()) - 1, Integer.parseInt(token.nextToken()) - 1));
                this.result.append("\n");
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public long initSegmentTree(int node, int start, int end) {
        if(start == end) {
            this.minSegmentArray[node] = this.dataArray[start];
        } else {
            int mid = (start + end) / 2;
            long min = Long.min(this.initSegmentTree(node * 2, start, mid), this.initSegmentTree(node * 2 + 1, mid + 1, end));
            this.minSegmentArray[node] = min;
        }

        return this.minSegmentArray[node];
    }

    public long min(int node, int start, int end, int minStart, int minEnd) {
        long min = Long.MAX_VALUE;

        if(end >= minStart && minEnd >= start) {
            if(minStart <= start && end <= minEnd) {
                min = this.minSegmentArray[node];
            } else {
                int mid = (start + end) / 2;
                min = Long.min(this.min(node * 2, start, mid, minStart, minEnd), this.min(node * 2 + 1, mid + 1, end, minStart, minEnd));
            }
        }

        return min;
    }

    public int getTreeSize(int dataSize) {
        int height = (int)Math.ceil(this.logBottomTop(2, dataSize));
        return (1 <<(height + 1));
    }

    public double logBottomTop(double bottom, double top) {
        return Math.log(top) / Math.log(bottom);
    }
}
```  

---
## Reference
[10868-최솟값](https://www.acmicpc.net/problem/10868)  
