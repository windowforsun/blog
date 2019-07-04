--- 
layout: single
classes: wide
title: "[풀이] 백준 6549 히스토그램에서 가장 큰 직사각형"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '히스토그램에서 가장 큰 직사각형을 구하라'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
---  

# 문제
- 주어진 히스토그램을 하나의 도형으로 만들때, 그 도형 내부에 가장 큰 직사각형의 넓이를 구하라.

## 입력
입력은 테스트 케이스 여러 개로 이루어져 있다. 각 테스트 케이스는 한 줄로 이루어져 있고, 직사각형의 수 n이 가장 처음으로 주어진다. (1 ≤ n ≤ 100,000) 그 다음 n개의 정수 h1, ..., hn (0 ≤ hi ≤ 1,000,000,000)가 주어진다. 이 숫자들은 히스토그램에 있는 직사각형의 높이이며, 왼쪽부터 오른쪽까지 순서대로 주어진다. 모든 직사각형의 너비는 1이고, 입력의 마지막 줄에는 0이 하나 주어진다.

## 출력
각 테스트 케이스에 대해서, 히스토그램에서 가장 넓이가 큰 직사각형의 넓이를 출력한다.

## 예제 입력

```
7 2 1 4 5 1 3 3
4 1000 1000 1000 1000
0
```  

## 예제 출력

```
8
4000
```  

## 풀이
- 히스토그램의 넓이는 각 구간에서 최소 높이에 의해 결정된다.
- 각 구간의 최소 높이는 [세그먼트 트리]({{site.baseurl}}{% link _posts/2019-06-12-algorithm-concept-segmenttree.md %}) 를 통해 구할 수 있다.
- 추가적으로 각 구간에서 최소 높이를 가진 직사각형의 인덱스가 필요하다.
	- 한 구간에서 최소 높이를 통해 그 구간의 넓이를 구한다.
	- 한 구간에서 최소 높이의 직사각형을 제외하고 양 옆의 분리 된 구간의 넓이를 구해 나가야 한다.
- 정리하면 아래와 같다.
	- start ~ end 에서 넓이를 구하기 위해 최소 높이 min 과 그 높이 인덱스 minIndex 를 구하고, 넓이를 계산한다.
	- minIndex 를 기준으로 분리되는 두 구간, start ~ minIndex-1, minIndex+1 ~ end 구간을 위와 같이 반복한다.

```java
public class Main {
    private StringBuilder result;
    private long maxArea;
    private int dataCount;
    private int dataStartIndex;
    private int treeSize;
    private int[] minSegmentTree;
    private int[] minIndexSegmentTree;

    public static void main(String[] args){
        new Main();
    }

    public Main() {
        this.input();
        this.output();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            StringTokenizer token;
            this.result = new StringBuilder();

            while(true) {
                token = new StringTokenizer(reader.readLine(), " ");
                this.dataCount = Integer.parseInt(token.nextToken());

                if(this.dataCount == 0) {
                    break;
                }

                this.dataStartIndex = this.getDataStartIndex(this.dataCount);
                this.treeSize = this.dataStartIndex * 2;
                this.minSegmentTree = new int[this.treeSize];
                this.minIndexSegmentTree = new int[this.treeSize];
                this.maxArea = Long.MIN_VALUE;

                for(int i = this.dataStartIndex; i < this.dataStartIndex + this.dataCount; i++) {
                    this.minSegmentTree[i] = Integer.parseInt(token.nextToken());
                    this.minIndexSegmentTree[i] = i - this.dataStartIndex;
                }

                // 주어진 히스토그램에 있는 직사각형의 높이로 최소 높이 세그먼트 트리와
                // 그에 대응되는 최소 높이 인덱스 세그먼트 트리를 만든다.
                this.initSegmentTree();
                this.solution(0, this.dataCount - 1);

                this.result.append(this.maxArea).append("\n");
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    // start-end 의 최대 넓이를 구한다.
    public void solution(int start, int end){
        if(start > end) {
            return;
        }

        // 최소 높이와 그 높이의 인덱스
        int[] minInfo = this.getMinInfo(start, end);
        int min = minInfo[0];
        int minIndex = minInfo[1];
        long area = ((end - start) + 1) * (long)min;

        if(this.maxArea < area) {
            this.maxArea = area;
        }

        if(start == end) {
            return;
        }

        // 최소 높이를 가진 인덱스를 기준으로 좌우로 최대 넓이를 탐색한다.
        this.solution(start, minIndex - 1);
        this.solution(minIndex + 1, end);
    }

    // 최소 높이와 그 인덱스를 반환한다.
    public int[] getMinInfo(int start, int end) {
        int min = Integer.MAX_VALUE;
        int minIndex = 0;
        start += this.dataStartIndex;
        end += this.dataStartIndex;

        while(start < end) {
            if(start % 2 == 1) {
                if(min > this.minSegmentTree[start]) {
                    min = this.minSegmentTree[start];
                    minIndex = this.minIndexSegmentTree[start];
                }

                start++;
            }

            if(end % 2 == 0) {
                if(min > this.minSegmentTree[end]) {
                    min = this.minSegmentTree[end];
                    minIndex = this.minIndexSegmentTree[end];
                }

                end--;
            }

            start /= 2;
            end /= 2;
        }

        if(start == end) {
            if(min > this.minSegmentTree[start]) {
                min = this.minSegmentTree[start];
                minIndex = this.minIndexSegmentTree[start];
            }
        }

        return new int[]{min, minIndex};
    }

    // 최소 높이 세그먼트 트리와 그 최소 높이의 인덱스 세그먼트 트리를 구성한다.
    public void initSegmentTree() {
        for(int i = this.treeSize - 1; i > 1; i -= 2) {
            if(this.minSegmentTree[i] > this.minSegmentTree[i - 1]) {
                this.minSegmentTree[i / 2] = this.minSegmentTree[i - 1];
                this.minIndexSegmentTree[i / 2] = this.minIndexSegmentTree[i - 1];
            } else {
                this.minSegmentTree[i / 2] = this.minSegmentTree[i];
                this.minIndexSegmentTree[i / 2] = this.minIndexSegmentTree[i];
            }
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
[6549-히스토그램에서 가장 큰 직사각형](https://www.acmicpc.net/problem/6549)  
