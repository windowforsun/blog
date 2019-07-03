--- 
layout: single
classes: wide
title: "[Algorithm 개념] 세그먼트 트리(Segment Tree)"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '세그먼트 트리를 구현하고 특정 구간의 값을 구해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
use_math : true
---  

## 세그먼트 트리(Segment Tree)란
- 이름 그대로 구간을 보존하고 있는 트리이다.
- 주로 완전 이진트리 혹은 포화 이진트리 형태를 사용한다.
- 주워진 쿼리에 대해 빠르게 응답하기 위한 자료구조이다.
	- 특정 구간의 합을 구한다.
	- 특정 구간의 최대, 최소 값을 구한다.
- 쿼리를 수행할 배열을 트리로 변환하여 관리하는데, 포인터 형식의 트리가 아닌 배열을 통해 트리를 구성한다.
	
## 세그먼트 트리의 특징
- 대체적으로 3개의 연산으로 구성되고, 시간 복잡도는 아래와 같다.
	- 세그먼트 트리 구성 : $O(2^{\lceil \log_2 N \rceil + 1})$ (이후 추가 설명)
	- 쿼리 수행(구간 합, 최대, 최소 ..) : $O(\log_2 N)$
	- 쿼리 수행 배열의 값 변경 : $O(\log_2 N)$
- 리프 노드는 쿼리를 수행 할 배열에 해당하는 데이터를 의미한다.
- 리프 노드를 제외한 다른 노드(루트 노드 포함)는 왼쪽 자식과 오른쪽 자식에 대한 쿼리 결과를 저장한다.

## 구간 합 세그먼트 트리
- 세그먼트 트리에서 가장 대중적인 구간 합을 예시로 들겠다.
- 아래와 같은 구간합을 구해야하는 배열 D가 있다.

index|0|1|2|3|4|5|6|7|8|9|10|11
---|---|---|---|---|---|---|---|---|---|---|---|---
value|1|2|3|4|5|6|7|8|9|10|11|12

- 세그먼트 트리의 정보는 배열에 담아지기 때문에 세그먼트 배열이 필요하고 적당한 크기가 필요하다.
	- 가장 간단한 방법은 dataSize(D 의 크기) * 4 를 하면된다.
	- dataSize 보다 큰 $2^k$ 의 k 를 찾아서 $2^{k +1}$ 값이 세그먼트 트리 배열의 크기이다.
	- 이를 수식으로 작성하면 다음과 같다. $2^{\lceil \log_2 N \rceil + 1}$
	- dataSize 가 12일 경우 세그먼트 트리 배열의 크기는 32가 된다.

- D 배열을 세그먼트 트리(S)로 구성하면 아래 그림과 같다.
- 각 노드는 (D 배열에서 쿼리 인덱스 범위), 쿼리 결과이자 S 배열의 값, 아래쪽에 S 배열의 인덱스로 구성되어 있다.

![그림1]({{site.baseurl}}/img/algorithm/concept-segmenttree-1.png)

- 부모노드는 2개의 자식을 가지게 되는데 각 자식의 인덱스를 알아내는 방법은 아래와 같다.
	- 부모 노드 인덱스 * 2 : 왼쪽 자식
	- (부모 노드 인덱스 * 2) + 1 : 오른쪽 자식
	
### 세그먼트 트리 구성
- 세그먼트 트리를 만들때 S 배열에 값이 들어가는 순서는 세그먼트 트리를 후위 순회한 결과랑 같다.
	- 1-2-3-3-6-4-5-9-6-15-21-7-8-15-9-24-10-11-21-12-33-57-78

- 세그먼트 트리를 만드는 메서드는 아래와 같다.

```java
/**
 * dataArray 를 기반으로 Segment Tree 생성
 * @param node 세그먼트 트리 배열에서 인덱스
 * @param start 데이터 배열에서 시작 인덱스
 * @param end 데이터 배열에서 종료 인덱스
 * @return 쿼리 결과값
 */
public long init(int node, int start, int end) {
    if(start == end) {
        // 리프 노드일 경우 (데이터 범위가 아니라, 하나의 데이터 일때)
        this.segmentTreeArray[node] = this.dataArray[start];
    } else {
        int mid = (start + end) / 2;
        // 현재 node 의 값은 node 의 좌측 자식 + 우측 자식
        long sum = this.init(node * 2, start, mid) + this.init(node * 2 + 1, mid + 1, end);
        this.segmentTreeArray[node] += sum;
    }

    return this.segmentTreeArray[node];
}
```  

- 해당 메서드 호출는 다음과 같다.

```java
this.init(1, 0, this.dataSize - 1);
```  

- 루트 노드를 호출로 시작해서 좌측, 우측 자식을 계속 방문하며 리프 노드일 경우 해당 start=end 에 해당하는 데이터 배열의 값을 넣어주고, 리프 노드가 아닐 경우 좌측 자식 + 우측 자식 합을 세그먼트 트리 배열 node 인덱스에 넣어 준다. 

### 쿼리 수행(구간 합)
- 구간 합을 구하는 메서드는 아래와 같다.

```java
/**
 * sumStart-sumEnd(dataArray 의 인덱스) 범위의 합을 구함
 * @param node 세그먼트 트리 배열에서 인덱스
 * @param start 데이터 배열에서 시작 인덱스
 * @param end 데이터 배열에서 종료 인덱스
 * @param sumStart 쿼리 시작 인덱스(데이터 배열)
 * @param sumEnd 쿼리 종료 인덱스(데이터 배열)
 * @return sumStart-sumEnd 의 쿼리 결과
 */
public long sum(int node, int start, int end, int sumStart, int sumEnd) {
    long sum = 0;

    // start-end 의 볌위가 sumStart-sumEnd 의 범위 어느 한 부분이라도 포함될 경우
    if(end >= sumStart && sumEnd >= start) {
        if(sumStart <= start && end <= sumEnd) {
            // start-end 범위가 sumStart-sumEnd 의범위와 같거나 포함될 경우
            sum = this.segmentTreeArray[node];
        } else {
            // 같거나 포함되지 않는 경우는 좌, 우 자식 중 sumStart-sumEnd 범위에 포함되는 합을 구함
            int mid = (start + end) / 2;
            sum = this.sum(node * 2, start, mid, sumStart, sumEnd) + this.sum(node * 2 + 1, mid + 1, end, sumStart, sumEnd);
        }
    }

    return sum;
}

/**
 * 구간 합을 구한다.
 * @param sumStart 쿼리 시작 인덱스(데이터 배열)
 * @param sumEnd 쿼리 종료 인덱스(데이터 배열)
 * @return sumStart-sumEnd 의 쿼리 결과
 */
public long getSum(int sumStart, int sumEnd) {
    return this.sum(1, 0, this.dataSize - 1, sumStart, sumEnd);
}
```  

- 2~5 인덱스에 해당하는 구간 합을 구할 때 메서드 호출은 아래와 같다.

```java
this.getSum(2, 5);
```  

- 2~5 인덱스 구간 합에서 방문 하는 노드는 아래와 같다.

![그림2]({{site.baseurl}}/img/algorithm/concept-segmenttree-2.png)




---
## Reference
[세그먼트 트리(Segment Tree) (수정: 2019-02-12)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220791986409)  
[세그먼트 트리(Segment Tree)](https://www.crocus.co.kr/648)  
[Segment Tree를 활용한 2D Range Update/Query](http://www.secmem.org/blog/2018/12/23/segment-tree-2d-range-update-query/)  
[세그먼트 트리 (Segment Tree)](https://www.acmicpc.net/blog/view/9)  
