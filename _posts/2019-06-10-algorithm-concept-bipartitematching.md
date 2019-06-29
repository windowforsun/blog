--- 
layout: single
classes: wide
title: "[Algorithm 개념] 이분 매칭(Bipartite Matching) 알고리즘"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '이분 그래프에서 이분 매칭으로 최대 매칭 개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Bipartite Matching
  - Hopcroft-Karp
  - Bipartite Graph
use_math : true
---  

## 이분 매칭 이란
- 그래프의 정점들을 두개의 그룹으로 나누었을 때, 모든 간선의 양 끝 정점이 서로 다른 그룹에 속하는 형태를 이분 그래프(Bipartite Graph)라고 한다.
- 두 그룹을 A 와 B라 하고, 간선의 방향이 모두 A->B 일때 최대 유량을 구하는 문제를 이분 매칭이라고 한다.
- A 그룹에서 B 그룹을 선택할 수 있는 노드는 하나만 선택 가능하다.
- 이분 매칭은 이분 그래프에서 매칭의 최대 개수와 동일하다.
- [Network Flow 알고리즘]({{site.baseurl}}{% link _posts/2019-06-07-algorithm-concept-networkflow.md %}) 을 이용해서도 정답을 도출해 낼 수 있다.
- 이분 매칭에서 Hopcroft-Karp Algorithm 알고리즘을 사용하면 $O(E \sqrt{V})$ 라는 빠른 시간으로 문제 해결이 가능하다.
- DFS 를 사용 할 경우 $O(VE)$ 의 시간 복잡도에 해결할 수 있다. 

![이분 매칭 1]({{site.baseurl}}/img/algorithm/concept-bipartitematching-1.png)


## DFS 를 이용한 이분 매칭 알고리즘
- 1~5번 노드를 A 그룹, 5~10을 B그룹으로 정한다.
- 1번 노드 부터 매칭이 시작된다.
- 1번 노드에서 DFS 탐색을 해서 B그룹 중 매칭 6번 노드가 매칭이 가능하므로 매칭 시킨다.

![이분 매칭 2]({{site.baseurl}}/img/algorithm/concept-bipartitematching-2.png)

- 2번 노드에서 DFS 탐색을 하면 B그룹 중 6번 노드와 매칭이 가능하다.
- 6번 노드는 1번노드와 매칭이 되어 있으므로, 1번 노드에서 6번 노드를 제외하고 DFS 탐색을 해서 B그룹 중 7번 노드와 매칭이 가능하므로 매칭 시킨다.
- 1번 노드는 7번과 매칭이 되었으므로, 2번은 6번과 매칭 시킨다.

![이분 매칭 3]({{site.baseurl}}/img/algorithm/concept-bipartitematching-3.png)

- 3번 노드에서 DFS 탐색을 하면 B그룹중 6번 노드와 매칭이 가능하다.
- 6번 노드와 매칭된 2번 노드는 7번을 제외하고 8번과 매칭가능하므로 매칭 시킨다.
- 2번 노드가 8번과 매칭 되었으므로, 3번은 6번과 매칭 시킨다.

![이분 매칭 4]({{site.baseurl}}/img/algorithm/concept-bipartitematching-4.png)

- 4번 노드에서 DFS 탐색을 하면 B그룹 중 9번과 매칭이 가능하므로 매칭 시킨다.

![이분 매칭 5]({{site.baseurl}}/img/algorithm/concept-bipartitematching-5.png)

- 5번 노드에서 DFS 탐색을 하면 B그룹 중 8번과 매칭 가능하다.
- 8번 노드와 매칭된 2번 노드는 8번을 제외한 B그룹 노드 중에서 6번과 매칭 가능하다.
- 6번 노드와 매칭된 3번 노드는 6번을 제외한 B그룹 노드 중에서 10번과 매칭 가능하므로 매칭 시킨다.
- 3번 노드가 10번 노드와 매칭 되었으므로, 2번 노드는 6번노드 그리고 5번 노드는 8번 노드와 매칭 시킨다.
- 최종적으로 아래와 같이 매칭되고, 최대 매칭 개수는 5가 된다.

![이분 매칭 6]({{site.baseurl}}/img/algorithm/concept-bipartitematching-6.png)

- 소스코드

```java
public class Main {
    private int result;
    private ArrayList<Integer>[] isAdjListArray;
    private int[] matched;
    private boolean[] isVisited;
    private int groupACount;
    private int groupBCount;
    private int max;

    public static void main(String[] args) {
        new Main();
    }

    public Main() {
        this.groupACount = 5;
        this.groupBCount = 5;
        this.max = this.groupACount + this.groupBCount;

        this.isAdjListArray = new ArrayList[this.groupACount + 1];
        this.matched = new int[this.max + 1];

        for(int i = 0; i <= this.groupACount; i++) {
            this.isAdjListArray[i] = new ArrayList<>();
        }
        
        this.isAdjListArray[1].add(6);
        this.isAdjListArray[1].add(7);
        this.isAdjListArray[2].add(6);
        this.isAdjListArray[2].add(8);
        this.isAdjListArray[3].add(6);
        this.isAdjListArray[3].add(8);
        this.isAdjListArray[3].add(10);
        this.isAdjListArray[4].add(9);
        this.isAdjListArray[4].add(10);
        this.isAdjListArray[5].add(8);

        this.solution();
        this.output();
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // A 그룹 노드를 한번씩 DFS 탐색을 통해 매칭 시킨다.
        for(int i = 1; i <= this.groupACount; i++) {
            // 방문 체크 배열 초기화
            this.isVisited = new boolean[this.max + 1];

            // 매칭에 성공하면 최대 매칭 개수 증가
            if(this.dfs(i)) {
                this.result++;
            }
        }
    }

    public boolean dfs(int start) {
        boolean result = false;

        // 방문하지 않았을 경우에만 
        if(!this.isVisited[start]) {
            this.isVisited[start] = true;

            ArrayList<Integer> adjList = this.isAdjListArray[start];
            int adj, adjListSize = adjList.size();

            // 매칭 가능한 B그룹 노드 탐색
            for(int i = 0; i < adjListSize; i++) {
                adj = adjList.get(i);

                // B그룹 노드가 매칭 되지 않거나
                // 해당 B그룹 노드와 매칭된 A그룹 노드가 매칭 가능한 다른 B그룹 노드가 있을 경우
                if(this.matched[adj] == 0 || this.dfs(this.matched[adj])) {
                    // 매칭 시킴
                    this.matched[adj] = start;
                    result = true;
                    break;
                }
            }
        }

        return result;
    }
}
```


## Hopcroft-Karp Algorithm

---
## Reference
[이분 매칭(Bipartite Matching)](https://jason9319.tistory.com/149)  
[호프크로프트 카프 알고리즘 (Hopcroft-Karp Algorithm)](http://blog.naver.com/PostView.nhn?blogId=kks227&logNo=220816033373)  
