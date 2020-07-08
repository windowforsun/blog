--- 
layout: single
classes: wide
title: "[풀이] 백준 1197 최소 스패닝 트리"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '그래프의 가중치의 합이 최소인 최소 스패닝 트리를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - MST
  - Disjoint Set
---  

# 문제
- 그래프가 주어졌을 때, 최소 스패닝 트리(MST) 를 구하라.
- MST 랑 주어진 모든 그래프의 정점을 연결하는 부분 그래프 중 가중치의 합이 최소인 트리를 말한다.

## 입력
첫째 줄에 정점의 개수 V(1 ≤ V ≤ 10,000)와 간선의 개수 E(1 ≤ E ≤ 100,000)가 주어진다. 다음 E개의 줄에는 각 간선에 대한 정보를 나타내는 세 정수 A, B, C가 주어진다. 이는 A번 정점과 B번 정점이 가중치 C인 간선으로 연결되어 있다는 의미이다. C는 음수일 수도 있으며, 절댓값이 1,000,000을 넘지 않는다.  

최소 스패닝 트리의 가중치가 -2147483648보다 크거나 같고, 2147483647보다 작거나 같은 데이터만 입력으로 주어진다.

## 출력
첫째 줄에 최소 스패닝 트리의 가중치를 출력한다.

## 예제 입력

```
3 3
1 2 1
2 3 2
1 3 3
```  

## 예제 출력

```
3
```  

## 풀이
- MST 를 구하기 위해 Disjoint Set 을 사용한다.
	- Disjoint Set 이란 중복되지 않는 부분 집합들로 나눠진 원소들에 대한 정보를 저장하고 조직하는 자료구조로 Union Find 를 통해 구현 가능하다.
- 이 문제의 정답을 구하는 방법을 그림으로 표현하면 아래와 같다.
	- 3 개의 node = (1, 2, 3) 와 3개의 edge = (1, 2, 3) 와 그에 해당 하는 가중치가 있다.
	
		![초기 그래프]({{site.baseurl}}/img/algorithm/bj-1197-1.png)
		
	- 3 개의 edge 의 가중치 중에 가장 작은 1을 선택해서 node1 과 node2 를 연결한다. 가중치의 합 = 1
		
		![초기 그래프 연결 1 2]({{site.baseurl}}/img/algorithm/bj-1197-2.png)
		
	- 남은 2개 edge 의 가중치 중 가장 적은 2를 선택해서 node2와 node3 을 연결한다. 가중치의 합 = 3
		
		![초기 그래프 연결 1 2 3]({{site.baseurl}}/img/algorithm/bj-1197-3.png)
		
- 이를 코드로 구현하기 위해서는 몇가지 주의 사항이 있다.
	1. 가중치가 적은 순으로 edge 를 연결해야 한다.
	1. 가중치가 가장 적은 순으로 edge 를 연결 하지만, Cycle 이 생겨서는 안된다. (트리 자료구조에는 Cycle 이 존재해서는 안된다.)
		- Cycle 검사를 위해 특정 node 의 root node 를 저장해서 Cycle 여부를 검사 한다.
	1. edge 가중치의 최대 값은 1000000 이고, node 의 최대 개수는 10000 개 이므로 모든 node 를 연결해야하는 edge 의 수는 9999 이된다. 즉 가중치의 최대 값은 9999000000 이 될 수 있으므로 long 형을 사용한다.

```java
public class Main {
    // 출력 결과
    private long result;
    // node 의 개수
    private int nodeCount;
    // edge 의 개수
    private int edgeCount;
    // 입력 받은 EdgeObj 배열
    private EdgeObj[] edgeArray;
    // index(node번호) 에 해당하는 root node 번호 저장 배열
    private int[] rootIndexArray;
    // index(node 번호) 의 rank 값 (path compression 에서 사용)
    private int[] rankArray;

    public Main() {
        this.result = 0;
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.edgeCount = Integer.parseInt(token.nextToken());
            this.edgeArray = new EdgeObj[this.edgeCount];

            int node1, node2, cost;

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");

                node1 = Integer.parseInt(token.nextToken());
                node2 = Integer.parseInt(token.nextToken());
                cost = Integer.parseInt(token.nextToken());

                this.edgeArray[i] = new EdgeObj(node1, node2, cost);
            }

//            this.rankArray = new int[this.nodeCount + 1];
            this.rootIndexArray = new int[this.nodeCount + 1];
            for(int i = 1; i <= this.nodeCount; i++) {
                this.rootIndexArray[i] = i;
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // MST 를 만들려면 cost 가 가장 적은 순으로 탐색을 해야 하기 때문에 edge 의 cost 로 내림차순 정렬
        Arrays.sort(this.edgeArray, new Comparator<EdgeObj>() {
            @Override
            public int compare(EdgeObj o1, EdgeObj o2) {
                return o1.cost - o2.cost;
            }
        });

        // 현재 연결할 edge
        EdgeObj currentEdge;
        // 연결된 node 의 개수, 연결할 edge index
        int unionNodeCount = 1, edgeIndex = 0;
        // 연결 했는지
        boolean isUnion;

        // 연결된 node 의 개수가 입력 받은 node 수 보다 작을 때 까지
        while(unionNodeCount < this.nodeCount) {
            currentEdge = this.edgeArray[edgeIndex];

            isUnion = this.unionNode(currentEdge.node1, currentEdge.node2, currentEdge.cost);
//            isUnion = this.unionNodeWithCompression(currentEdge.node1, currentEdge.node2, currentEdge.cost);

            // union 메서드에서 연결을 했을 경우 연결된 node 개수 카운트
            if(isUnion) {
                unionNodeCount++;
            }

            edgeIndex++;
        }
    }

    // 매개변수 node 의 root node 를 반환
    public int findNodeRoot(int node) {
        // 현재 node 의 root node
        int root = this.rootIndexArray[node];

        // 현재 node 가 root node 가 아니라면
        if(node != root) {
            // root node 의 root node 계속 찾으며 가장 상위 root node 를 찾음
            root = this.findNodeRoot(root);

            // 가장 상위 root node 로 현재 node 의 root node 값 갱신
            this.rootIndexArray[node] = root;
        }

        return root;
    }

    // node1, node2 를 union 하고 cost 값을 더함 메서드 성공시 true, 실패시 false
    public boolean unionNode(int node1, int node2, int cost) {
        // node1 의 root node
        int node1Root = this.findNodeRoot(node1);
        // node2 의 root node
        int node2Root = this.findNodeRoot(node2);

        // 두 node 의 root node 가 같다면 이미 연결된 node 즉 Cycle 이 존재
        if(node1Root == node2Root) {
            return false;
        }

        // 두 node 가 연결이 되지 않았으면, 항상 node2 을 node1 의 root node 로 만듦
        this.rootIndexArray[node1Root] = node2Root;
        // cost 값 추가
        this.result += (long)cost;

        return true;
    }

    // edge 정보 클래스
    class EdgeObj {
        // edge 에 연결된 node1
        public int node1;
        // edge 에 연결된 node2
        public int node2;
        // edge 의 cost 값
        public int cost;

        public EdgeObj(int node1, int node2, int cost) {
            this.node1 = node1;
            this.node2 = node2;
            this.cost = cost;
        }
    }

    // unionNode 메서드와 수행 결과는 비슷하지만 path compression 이 추가 되었다.
    public boolean unionNodeWithCompression(int node1, int node2, int cost) {
        int node1Root = this.findNodeRoot(node1);
        int node2Root = this.findNodeRoot(node2);
        int bigRankRoot, smallRankRoot;

        if(node1Root == node2Root) {
            return false;
        }

        // 두 root node 중 rank 값이 큰 것과 작은 것을 구분한다.
        if(this.rankArray[node1Root] >= this.rankArray[node2Root]) {
            bigRankRoot = node1Root;
            smallRankRoot = node2Root;
        } else {
            bigRankRoot = node2Root;
            smallRankRoot = node1Root;
        }

        // rank 값이 작은 root node 를 rank 값이 큰 root node 로 설정한다.
        this.rootIndexArray[smallRankRoot] = bigRankRoot;
        this.result += (long)cost;

        // 두 root node 의 rank 값이 같다면 임의로 하나의 rank 값을 증가시켜 준다.
        if(this.rankArray[bigRankRoot] == this.rankArray[smallRankRoot]) {
            this.rankArray[bigRankRoot]++;
        }

        return true;
    }
}
```  

---
## Reference
[1197-최소 스패닝 트리](https://www.acmicpc.net/problem/1197)  
