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
- 스택에 존재하는 노드는 검은색, 삭제된 노드는 붉은색으로 표시한다.
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

    public Main() {
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
            StringTokenizer token = new StringTokenizer(reader.readLine());
            int start, end;

            this.nodeCount = Integer.parseInt(token.nextToken());
            this.edgeCount = Integer.parseInt(token.nextToken());
            this.stack = new LinkedList<>();
            this.isVisited = new boolean[this.nodeCount + 1];
            this.sccArray = new ArrayList[this.nodeCount];
            this.graph = new ArrayList[this.nodeCount + 1];
            this.reverseGraph = new ArrayList[this.nodeCount + 1];

            for(int i = 0; i <= this.nodeCount; i++) {
                this.graph[i] = new ArrayList();
                this.reverseGraph[i] = new ArrayList();
            }

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                start = Integer.parseInt(token.nextToken());
                end = Integer.parseInt(token.nextToken());

                this.graph[start].add(end);
                this.reverseGraph[end].add(start);
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        int currentNode, count = 0;

        this.result = new StringBuilder();

        // 정방향 그래프 DFS 탐색 하며 스택에 노드 넣어주기
        for(int i = 1; i <= this.nodeCount; i++) {
            if(!this.isVisited[i]) {
                this.dfs(i);
            }
        }

        this.isVisited = new boolean[this.nodeCount + 1];

        // 스택에서 노드를 빼면서 역방향 그래프 DFS 탐색
        while(!this.stack.isEmpty()) {
            currentNode = this.stack.removeFirst();

            // 스택에서 빼내어진 노드를 DFS 탐색 수행 할때마다 SCC 집합이 하나씩 생성됨
            if(!this.isVisited[currentNode]) {
                this.sccArray[count] = new ArrayList();
                this.reverseDfs(currentNode, count);
                count++;
            }

        }
        
        // 구성된 SCC 집합 출력
        for(int i = 0; i < count; i++) {
        	System.out.println(Arrays.toString(this.sccArray[i].toArray()));
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

        // 현재 node 에서 모든 노드를 방문하고 나면 스택에 넣음
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

        // SCC 집합 개수를 기준으로 해당되는 node 추가
        this.sccArray[count].add(node);
    }
}
```  

## Tarjan's Algorithm
- 타잔 알고리즘은 한번의 DFS 연산과 하나의 방향 그래프, 스택을 이용해서 SCC 집합을 구할 수 있는 알고리즘이다.
- 알고리즘 순서
	1. DFS 는 가장 인접한 노드들 중 가장 적은 DFS 순서번호를 반환한다.
	1. 주어진 방향 그래프의 임의의 정점에서 부터 DFS 를 수행한다. 
	1. DFS 를 방문하는 순서에 맞게 각 노드에 번호를 부여하고, 스택에 넣어준다.
		- DFS 방문 순서번호는 해당 노드를 방문 했는지에 대한 쓰임과 만들어야 하는 SCC 노드들 중 스택의 최대하단의 노드인지 판별할 때 사용 된다.
	1. 현재 노드의 DFS 순서번호와 인접 노드의 최소 순서번호가 같다면 스택에서 자신보다 위에 있는(나중에 들어간) 노드를 빼서 SCC 집합으로 만든다.
		- SCC 집합을 만들때, SCC 집합으로 포함되는 각 노드들에 SCC 집합에 대한 연산이 모두 끝났다는 finished 와 같은 불린 배열에 표시한다.
		
### 예시
- 적은 노드 번호가 방문에서 더 높은 우선순위를 갖는다.
- 방문한 노드는 붉은색으로 표시하고 DFS 순서번호는 괄호로 표시한다.
- 스택에 존재하는 노드는 검은색, 삭제된 노드는 붉은색으로 표시한다.
- 노드 탐색은 1번 노드 부터 진행한다.
- 아래와 같은 연결 그래프가 존재한다.

![tarjan]({{site.baseurl}}/img/algorithm/concept-scc-tarjan-1.png)

- 1번 노드에서 방문 가능한 노드를 방문하면 아래와 같다. (1->7->1->6->1)
	- 7->1, 6->1 탐색의 경우 7번, 6번 DFS 탐색에서 1번 노드를 다시 탐색하는 것이 아니다.
	- 1번 노드의 순서 번호가 이미 부여 되었기 때문에 7번, 6번 DFS 탐색을 빠져나와 1번 DFS 탐색으로 다시 돌아가는 것을 뜻한다.

![tarjan]({{site.baseurl}}/img/algorithm/concept-scc-tarjan-2.png)

- 1번 노드에서 더이상 방문할 노드가 없기 때문에 스택에서 1번 위에 쌓인 노드를 모두 빼내어 SCC 집합으로 묶어 준다.

![tarjan]({{site.baseurl}}/img/algorithm/concept-scc-tarjan-3.png)

- 다음 노드인 2번 노드를 방문 한다.
- 2번 노드는 더이상 방문할 수 있는 노드가 없기 때문에 2번 위에 쌓인 노드들을 빼내어 SCC 집합으로 만든다.
	- 1번 노드의 경우 이미 SCC 집합에 구성되었기 때문에 방문할 수 없다.

![tarjan]({{site.baseurl}}/img/algorithm/concept-scc-tarjan-4.png)

- 다음 노드인 3번 노드를 방문한다.
- 3번 노드에서 방문 가능한 노드를 모두 방문하면 아래와 같다. (3->4->5->3)
	- 5->3 탐색의 경우도 5번 DFS 탐색에서 3번 노드의 순서 번호가 이미 부여 되었기 때문에 DFS 탐색을 빠져나와 3번 DFS 탐색으로 돌아가는 것이다.

![tarjan]({{site.baseurl}}/img/algorithm/concept-scc-tarjan-5.png)

- 3번 노드에서 더이상 방문할 노드가 없기 때문에 스택에서 3번 위에 쌓이 노드를 모두 빼내어 SCC 집합으로 묶어 준다.

![tarjan]({{site.baseurl}}/img/algorithm/concept-scc-tarjan-6.png)

### 소스코드

```java
public class Main {
    // 노드 수
    private int nodeCount;
    // 엣지 수
    private int edgeCount;
    // 방향 그래프 정보
    private ArrayList<Integer>[] graph;
    // 스택
    private LinkedList<Integer> stack;
    // DFS 탐색 순서번호
    private int[] dfsCountArray;
    // SCC 탐색이 완료된 노드인지(SCC 집합에 이미 구성원인)
    private boolean[] finished;
    // 현재 구성된 SCC 집합 배열
    private ArrayList<Integer>[] sccGroupList;
    // DFS 탐색 할때마다 증가하는 DFS 순서번호 변수
    private int dfsCount;
    // 현재 구성된 SCC 집합 개수
    private int sccGroupCount;

    public Main() {
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
            this.graph = new ArrayList[this.nodeCount + 1];
            this.stack = new LinkedList<>();
            this.dfsCountArray = new int[this.nodeCount + 1];
            this.finished = new boolean[this.nodeCount + 1];
            this.sccGroupList = new ArrayList[this.nodeCount + 1];
            this.result = new StringBuilder();
            this.dfsCount = 1;

            for(int i = 1; i <= this.nodeCount; i++) {
                this.graph[i] = new ArrayList<>();
            }

            for(int i = 0; i < this.edgeCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");

                this.graph[Integer.parseInt(token.nextToken())].add(Integer.parseInt(token.nextToken()));
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 1번 노드 부터 순차적으로 DFS 순서 번호가 부여되지 않았으면 DFS 탐색을 수행
        for(int i = 1; i <= this.nodeCount; i++) {
            if(this.dfsCountArray[i] == 0) {
                this.dfs(i);
            }
        }
        
        // 구성된 SCC 집합 출력
        for(int i = 0; i < this.sccGroupCount; i++) {
        	System.out.println(Arrays.toString(this.sccGroupList[i].toArray()));
        }
    }

    public int dfs(int node) {
        // 현재 노드의 DFS 탐색 순서번호
        this.dfsCountArray[node] = this.dfsCount++;
        // DFS 탐색하는 순서대로 스택에 넣음
        this.stack.addFirst(node);

        // 현재 노드 및 인접 노드 중 최소 순서 번호
        int dfsCountMin = this.dfsCountArray[node];
        // 현재 노드에서 인접한 노드 리스트
        ArrayList<Integer> adjList = this.graph[node];

        for(int next : adjList) {
            if(this.dfsCountArray[next] == 0) {
                // 아직 방문하지 않은 노드
                dfsCountMin = Integer.min(dfsCountMin, this.dfs(next));
            } else if(!this.finished[next]) {
                // 방문 하였지만, SCC 집합으로 구성되지 않은 노드
                dfsCountMin = Integer.min(dfsCountMin, this.dfsCountArray[next]);
            }
        }

        // 현재 노드가 인접한 노드들 중 가장 순서 번호가 빠르다면 SCC 추출
        if(dfsCountMin == this.dfsCountArray[node]) {
            int sccNode;
            ArrayList<Integer> sccGroup = new ArrayList<>();

            // 스택의 맨위에서부터 하나씩 빼면서 자신이 나올때까지 반복
            do {
                sccNode = this.stack.removeFirst();
                sccGroup.add(sccNode);
                this.finished[sccNode] = true;

            } while(sccNode != node);
            
            this.sccGroupList[this.sccGroupCount] = sccGroup;
            this.sccGroupCount++;
        }

        return dfsCountMin;
    }
}
```  

## SCC 활용
- SCC 의 개념과 특징, 구현방법에 대해 알아보았는데 이런 SCC 를 어디에 활용할 수 있을까 ?

![scc 활용]({{site.baseurl}}/img/algorithm/concept-scc-application-1.png)

- 위의 그림을 보면 SCC 집합 단위로 다시 새로운 방향 그래프가 만들어지는 것을 확인 할 수 있다.
- 그리고 SCC 집합으로 이루어진 그래프는 DAG(Directed Acyclic Graph) 이므로 사이클이 존재하지 않는다.
- SCC 집합 구성원들은 사이클로 구성되어 있기 때문에 하나의 SCC 집합의 구성원이 다른 SCC 집합의 구성원으로 갈 수 있다는 것을 알수 있다.
- 이는 SCC 단위로 위상 정렬을 할 수 있다는 의미가 된다.
	- SCC 단위로 그래프를 만들기 전에는 노드간의 사이클이 존재해 위상정렬이 불가능 했지만, SCC 단위로 만들어진 그래프에서는 위상 정렬이 가능하다.


---
## Reference
[SCC(Strongly Connected Component)](https://jason9319.tistory.com/98)  
[강한 연결 요소(Strongly Connected Component)](https://kks227.blog.me/220802519976)  
[SCC-코사라주 알고리즘(Kosaraju's Algorithm)](https://wondy1128.tistory.com/130)  
[강결합 컴포넌트 (Strongly Connected Component)](https://seungkwan.tistory.com/7)  
[SCC(Strongly Connected Component)](https://www.crocus.co.kr/950)  
