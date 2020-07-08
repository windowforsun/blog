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
use_math : true
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
- 반복문을 통해서만 문제를 해결하면 최대 $O(NM)$ 의 시간 복잡도가 소요된다.
- [세그먼트 트리]({{site.baseurl}}{% link _posts/algorithm/2019-06-12-algorithm-concept-segmenttree.md %}) 를 사용하면 이를 $O(M \log_2 N)$ 의 시간 복잡도로 줄일 수 있다.

```java
public class Main {
    private StringBuilder result;
    private int numCount;
    private int queryCount;
    private int[] minSegmentArray;
    private int dataStartIndex;
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
            this.dataStartIndex = this.getDataStartIndex(this.numCount);
            this.treeSize = this.dataStartIndex * 2;
            this.minSegmentArray = new int[this.treeSize];
            this.result = new StringBuilder();

            Arrays.fill(this.minSegmentArray, Integer.MAX_VALUE);

            for(int i = this.dataStartIndex; i < this.dataStartIndex + this.numCount; i++) {
                this.minSegmentArray[i] = Integer.parseInt(reader.readLine());
            }

            this.initSegmentTree();

            for(int i = 0; i < this.queryCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                this.result.append(this.min(Integer.parseInt(token.nextToken()) - 1, Integer.parseInt(token.nextToken()) - 1));
                this.result.append("\n");
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void initSegmentTree() {
        for(int i = this.treeSize - 1; i > 0; i -= 2) {
            this.minSegmentArray[i / 2] = Integer.min(this.minSegmentArray[i], this.minSegmentArray[i - 1]);
        }
    }

    public int min(int minStart, int minEnd) {
        int min = Integer.MAX_VALUE;
        minStart += this.dataStartIndex;
        minEnd += this.dataStartIndex;

        while(minStart < minEnd) {
            if(minStart % 2 == 1) {
                min = Integer.min(min, this.minSegmentArray[minStart]);
                minStart++;
            }

            if(minEnd % 2 == 0) {
                min = Integer.min(min, this.minSegmentArray[minEnd]);
                minEnd--;
            }

            minStart /= 2;
            minEnd /= 2;
        }

        if(minStart == minEnd) {
            min = Integer.min(min, this.minSegmentArray[minStart]);
        }

        return min;
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
[10868-최솟값](https://www.acmicpc.net/problem/10868)  
