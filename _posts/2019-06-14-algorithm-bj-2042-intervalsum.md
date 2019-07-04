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
    private int dataStartIndex;
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
            this.dataStartIndex = this.getDataStartIndex(this.numCount);
            this.treeSize = this.dataStartIndex * 2;
            this.sumSegmentTree = new long[this.treeSize];
            this.result = new StringBuilder();

            for(int i = this.dataStartIndex; i < this.dataStartIndex + this.numCount; i++) {
                this.sumSegmentTree[i] = Long.parseLong(reader.readLine());
            }

            this.initSegmentTree();
            int count = this.updateCount + this.queryCount;

            for(int i = 0; i < count; i++) {
                token = new StringTokenizer(reader.readLine(), " ");

                this.solution(Integer.parseInt(token.nextToken()), Integer.parseInt(token.nextToken()), Long.parseLong(token.nextToken()));
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution(int type, int indexOrStart, long updateOrEnd) {
        if(type == UPDATE) {
            this.update(indexOrStart - 1, updateOrEnd);
        } else {
            this.result.append(this.sum(indexOrStart - 1, updateOrEnd - 1)).append("\n");
        }
    }

    public void initSegmentTree() {
        for(int i = this.treeSize - 1; i > 0; i-= 2) {
            System.out.println((i / 2) + ", " + i + ", " + (i - 1));
            this.sumSegmentTree[i / 2] = this.sumSegmentTree[i] + this.sumSegmentTree[i - 1];
        }
    }

    public void update(int index, long value) {
        index += this.dataStartIndex;
        this.sumSegmentTree[index] = value;
        while(index > 1) {
            index /= 2;
            this.sumSegmentTree[index] = this.sumSegmentTree[index * 2] + this.sumSegmentTree[index * 2 + 1];
        }
    }

    public long sum(int sumStart, long sumEnd) {
        long sum = 0;
        sumStart += this.dataStartIndex;
        sumEnd += this.dataStartIndex;

        while(sumStart < sumEnd) {
            if(sumStart % 2 == 1) {
                sum += this.sumSegmentTree[sumStart];
                sumStart++;
            }

            if(sumEnd % 2 == 0) {
                sum += this.sumSegmentTree[(int)sumEnd];
                sumEnd--;
            }

            sumStart /= 2;
            sumEnd /= 2;
        }

        if(sumStart == sumEnd) {
            sum += this.sumSegmentTree[sumStart];
        }

        return sum;
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
[2042-구간 합 구하기](https://www.acmicpc.net/problem/2042)  
