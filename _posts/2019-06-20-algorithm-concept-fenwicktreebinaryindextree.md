--- 
layout: single
classes: wide
title: "[Algorithm 개념] 펜윅 트리(Fenwick Tree, Binary Indexed Tree(BIT))"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '펜윅 트리에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Fenwick Tree
  - Binary Indexed Tree
  - Segment Tree
  - BIT
use_math : true
---  

## 펜윅 트리(Fenwick Tree)란
- 펜윅 트리는 [세그먼트 트리]({{site.baseurl}}{% link _posts/2019-06-12-algorithm-concept-segmenttree.md %}) 에서 메모리를 더 절약 가능한 방법이다.
	- 세그먼트 트리를 구성하기 위해서는 $2^{\lceil \log_2 N \rceil + 1}$ 의 공간이 필요하다.
	- 펜윅 트리를 구성하기 위해서는 $N+1$ 의 공간이 필요하다.
- 세그먼트 트리와 많은 부분에서 비슷하며, 코드는 비교적 간결한다.
- 시간 복잡도 또한 세그먼트 트리와 동일하다.
	- 트리 구성 : $O(2^{\lceil \log_2 N \rceil + 1})$ 보다는 더 소요됨
	- 쿼리 수행(구간 합, 최대, 최소 ..) : $O(\log_2 N)$
	- 쿼리 수행 배열의 값 변경 : $O(\log_2 N)$
- 펜윅 트리는 비트 연산을 기초로 각 동작을 수행한다.
	- 비트 연산을 사용해야 하기때문에 펜윅 트리 배열은 1부터 시작해야 한다.
- Binary Indexed Tree(BIT) 라고도 부른다.
	
## 펜윅 트리(Fenwick Tree) 의 원리
- 펜윅 트리는 각 구간의 합을 구하지 않고, 부분 합만을 빠르게 계산하여 부분 합을 통해 구간 합을 계산할 수 있다.
	- 1~12 의 인덱스를 가진 배열에서 3~6의 구간을 합을 구할때 펜윅 트리는 `1~6부분합 - 1~(3-1)부분합`을 통해 결과 값을 구한다.
	
## 예시
- 펜윅 트리도 세그먼트 트리와 동일하게 구간 합을 예시로 든다.
- 아래와 같은 구간합을 구해야하는 배열 D가 있다.

index|0|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15
---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
value|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16

- D배열을 세그먼트 트리로 나타내면 아래와 같다.

![그림1]({{site.baseurl}}/img/algorithm/concept-fenwicktree-1.png)

- D배열을 펜윅 트리로 나타내면 아래와 같다.
	- 펜윅 트리에서 사용하지 않는 구간(값)은 `.`으로 표시했다.
	- 각 노드의 정보는 `D배열인덱스구간(합)` 로 구성되어 있다.
	
![그림2]({{site.baseurl}}/img/algorithm/concept-fenwicktree-2.png)

- 펜윅 트리 배열(F) 의 인덱스까지 함께 표현하면 아래와 같다.
	- 각 노드의 정보는 `D배열인덱스구간(합)[F배열인덱스]` 로 구성되어 있다.

![그림3]({{site.baseurl}}/img/algorithm/concept-fenwicktree-3.png)

### 값 변경 및 펜윅 트리 생성
- 값을 변경하는 메서드는 아래와 같다.

```java
/**
 * 데이터 배열과 펜윅 트리 index 에 해당하는 값 변경
 * @param index
 * @param value
 */
public void updateValue(int index, int value) {
	// 데이터 배열이 0번째 부터 시작하고 펜윅 트리 배열은 1번째 부터 시작하기 때문에
	int diff = value - this.dataArray[index - 1];
	this.dataArray[index - 1] = value;
	this.update(index, diff);
}

/**
 * 펜윅 트리에서 index 에 해당하는 값 diff 만큼 변경
 * @param index
 * @param diff
 */
public void update(int index, int diff) {

	while(index < this.treeSize) {
		this.fenwickTree[index] += diff;
		// index 의 2진 값 중 가장 오른쪽(최하위 비트) 1에 1더하기
		index += (index & -index);
	}
}
```  

- D배열 2인덱스의 값을 4로 변경하는 호출은 아래와같다.

```java
this.updateValue(1, 2);
```  

- 호출에 따른 펜윅 트리의 변화를 표현하면 아래 그림과 같다.
	- F 배열의 `1->2->4->8->16` 인덱스 순으로 값이 변경되었다.

![그림4]({{site.baseurl}}/img/algorithm/concept-fenwicktree-4.png)


- 9인덱스 값을 10으로 변경할 때, 펜윅 트리의 변화를 표현하면 아래 그림과 같다.
	- F 배열의 `9->10->12->16` 인덱스 순으로 값이 변경되었다.
	
![그림5]({{site.baseurl}}/img/algorithm/concept-fenwicktree-5.png)

- 펜윅 트리는 비트연산에 기반을 둔다고 언급했었다. 변경이 된 인덱스를 비트로 변경하면 아래와 같다.
	- 1인덱스 값 변경 : `1->10->100->1000->10000`
	- 9인덱스 값 변경 : `1001->1010->1100->10000`
	
- 펜윅 트리 값 변경에서 다음 인덱스의 계산은 `인덱스의 2진 값에서 가장 오른쪽에 위치한 1(가장 하위 비트)에 1을 더한다.` 이다.
	- `1+1=10`
	- `10+10=100`
	- `1001+1=1010`
	- `1010+10=1100`

- `update` 메서드에서 `index += (index & -index);` 부분을 다시 표현하면 아래와 같다.
	- `index + (index & -index) = nextIndex`
	- `11000 + 1000 = 100000`
	- 11000의 음수 표현은 XOR(11000) 한 후에 1을 더해주면 된다.
	
- 펜윅 트리의 생성은 `update` 메서드를 통해 D배열의 값을 넣어 모두 호출해 주면 된다.

```java
for(int i = 0; i < this.dataCount; i++) {
	this.update(i + 1, this.dataArray[i]);
}

/**
 * 펜윅 트리에서 index 에 해당하는 값 diff 만큼 변경
 * @param index
 * @param diff
 */
public void update(int index, int diff) {

	while(index < this.treeSize) {
		this.fenwickTree[index] += diff;
		// index 의 2진 값 중 가장 오른쪽(최하위 비트) 1에 1더하기
		index += (index & -index);
	}
}
```  

## 쿼리 수행(구간 합)
- 구간 합을 구하는 메서드는 아래와 같다.

```java
/**
 * 데이터 배열의 인덱스 start-end 구간의 합을 구한다.
 * @param start
 * @param end
 * @return
 */
public long sum(int start, int end) {
	return this.sum(end) - this.sum(start - 1);
}

/**
 * 1~index 까지의 합
 * @param index
 * @return
 */
public long sum(int index) {
	long sum = 0;

	while(index > 0) {
		sum += this.fenwickTree[index];
		// index 의 2진 값 중 가장 오른쪽(최하위 비트) 1에 1 빼기
		index -= (index & -index);
	}

	return sum;
}
```  

- 5~8의 구간합을 구하는 호출은 아래와 같다.
	- 펜윅 트리에서 5~8의 구간합은 (1~8의 합) - (1~4의 합) 으로 계산한다.
	
```java
this.update(5, 8);
```  

- 호출에 따른 펜윅 트리에서 사용하는 노드는 아래와 같다.
	- `(36)[8] - (10)[4] = 26`

![그림6]({{site.baseurl}}/img/algorithm/concept-fenwicktree-6.png)

-  8~15의 구간의 합을 구할 때 펜윅 트리에서 사용하는 노드는 아래와 같다.
	- 1~15합 - 1~7합 을 수행해서 8~15 구간의 합을 구한다.
	- 1~15합의 방문 노드 인덱스 : `15->14->12->8`
	- 1~7합의 방문 노드 인덱스 : `7->6->4`
	
![그림7]({{site.baseurl}}/img/algorithm/concept-fenwicktree-7.png)

- 펜윅 트리에서 구간 합을 구할 때 방문하는 노드를 비트로 표현하면 아래와 같다.
	- 1~15합 : `1111->1110->1100->1000`
	- 1~7합 : `111->110->100`

- 펜윅 트리 쿼리 수행(구간 합)에서 다음 인덱스의 계산은 `인덱스의 2진 값에서 가장 오른쪽에 위치한 1(가장 하위 비트)에 1을 뺀다.` 이다.
	- `111-1=110`
	- `110-10=100`
	
- `sum` 메서드에서 `index -= (index & -index);` 를 다시 표현하면 아래와 같다.
	- `index - (index & -index) = nextIndex`
	- `1100 - 100 = 1000`
	- 1100의 음수 표현은 XOR(1100) 한 후에 1을 더해주면 된다.

- 전체 소스 코드

```java
public class Main {
    private long[] fenwickTree;
    private int[] dataArray;
    private int dataCount;
    private int treeSize;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.dataArray = new int[]{
                1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16
        };
        this.dataCount = this.dataArray.length;
        this.treeSize = this.dataCount + 1;
        this.fenwickTree = new long[this.treeSize];

        for(int i = 0; i < this.dataCount; i++) {
            this.update(i + 1, this.dataArray[i]);
        }

        System.out.println(this.sum(1, 16));
        System.out.println(this.sum(1, 12));
        System.out.println(this.sum(2, 3));
        System.out.println(this.sum(2, 5));
        System.out.println(this.sum(10, 10));
        System.out.println();
        this.updateValue(2, 4);
        System.out.println(this.sum(1, 16));
        System.out.println(this.sum(1, 12));
        System.out.println(this.sum(2, 3));
        System.out.println(this.sum(2, 5));
        System.out.println(this.sum(10, 10));
    }

    /**
     * 데이터 배열과 펜윅 트리 index 에 해당하는 값 변경
     * @param index
     * @param value
     */
    public void updateValue(int index, int value) {
        // 데이터 배열이 0번째 부터 시작하고 펜윅 트리 배열은 1번째 부터 시작하기 때문에
        int diff = value - this.dataArray[index - 1];
        this.dataArray[index - 1] = value;
        this.update(index, diff);
    }

    /**
     * 펜윅 트리에서 index 에 해당하는 값 diff 만큼 변경
     * @param index
     * @param diff
     */
    public void update(int index, int diff) {

        while(index < this.treeSize) {
            this.fenwickTree[index] += diff;
            // index 의 2진 값 중 가장 오른쪽(최하위 비트) 1에 1더하기
            index += (index & -index);
        }
    }

    /**
     * 데이터 배열의 인덱스 start-end 구간의 합을 구한다.
     * @param start
     * @param end
     * @return
     */
    public long sum(int start, int end) {
        return this.sum(end) - this.sum(start - 1);
    }

    /**
     * 1~index 까지의 합
     * @param index
     * @return
     */
    public long sum(int index) {
        long sum = 0;

        while(index > 0) {
            sum += this.fenwickTree[index];
            // index 의 2진 값 중 가장 오른쪽(최하위 비트) 1에 1 빼기
            index -= (index & -index);
        }

        return sum;
    }
}
```

---
## Reference
[펜윅 트리 (바이너리 인덱스 트리)](https://www.acmicpc.net/blog/view/21)  
[펜윅 트리(Fenwick Tree, Binary Indexed Tree, BIT)](https://www.crocus.co.kr/666)  
