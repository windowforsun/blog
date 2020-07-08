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
- 주워진 쿼리에 대해 빠르게 응답하기 위한 자료구조이다.
	- 특정 구간의 합을 구한다.
	- 특정 구간의 최대, 최소 값을 구한다.
- 쿼리를 수행할 배열을 트리로 변환하여 관리하는데, 포인터 형식의 트리가 아닌 배열을 통해 트리를 구성한다.
- 리프 노드는 쿼리를 수행 할 배열에 해당하는 데이터를 의미한다.
- 리프 노드를 제외한 다른 노드(루트 노드 포함)는 왼쪽 자식과 오른쪽 자식에 대한 쿼리 결과를 저장한다.
- 대체적으로 3개의 연산으로 구성되고, 시간 복잡도는 아래와 같다.
	- 세그먼트 트리 구성 : $O(2^{\lceil \log_2 N \rceil + 1})$
	- 쿼리 수행(구간 합, 최대, 최소 ..) : $O(\log_2 N)$
	- 쿼리 수행 배열의 값 변경 : $O(\log_2 N)$

## 구간 합 세그먼트 트리 - 1
- 완전 이진트리를 구성하며 세그먼트 트리를 구성하고 쿼리의 결과값을 도출하는 방법에 대해 알아본다.
	
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

- 1~5 인덱스에 해당하는 구간 합을 구할 때 메서드 호출은 아래와 같고 결과 값은 20 이다.

```java
this.getSum(1, 5);
```  

- 1~5 인덱스 구간 합에서 방문 하는 노드는 아래와 같다.
	- 방문만 하는 노드는 회색 원
	- 쿼리의 결과 값으로 사용된 노드는 붉은 원

![그림2]({{site.baseurl}}/img/algorithm/concept-segmenttree-2.png)

### 값 변경
- 세그먼트 트리에서 값을 변경하는 메서드는 아래와 같다.

```java
/**
 * dataArray 의 값을 변경하고 관련된 Segment Tree 의 값도 변경
 * @param index
 * @param value
 */
public void updateData(int index, int value) {
    if(index < 0 || index >= this.dataSize) {
        return;
    }

    // 갱신 전 데이터
    long before = this.dataArray[index];
    // 갱신 하려는 데이터와 기존 데이터의 차
    long diff = value - before;
    // dataArray 에는 갱신 데이터로 설정
    this.dataArray[index] = value;
    // Segment Tree 에는 데이터의 차를 더해 줌
    this.update(1, 0, this.dataSize - 1, index, diff);
}

/**
 * dataArray 의 index 에 해당하는 값 변경에 따른
 * Segment Tree 업데이트 (재귀)
 * @param node
 * @param start
 * @param end
 * @param index
 * @param diff
 */
public void update(int node, int start, int end, int index, long diff) {
    // start-end 범위에 포함되지 않는 index 일 경우
    if(index < start || end < index) {
        return;
    }

    // start-end 범위에 포함될 경우 값을 diff 만큼 더해줌
    this.segmentTreeArray[node] += diff;

    // 리프 노드로 왔을 때 종료
    if(start == end) {
        return;
    }

    // node 의 좌, 우 노드 갱신 작업 수행
    int mid = (start + end) / 2;
    this.update(node * 2, start, mid, index, diff);
    this.update(node * 2 + 1, mid + 1, end, index, diff);
}
```  

- 4번 인덱스의 값을 14로 변경할 때 변경되는 노드는 아래와 같다.

![그림3]({{site.baseurl}}/img/algorithm/concept-segmenttree-3.png)

### 전체 소스코드

```java
public class Main {
    private int[] dataArray;
    private long[] segmentTreeArray;
    private int treeSize;
    private int dataSize;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.dataArray = new int[]{
                1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12
        };

        this.dataSize = this.dataArray.length;
        this.treeSize = this.getTreeSize(this.dataSize);
        this.segmentTreeArray = new long[this.treeSize];
        this.init(1, 0, this.dataSize - 1);

        System.out.println(Arrays.toString(this.dataArray));
        System.out.println(this.getSum(0, 2));

        this.updateData(0, -1);
        System.out.println(Arrays.toString(this.dataArray));
        System.out.println(this.getSum(0, 3));

        this.updateData(1, -100);
        System.out.println(Arrays.toString(this.dataArray));
        System.out.println(this.getSum(1, 4));

        this.updateData(2, 10000);
        System.out.println(Arrays.toString(this.dataArray));
        System.out.println(this.getSum(2, 5));

    }

    /**
     * dataArraySize 를 기반으로 Segment Tree 배열의 크기 계산
     * @param dataSize
     * @return
     */
    public int getTreeSize(int dataSize) {
        int result = 0;
        int height = (int) Math.ceil(this.logBottomTop(dataSize, 2));
        // Segment Tree 배열의 크기는 dataArraySize 보다 크기지만 가장 작은 2^N의 N 값
        result = (1 << (height + 1));

        return result;
    }

    public double logBottomTop(double top, double bottom) {
        return Math.log(top) / Math.log(bottom);
    }

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

    /**
     * dataArray 의 값을 변경하고 관련된 Segment Tree 의 값도 변경
     * @param index
     * @param value
     */
    public void updateData(int index, int value) {
        if(index < 0 || index >= this.dataSize) {
            return;
        }

        // 갱신 전 데이터
        long before = this.dataArray[index];
        // 갱신 하려는 데이터와 기존 데이터의 차
        long diff = value - before;
        // dataArray 에는 갱신 데이터로 설정
        this.dataArray[index] = value;
        // Segment Tree 에는 데이터의 차를 더해 줌
        this.update(1, 0, this.dataSize - 1, index, diff);
    }
    /**
     * dataArray 의 index 에 해당하는 값 변경에 따른
     * Segment Tree 업데이트 (재귀)
     * @param node
     * @param start
     * @param end
     * @param index
     * @param diff
     */
    public void update(int node, int start, int end, int index, long diff) {
        // start-end 범위에 포함되지 않는 index 일 경우
        if(index < start || end < index) {
            return;
        }

        // start-end 범위에 포함될 경우 값을 diff 만큼 더해줌
        this.segmentTreeArray[node] += diff;

        // 리프 노드로 왔을 때 종료
        if(start == end) {
            return;
        }

        // node 의 좌, 우 노드 갱신 작업 수행
        int mid = (start + end) / 2;
        this.update(node * 2, start, mid, index, diff);
        this.update(node * 2 + 1, mid + 1, end, index, diff);
    }

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

}
```  

## 구간 합 세그먼트 트리 - 2
- 방법 1과 달리 완전 이진 트리를 유지하지는 않지만 비교적 빠르게 세그먼트 트리를 구성하고 쿼리의 결과값을 도출하는 방법에 대해 알아본다.
- 방법 1과 동일한 배열 D가 있다.

index|0|1|2|3|4|5|6|7|8|9|10|11
---|---|---|---|---|---|---|---|---|---|---|---|---
value|1|2|3|4|5|6|7|8|9|10|11|12

- 세그먼트 트리 배열의 크기를 구하기 전에 세그먼트 트리 배열에서 D 배열의 값들이 위치해야하는 첫번째 인덱스를 찾아야 한다.
- 데이터 값이 위치하는 첫번째 인덱스와 세그먼트 트리 배열 크기는 아래와 같이 구할 수 있다.
	- 데이터 값이 위치하는 첫번째 인덱스(ds) = $2^{\lceil \log_2 N \rceil}$
	- 세그먼트 트리 배열의 크기 = $2^{\lceil \log_2 N \rceil + 1}$
- D 배열을 세그먼트 트리(S)로 구성하면 아래 그림과 같다.
- 각 노드는 (D 배열에서 쿼리 인덱스 범위), 쿼리 결과이자 S 배열의 값, 아래쪽에 S 배열의 인덱스로 구성되어 있다.

![그림4]({{site.baseurl}}/img/algorithm/concept-segmenttree-4.png)

- 부모노드는 2개의 자식을 가지게 되는데 각 자식의 인덱스를 알아내는 방법은 아래와 같다.
	- 부모 노드 인덱스 * 2 : 왼쪽 자식
	- (부모 노드 인덱스 * 2) + 1 : 오른쪽 자식

### 세그먼트 트리 구성
- 방법 1과 지금까지는 트리의 구조 외에 많은 부분에서 공통점을 보였지만, 트리를 구성하는 방법 부터 차이점이 보인다.
- 방법 1은 루트 노드에서부터 세그먼트 트리의 값을 넣으며 완전 이진 트리형 세그먼트 트리를 구성했다.
- 방법 2는 세그먼트 트리 배열(S)의 ds 인덱스 부터 데이터 배열의 값을 먼저 넣는다.
- 그리고 아래 메서드를 통해 리프노드 부터 시작해서 세그먼트 트리를 구성한다.

```java
/**
 * 세그먼트 트리 생성
 * treeSize / 2 인덱스 부터 트리를 생성할 값들이 들어 있는 상태
 */
public void initSegmentTree() {
    // 트리 배열의 마지막 인덱스부터 2씩 감소
    for(int i = this.treeSize - 1; i > 0; i -= 2) {
        // i / 2 는 부모, i 는 왼쪽자식, i - 1 은 오른쪽 자식
        this.segmentTreeArray[i / 2] = this.segmentTreeArray[i] + this.segmentTreeArray[i - 1];
    }
}
```  

- 세그먼트 트리가 생성되는 순서는 미리 들어가 있는 리프 노드를 제외하고 세그먼트 트리 배열 인덱스가 큰것 부터 루트노드(1) 순으로 생성된다.

### 쿼리 수행(구간 합)
- 구간 합을 구하는 메서드는 아래와 같다.

```java
/**
 * 데이터 배열 인덱스 sumStart-sumEnd 구간의 합을 구한다.
 * @param sumStart 데이터 배열에서 시작 인덱스
 * @param sumEnd 데이터 배열에서 종료 인덱스
 * @return
 */
public long getSum(int sumStart, int sumEnd) {
    long sum = 0;

    // sumStart == sumEnd 일때 까지 반복
    while(sumStart < sumEnd) {

        // 구간 합 시작 값이 sumStart 의 부모 노드(sumStart / 2)를 기준으로 오른쪽 자식일 경우
        if(sumStart % 2 == 1) {
            // 구간 합 값에 추가
            sum += this.segmentTreeArray[sumStart];
            // 시작 값을 오른쪽 부모의 왼쪽 자식으로 이동
            sumStart++;
        }

        // 구간 합 종료 값이 sumEnd 의 부모 노드(sumEnd / 2)를 기준으로 왼쪽 자식일 경우
        if(sumEnd % 2 == 0) {
            // 구간 합 값에 추가
            sum += this.segmentTreeArray[sumEnd];
            // 종료 값을 왼쪽 부모의 오른쪽 자식으로 이동
            sumEnd--;
        }

        // 계속해서 루트로 이동
        sumStart /= 2;
        sumEnd /= 2;
    }

    // 구간 합에 해당하는 노드를 찾은 경우
    if(sumStart == sumEnd) {
        sum += this.segmentTreeArray[sumStart];
    }

    return sum;
}
```  

- 1~5 인덱스에 해당하는 구간 합을 구할 때 메서드 호출은 아래와 같고 결과 값은 20 이다.

```java
this.getSum(1, 5);
```  

- 1~5 인덱스 구간 합에서 방문 하는 노드는 아래와 같다.
	- 방문만 하는 노드는 회색 원
	- 쿼리의 결과 값으로 사용된 노드는 붉은 원

![그림5]({{site.baseurl}}/img/algorithm/concept-segmenttree-5.png)

- 주의해야 할 점은 쿼리에서 시작 노드 인덱스는 짝수, 종료 노드에서 인덱스는 홀수 일때 만 부모노드의 값이 현재 쿼리의 결과 값으로 적용된다.
	- 시작 노드 인덱스가 홀수 이거나, 종료 노드 인덱스가 짝수 일 경우 현재 쿼리에서 해당 인덱스의 값은 부모 노드의 값을 통해 구할 수 없고 자신만 쿼리의 결과 값으로 적용된다.
	- 그런 이유로 위의 상황에서 시작 인덱스는 +1 을 해서 오른쪽 부모의 왼쪽 자식으로 이동하고, 종료 인덱스는 -1을 통해 왼쪽 부모의 오른쪽 자식으로 이동한다.


### 값 변경
- 값 변경 관련 메서드는 아래와 같다.

```java
/**
 * 데이터 배열 index 에 해당하는 값을 value 변경 및 세그먼트 트리의 값도 변경
 * @param index 데이터 배열에서 인덱스
 * @param value 변경할 값
 */
public void update(int index, long value) {
    // 데이터 배열에서 index 는 세그먼트 트리 배열에서 index + (treeSize / 2)
    index += this.dataStartIndex;
    this.segmentTreeArray[index] = value;

    // index 에 해당하는 리프 노드에서 부터 루트 노드까지 관련 있는 값을 변경
    while(index > 1) {
        index /= 2;
        // index 부모 노드, index * 2 왼쪽 자식, index * 2 + 1 오른쪽 자식
        this.segmentTreeArray[index] = this.segmentTreeArray[index * 2] + this.segmentTreeArray[index * 2 + 1];
    }
}
```  

### 전체 소스 코드

```java
public class Main {
    private int[] dataArray;
    private long[] segmentTreeArray;
    private int treeSize;
    private int dataStartIndex;
    private int dataSize;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.dataArray = new int[]{
                1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12
        };
        this.dataSize = this.dataArray.length;
        // 세그먼트 트리 배열에서 데이터 배열의 값이 들어가는 시작위치
        this.dataStartIndex = this.getTreeSize(this.dataSize);
        // 트리 배열의 크기는 데이터 배열의 시작 위치 * 2
        this.treeSize = this.dataStartIndex * 2;
        this.segmentTreeArray = new long[this.treeSize];

        // 세그먼트 트리 배열에 데이터 값 설정
        for(int i = this.dataStartIndex; i < this.dataStartIndex + this.dataSize; i++) {
            this.segmentTreeArray[i] = this.dataArray[i - this.dataStartIndex];
        }

        this.initSegmentTree();

        System.out.println(Arrays.toString(this.dataArray));
        System.out.println(this.getSum(this.dataStartIndex + 0, this.dataStartIndex + 2));
        System.out.println(this.getSum(this.dataStartIndex + 1, this.dataStartIndex + 1));
        System.out.println(this.getSum(this.dataStartIndex + 2, this.dataStartIndex + 5));
        System.out.println(this.getSum(this.dataStartIndex + 2, this.dataStartIndex + 10));
        System.out.println(this.getSum(this.dataStartIndex + 0, this.dataStartIndex + 10));
        System.out.println(this.getSum(this.dataStartIndex + 1, this.dataStartIndex + 11));
        System.out.println();
        this.update(2, 4);
        System.out.println(this.getSum(this.dataStartIndex + 0, this.dataStartIndex + 2));
        System.out.println(this.getSum(this.dataStartIndex + 1, this.dataStartIndex + 1));
        System.out.println(this.getSum(this.dataStartIndex + 2, this.dataStartIndex + 5));
        System.out.println(this.getSum(this.dataStartIndex + 2, this.dataStartIndex + 10));
        System.out.println(this.getSum(this.dataStartIndex + 0, this.dataStartIndex + 10));
        System.out.println(this.getSum(this.dataStartIndex + 1, this.dataStartIndex + 11));
        System.out.println();
        this.update(11, 22);
        System.out.println(this.getSum(this.dataStartIndex + 0, this.dataStartIndex + 2));
        System.out.println(this.getSum(this.dataStartIndex + 1, this.dataStartIndex + 1));
        System.out.println(this.getSum(this.dataStartIndex + 2, this.dataStartIndex + 5));
        System.out.println(this.getSum(this.dataStartIndex + 2, this.dataStartIndex + 10));
        System.out.println(this.getSum(this.dataStartIndex + 0, this.dataStartIndex + 10));
        System.out.println(this.getSum(this.dataStartIndex + 1, this.dataStartIndex + 11));
        System.out.println(this.getSum(this.dataStartIndex + 11, this.dataStartIndex + 11));
    }

    /**
     * 세그먼트 트리 생성
     * treeSize / 2 인덱스 부터 트리를 생성할 값들이 들어 있는 상태
     */
    public void initSegmentTree() {
        // 트리 배열의 마지막 인덱스부터 2씩 감소
        for(int i = this.treeSize - 1; i > 0; i -= 2) {
            // i / 2 는 부모, i 는 왼쪽자식, i - 1 은 오른쪽 자식
            this.segmentTreeArray[i / 2] = this.segmentTreeArray[i] + this.segmentTreeArray[i - 1];
        }
    }

    /**
     * 데이터 배열 index 에 해당하는 값을 value 변경 및 세그먼트 트리의 값도 변경
     * @param index 데이터 배열에서 인덱스
     * @param value 변경할 값
     */
    public void update(int index, long value) {
        // 데이터 배열에서 index 는 세그먼트 트리 배열에서 index + (treeSize / 2)
        index += this.dataStartIndex;
        this.segmentTreeArray[index] = value;

        // index 에 해당하는 리프 노드에서 부터 루트 노드까지 관련 있는 값을 변경
        while(index > 1) {
            index /= 2;
            // index 부모 노드, index * 2 왼쪽 자식, index * 2 + 1 오른쪽 자식
            this.segmentTreeArray[index] = this.segmentTreeArray[index * 2] + this.segmentTreeArray[index * 2 + 1];
        }
    }

    /**
     * 데이터 배열 인덱스 sumStart-sumEnd 구간의 합을 구한다.
     * @param sumStart 데이터 배열에서 시작 인덱스
     * @param sumEnd 데이터 배열에서 종료 인덱스
     * @return
     */
    public long getSum(int sumStart, int sumEnd) {
        long sum = 0;

        // sumStart == sumEnd 일때 까지 반복
        while(sumStart < sumEnd) {

            // 구간 합 시작 값이 sumStart 의 부모 노드(sumStart / 2)를 기준으로 오른쪽 자식일 경우
            if(sumStart % 2 == 1) {
                // 구간 합 값에 추가
                sum += this.segmentTreeArray[sumStart];
                // 시작 값을 오른쪽 부모의 왼쪽 자식으로 이동
                sumStart++;
            }

            // 구간 합 종료 값이 sumEnd 의 부모 노드(sumEnd / 2)를 기준으로 왼쪽 자식일 경우
            if(sumEnd % 2 == 0) {
                // 구간 합 값에 추가
                sum += this.segmentTreeArray[sumEnd];
                // 종료 값을 왼쪽 부모의 오른쪽 자식으로 이동
                sumEnd--;
            }

            // 계속해서 루트로 이동
            sumStart /= 2;
            sumEnd /= 2;
        }

        // 구간 합에 해당하는 노드를 찾은 경우
        if(sumStart == sumEnd) {
            sum += this.segmentTreeArray[sumStart];
        }

        return sum;
    }

    /**
     *
     * @param dataCount
     * @return
     */
    public int getTreeSize(int dataCount) {
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
[세그먼트 트리(Segment Tree) (수정: 2019-02-12)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220791986409)  
[세그먼트 트리(Segment Tree)](https://www.crocus.co.kr/648)  
[Segment Tree를 활용한 2D Range Update/Query](http://www.secmem.org/blog/2018/12/23/segment-tree-2d-range-update-query/)  
[세그먼트 트리 (Segment Tree)](https://www.acmicpc.net/blog/view/9)  
