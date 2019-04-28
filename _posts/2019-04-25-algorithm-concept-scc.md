--- 
layout: single
classes: wide
title: "[Algorithm 개념] SCC(Strongly Connected Component) 강한 연결 요소 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: 'SCC, Kosaraju 와 Tarjan Algorith 에 대해 알아보고 이를 구현해보자 '
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - SCC
  - Kosaraju's
  - Tarjan's
  - Spanning Tree
---  

## SCC(Strongly Connected Component) 란
- 방향성 그래프에서 아래 조건을 만족하는 정점들의 집합을 SCC 라고 한다.
	1. 두 정점 A, B 가 있을 때, A->B 로 가는 경로가 존재한다.
	1. A->B, B->A 의 결로가 동시에 존재 할 수 없다.
	1. A->B 라는 경로는 A->B 처럼 바로 연결 되어 있어도 되고, A->..->B 와 같이 연결되어 있어도 된다.
	1. SCC 집합 끼리는 사이클이 존재하지 않는다.

## SCC(Strongly Connected Component) 특징
- 같은 SCC 집합 내에 뽑은 입의의 A, B 점에서 A->B, B->A 의 경로(직/간접적)가 항상 존재한다.
- 서로 다른 SCC 집합끼리는 사이클이 존재하지 않는다.
- SCC 는 Maximal 한 성질을 가지고 있기 때문에, SCC 가 형성된다면 형성될 수 있는 가장 큰 집합으로 형성된다.

## Kosaraju's Algorithm
- 주어진 하나의 방향 그래프가 있다.
- 주어진 방향 그래프와 역방향을 가지는 역방향 그래프를 만든다.
- 정점을 담을 스택을 만든다.
	- 코사라주 알고리즘은 위상정렬을 이용하고, 방향 그래프와 역방향 그래프가 동일한 SCC 를 구성하는 것을 이용한다.
- 알고리즘 순서
	1. 주어진 방향 그래프에 임의의 정점부터 DFS 를 수행한다. DFS 가 끝나는 순서대로 스택에 넣어준다.
		1. DFS 를 수행한 후 아직 방문하지 않은 정점이 있는 경우, 해당 정점부터 다시 DFS 를 수행한다.
		1. 모든 정점을 방문하여 DFS 를 완료하게 되면 스택에 모든 정점이 담기게 된다.
	1. 스택의 top 부터 원소를 하나씩 빼내어, 역방향 그래프를 기준으로 DFS 를 수행한다.
		1. 탐색되는 모든 정점을 SCC 그룹으로 묶는다.
		1. 스택이 비어있을 때 까지 진행한다.
		1. 스택에서 빼낸 원소가 이미 방문 했다면 DFS 탐색은 하지 않는다.
		
### 예시
- 아래와 같은 주어진 방향 그래프가 있다.

![kosaraju 1]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-1.png)

- 방향이 반대인 역방향 그래프는 아래와 같다.

![kosaraju re 1]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-re-1.png)
		
- 주어진 그래프, 역방향 그래프, 스택을 준비한다.
- 예시에서는 작은 노드 번호에 높은 우선순위가 있다고 가정하겠다.
- 방문 한 노드는 노드에 빨간색, 방문으로 사용한 엣지는 순서대로 숫자를 기입한다.
- 1번 노드로 시작해서 DFS 를 수행한다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-2.png)

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-3.png)

- 6 번 노드에서 방문할 수 있는 노드는 1번 밖에 없기 때문에 6번을 스택에 넣고 1번으로 돌아간다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-4.png)

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-5.png)

- 7 번 노드에서 방문할 수 있는 노드는 1번 밖에 없기 때문에 7번을 스택에 넣고 1번으로 돌아간다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-6.png)

- 1번 노드에서 방문할 수 있는 노드는 모두 방문 했기 때문에 1번을 스택에 넣고 다음 방문하지 않은 노드를 방문한다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-7.png)

- 2번 노드에서 방문 할 수 있는 노드가 없기 때문에 2번을 스택에 넣고 다음 방문하지 않는 노드를 방문한다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-8.png)

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-9.png)

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-10.png)

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-11.png)

- 5번 노드에서 더 이상 방문 할 수 있는 노드가 없기 때문에 5번을 스택에 넣고 4번 노드로 돌아간다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-12.png)

- 4번 노드에서 더 이상 방문 할 노드가 없기 때문에 4번을 스택에 넣고 3번 노드로 돌아간다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-13.png)

- 3번 노드에서 더 이상 방문할 노드가 없기 때문에 3번을 스택에 놓고 DFS 를 종료한다.

![kosaraju dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-14.png)

- 모든 노드가 들어간 스택의 top 에서부터 하나씩 빼내며 역방향 그래프로 DFS 탐색을 수행한다.
- 빼낸 노드번호는 스택에서 빨간색으로 처리한다.
- 3번 노드를 스택에서 빼내고 3->5->5 순으로 DFS 탐색을 했다.
- 3번 노드에서는 더 이상 방문할 노드가 없기 때문에 3, 4, 5는 SCC 를 이루고 있다.

![kosaraju re dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-re-2.png)

- 스택에서 이미 방문한 5, 4번 노드를 빼낸다.
- 2번 노드를 스택에서 빼내고 방문한다. 2번 노드에서 더 이상 방문 할 노드가 없기 때문에 2번은 SCC 를 이루고 있다.

![kosaraju re dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-re-3.png)

- 1번 노드를 스택에서 빼내고 방문한다.
- 1->6->7 순으로 방문 후 1번 노드에서 더이상 방문할 노드가 없기 때문에 1, 6, 7는 SCC 를 이루고 있다.

![kosaraju re dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-re-4.png)

- 스택에 남은 6, 7 까지 빼낸다.
- 최종적으로 서로다른 3개의 SCC 집합으로 이루고 있음을 알아낼 수 있다.

![kosaraju re dfs]({{site.baseurl}}/img/algorithm/concept-scc-kosaraju-re-5.png)

- 코사라주 알고리즘은 DFS 를 2번 수행하기 때문에 DFS 와 같은 시간복잡도인 O(v+e) 가 된다.
- 하지만 DFS 를 2번 수행한다는 점, 스택과 추가적인 역방향 그래프를 표현해야 하기 때문에 시간적, 공간적으로 다른 알고리즘에 비해 살짝 좋지 않다.

### 소스코드

```java
public class Main {
	
    private int nodeCount;
    private int edgeCount;
    // 정방향 그래프
    private ArrayList[] graph;
    // 역방향 그래프
    private ArrayList[] reverseGraph;
    // SCC 집합의 개수에 따른 노드 번호 배열
    private ArrayList[] sccArray;
    // DFS 에서 사용하는 배열
    private LinkedList<Integer> stack;
    // 방문 플래그 배열
    private boolean[] isVisited;
    
    public static void main(String[] args) {    	
    	
    	// 그래프 정보 입력 및 graph, reverseGraph 구성하기
    	
        int currentNode, count = 0;
        ArrayList<Integer> scc;

        // 정방향 그래프 DFS
        for(int i = 1; i <= this.nodeCount; i++) {
            if(!this.isVisited[i]) {
                this.dfs(i);
            }
        }

        this.isVisited = new boolean[this.nodeCount + 1];

        // 스택에서 하나씩 빼면서 역방향 그래프 DFS
        while(!this.stack.isEmpty()) {
            currentNode = this.stack.removeFirst();

            // 스택에서 뺀 노드를 DFS 탐색 할때마다 SCC 집합이 생성됨
            if(!this.isVisited[currentNode]) {
                this.sccArray[count] = new ArrayList();
                this.reverseDfs(currentNode, count);
                count++;
            }
        }        
    }    
    
    public void dfs(int node) {
        this.isVisited[node] = true;

        ArrayList<Integer> adjNodeList = this.graph[node];

        for(int adjNode : adjNodeList) {
            if(!this.isVisited[adjNode]) {
                this.dfs(adjNode);
            }
        }

        // 현재 node 에서 방문할 수 있는 노드를 모두 방문하고나면 stack 에 현재 node 를 넣음
        this.stack.addFirst(node);
    }

    public void reverseDfs(int node, int count) {
        this.isVisited[node] = true;

        ArrayList<Integer> adjNodeList = this.reverseGraph[node];

        for(int adjNode : adjNodeList) {
            if(!this.isVisited[adjNode]) {
                this.reverseDfs(adjNode, count);
            }
        }

        // SCC 개수를 기준으로 해당되는 node 추가
        this.sccArray[count].add(node);
    }
}
```  

## Tarjan's Algorithm
- 타잔 알고리즘은 한번 


---
## Reference
[SCC(Strongly Connected Component)](https://jason9319.tistory.com/98)  
[강한 연결 요소(Strongly Connected Component)](https://kks227.blog.me/220802519976)  
[SCC-코사라주 알고리즘(Kosaraju's Algorithm)](https://wondy1128.tistory.com/130)  
[강결합 컴포넌트 (Strongly Connected Component)](https://seungkwan.tistory.com/7)  
[SCC(Strongly Connected Component)](https://www.crocus.co.kr/950)  
