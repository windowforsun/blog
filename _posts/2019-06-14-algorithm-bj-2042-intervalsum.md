--- 
layout: single
classes: wide
title: "[풀이] 백준 2042 구간 합 구하기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '세그먼트 트리를 이용해서 변경이 많은 구간합을 구하자'
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
- N 개의 수가 주어지고, 빈번한 수의 변경이 일어 난다.
- 이때 N 개의 수에서 특정 구간의 합을 구하라.

## 입력
첫째 줄에 수의 개수 N(1 ≤ N ≤ 1,000,000)과 M(1 ≤ M ≤ 10,000), K(1 ≤ K ≤ 10,000) 가 주어진다. M은 수의 변경이 일어나는 회수이고, K는 구간의 합을 구하는 회수이다. 그리고 둘째 줄부터 N+1번째 줄까지 N개의 수가 주어진다. 그리고 N+2번째 줄부터 N+M+K+1번째 줄까지 세 개의 정수 a, b, c가 주어지는데, a가 1인 경우 b번째 수를 c로 바꾸고 a가 2인 경우에는 b번째 수부터 c번째 수까지의 합을 구하여 출력하면 된다.

a가 1인 경우 c는 long long 범위를 넘지 않는다.

## 출력
첫째 줄부터 K줄에 걸쳐 구한 구간의 합을 출력한다. 단, 정답은 long long 범위를 넘지 않는다.

## 예제 입력

```
5 2 2
1
2
3
4
5
1 3 6
2 2 5
1 5 2
2 3 5
```  

## 예제 출력

```
17
12
```  

## 풀이
- 반복문 2개로 문제를 풀게 되면 한번의 구간합을 구할때 마다 최대 $O(N^2)$ 의 시간 복잡도가 필요하다.
- [세그먼트 트리]({{site.baseurl}}{% link _posts/2019-06-12-algorithm-concept-segmenttree.md %}) 를 사용하게 되면 $O(\log_2 N)$ 의 시간 복잡도로 구간합을 구할 수 있다.

```java
public class Main {
    private StringBuilder result;
    private int numCount;
    private int updateCount;
    private int queryCount;
    private int treeSize;
    private long[] dataArray;
    private long[] sumSegmentTree;
    private final int UPDATE = 1;
    private final int QUERY = 2;


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
            this.updateCount = Integer.parseInt(token.nextToken());
            this.queryCount = Integer.parseInt(token.nextToken());
            this.dataArray = new long[this.numCount];
            this.treeSize = this.getTreeSize(this.numCount);
            this.sumSegmentTree = new long[this.treeSize + 1];
            this.result = new StringBuilder();

            for(int i = 0; i < this.numCount; i++) {
                this.dataArray[i] = Integer.parseInt(reader.readLine());
            }

            this.initSegmentTree(1, 0, this.numCount - 1);
            int count = this.updateCount + this.queryCount;

            for(int i = 0; i < count; i++) {
                token = new StringTokenizer(reader.readLine(), " ");

                this.solution(Integer.parseInt(token.nextToken()), Integer.parseInt(token.nextToken()), Integer.parseInt(token.nextToken()));
            }


        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution(int type, int indexOrStart, int updateOrEnd) {
        if(type == UPDATE) {
            // 업데이트 수행
            this.update(indexOrStart - 1, updateOrEnd);
        } else {
            // 구간 합
            this.result.append(this.sum(1, 0, this.numCount - 1, indexOrStart - 1, updateOrEnd - 1)).append("\n");
        }
    }

    // 세그먼트 트리 초기화
    public long initSegmentTree(int node, int start, int end) {
        if(start == end) {
            return this.sumSegmentTree[node] = this.dataArray[start];
        }

        int mid = (start + end) / 2;
        long sum = this.initSegmentTree(node * 2, start, mid) + this.initSegmentTree(node * 2 +1, mid + 1, end);
        this.sumSegmentTree[node] += sum;

        return this.sumSegmentTree[node];
    }

    // 값 변경
    public void update(int index, int value) {
        if(index < 0 || index >= this.numCount) {
            return;
        }

        long before = this.dataArray[index];
        long diff = value - before;
        this.dataArray[index] = value;
        this.updateSegmentTree(1, 0, this.numCount - 1, index, diff);
    }

    public void updateSegmentTree(int node, int start, int end, int index, long diff) {
        if(index < start || end < index) {
            return;
        }

        this.sumSegmentTree[node] += diff;

        if(start == end) {
            return;
        }

        int mid = (start + end) / 2;
        this.updateSegmentTree(node * 2, start, mid, index, diff);
        this.updateSegmentTree(node * 2 + 1, mid + 1, end, index, diff);
    }

    // sumStart-sumEnd 의 구간합
    public long sum(int node, int start, int end, int sumStart, int sumEnd) {
        long sum = 0;

        if(end >= sumStart && sumEnd >= start) {
            if(sumStart <= start && end <= sumEnd) {
                sum = this.sumSegmentTree[node];
            } else {
                int mid = (start + end) / 2;
                sum = this.sum(node * 2, start, mid, sumStart, sumEnd) + this.sum(node * 2 + 1, mid + 1, end, sumStart, sumEnd);
            }
        }

        return sum;
    }

    public int getTreeSize(int dataSize) {
        int result = 0;
        int height = (int)Math.ceil(this.logBottomTop(2, dataSize));
        result = (1 << (height + 1));

        return result;
    }

    public double logBottomTop(double bottom, double top) {
        return Math.log(top) / Math.log(bottom);
    }
}
```  

---
## Reference
[2042-구간 합 구하기](https://www.acmicpc.net/problem/2042)  
