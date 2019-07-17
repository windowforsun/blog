--- 
layout: single
classes: wide
title: "[Algorithm 개념] Persistent Segment Tree"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: 'Persistent Segment Tree 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Segment Tree
  - Persistent Segment Tree
use_math : true
---  

## Persistent Segment Tree 란
- Persistent Segment Tree 는 가상으로 N 개의 Segment Tree 를 두는데 공간 복잡도는 $O(\log_2 N)$ 인 Segment Tree 를 만드는 방법이다.
- Persistent Segment Tree 는 시간 복잡도
	- 트리 생성 : $O(N \log_2 N)$
	- 쿼리 수행 : $O(\log_2 N)$
	- 값 변경 : $O(\log_2 N)$
	
## Persistent Segment Tree 의 원리
- 세그먼트 트리는 1차원 배열을 통해 트리 구조를 구성했다면, Persistent Segment Tree 는 노드에 좌, 우측 
- 특정 값을 변경할 때, 트리구조에서 값 변경이 필요한 노드만 새로만들고 다른 노드는 원래의 트리와 공유한다.

## 예시
- Persistent Segment Tree 도 세그먼트 트리와 동일하게 구간 합을 예시로 든다.
- 아래와 같은 구간합을 구해야하는 배열 D가 있다.

index|0|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15
---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---
value|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16

- D배열을  Persistent Segment Tree 로 나타내면 아래와 같다.

![그림1]({{site.baseurl}}/img/algorithm/concept-fenwicktree-1.png)

- 위에서 언급해던 것과 같이 Persistent Segment Tree 는 위 트리가 배열로 구성된것이 아니라 각 노드의 오른쪽 노드, 왼쪽 노드에 노드의 레퍼런스를 넣어 구성한다.
	- `rootNode.right = rightNode`
	- `rootNode.left = leftNode`

```java
class Node {
    public int value;
    public Node left;
    public Node right;

    public Node(int value, Node left, Node right) {
        this.value = value;
        this.left = left;
        this.right = right;
    }
}
```  

### 트리 생성

- 트리를 생성하는 메서드는 아래와 같다.

```java
public void init(Node node, int start, int end) {
    if(start == end) {
        node.value = this.dataArray[start];
    } else {
        int mid = (start + end) / 2;
        node.left = new Node(0, null, null);
        node.right = new Node(0, null, null);
        this.init(node.left, start, mid);
        this.init(node.right, mid + 1, end);

        node.value = node.left.value + node.right.value;
    }
}
```  

- D 배열을 트리로 구성할때는 아래와 같이 호출 하면 된다.
	- 새로운 트리를 만들기 전에 `root` 노드를 만들어주고 이를 메서드의 인자값으로 전달해야 한다.

```java
Node root1 = new Node(0, null, null);
this.init(root1, 0, this.dataArray.length - 1);
```  

## 값 변경
- 값을 변경하는 메서드는 아래와 같다.

```java
public void update(Node prev, Node node, int start, int end, int index, int value) {
    if(index < start || end < index) {
        return;
    }

    if(start == end) {
        node.value = value;
    } else {
        int mid = (start + end) / 2;

        if(index <= mid) {
            node.right = prev.right;
            node.left = new Node(0, null, null);
            update(prev.left, node.left, start, mid, index, value);
        } else {
            node.left = prev.left;
            node.right = new Node(0, null, null);
            update(prev.right, node.right, mid + 1, end, index, value);
        }

        node.value = node.left.value + node.right.value;
    }
}
```  

- 어느 하나의 값을 변경하게 되면 `root` 는 항상 변하기 때문에 새로운 `root`를 메서드의 인자값으로 전달해준다.
- 인덱스 0의 값을 2로 변경하는 호출을 아래와 같다.

```java
Node root2 = new Node(0, null, null);
this.update(root1, root2, 0, this.dataArray.length, 0, 2);
```  

- 0번째 인덱스(값=1)을 2로 변경할때 변경되는 노드들은 아래와 같다.

![그림2]({{site.baseurl}}/img/algorithm/concept-persistentsegmenttree-1.png)

- 세그먼트 트리라면 해당 노드들의 값을 직접 변경하지만, Persistent Segment Tree 는 변경이 필요한 노드만을 새로만들어 값 변경을 수행한다.

![그림3]({{site.baseurl}}/img/algorithm/concept-persistentsegmenttree-2.png)

- 초기 상태인 `root1` 과 새로운 노드가 추가된 상태인 `root2` 를 인자 값으로 넣어주면 `root2` 에는 값이 실제로 변경되는 노드만 새로 만들어 연결하고 기존의 노드는 `root1` 의 노드들을 그래도 사용한다.

## 쿼리 수행(구간 합)
- 구간 합을 구하는 메서드는 아래와 같다.

```java
public int query(Node node, int start, int end, int queryStart, int queryEnd) {
    int result = 0;

    if((end >= queryStart || queryEnd >= start) && node != null) {
        if(queryStart <= start && end <= queryEnd) {
            result = node.value;
        } else {
            int mid = (start + end) / 2;
            result += this.query(node.left, start, mid, queryStart, queryEnd);
            result += this.query(node.right, mid + 1, end, queryStart, queryEnd);
        }
    }

    return result;
}
```  

- `root1` 에서 0~5 구간의 합을 구할 때 메서드 호출은 아래와 같고 합은 21이 된다.

```java
this.query(root1, 0, this.dataArray.length - 1 , 0, 5)
```  

![그림4]({{site.baseurl}}/img/algorithm/concept-persistentsegmenttree-3.png)

- `root2` 에서 0~5 구간의 합을 구할 때 메서드 호출은아래와 같고 합은 22가 된다.

```java
this.query(root2, 0, this.dataArray.length - 1 , 0, 5)
```  

![그림4]({{site.baseurl}}/img/algorithm/concept-persistentsegmenttree-4.png)

## 소스코드

```java
public class Main {
    private int[] dataArray;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.dataArray = new int[]{
                1, 2, 3, 4, 5, 6, 7, 8, 9, 10
        };

        Node root1 = new Node(0, null, null);
        this.init(root1, 0, this.dataArray.length - 1);
        System.out.println(this.query(root1, 0, this.dataArray.length - 1 , 0, 9));
        System.out.println(this.query(root1, 0, this.dataArray.length - 1 , 0, 3));
        System.out.println(this.query(root1, 0, this.dataArray.length - 1 , 1, 3));

        System.out.println();

        Node root2 = new Node(0, null, null);
        this.update(root1, root2, 0, this.dataArray.length, 0, 2);
        System.out.println(this.query(root2, 0, this.dataArray.length - 1 , 0, 9));
        System.out.println(this.query(root2, 0, this.dataArray.length - 1 , 0, 3));
        System.out.println(this.query(root2, 0, this.dataArray.length - 1 , 1, 3));

        System.out.println();

        System.out.println(this.query(root1, 0, this.dataArray.length - 1 , 0, 9));
        System.out.println(this.query(root1, 0, this.dataArray.length - 1 , 0, 3));
        System.out.println(this.query(root1, 0, this.dataArray.length - 1 , 1, 3));
    }

    public void init(Node node, int start, int end) {
        if(start == end) {
            node.value = this.dataArray[start];
        } else {
            int mid = (start + end) / 2;
            node.left = new Node(0, null, null);
            node.right = new Node(0, null, null);
            this.init(node.left, start, mid);
            this.init(node.right, mid + 1, end);

            node.value = node.left.value + node.right.value;
        }
    }

    public void update(Node prev, Node node, int start, int end, int index, int value) {
        if(index < start || end < index) {
            return;
        }

        if(start == end) {
            node.value = value;
        } else {
            int mid = (start + end) / 2;

            if(index <= mid) {
                node.right = prev.right;
                node.left = new Node(0, null, null);
                update(prev.left, node.left, start, mid, index, value);
            } else {
                node.left = prev.left;
                node.right = new Node(0, null, null);
                update(prev.right, node.right, mid + 1, end, index, value);
            }

            node.value = node.left.value + node.right.value;
        }
    }

    public int query(Node node, int start, int end, int queryStart, int queryEnd) {
        int result = 0;

        if((end >= queryStart || queryEnd >= start) && node != null) {
            if(queryStart <= start && end <= queryEnd) {
                result = node.value;
            } else {
                int mid = (start + end) / 2;
                result += this.query(node.left, start, mid, queryStart, queryEnd);
                result += this.query(node.right, mid + 1, end, queryStart, queryEnd);
            }
        }

        return result;
    }

    class Node {
        public int value;
        public Node left;
        public Node right;

        public Node(int value, Node left, Node right) {
            this.value = value;
            this.left = left;
            this.right = right;
        }
    }
}
```  

---
## Reference
[Persistent Segment Tree | Set 1 (Introduction)](https://www.geeksforgeeks.org/persistent-segment-tree-set-1-introduction/)  
[Persistent segment trees – Explained with spoj problems](https://blog.anudeep2011.com/persistent-segment-trees-explained-with-spoj-problems/)  
[Persistent Segment Tree](https://blog.myungwoo.kr/100)  
[배열 기반 Persistent Segment Tree](https://junis3.tistory.com/8)  
[Persistent Segment Tree](https://softgoorm.tistory.com/12)  
[Persistent Segment Tree](https://hongjun7.tistory.com/64)  
