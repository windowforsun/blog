--- 
layout: single
classes: wide
title: "[풀이] 백준 1135 뉴스 전하기"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '트리구조의 집단에서 뉴스를 전하는 최소 시간을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Graph
  - DFS
  - Greedy
---  

# 문제
- 트리 구조로 구성되있는 집단에서 Root 에서 부터 뉴스를 자식노드에게 전하게 된다.
- Root 노드를 제외한 각 노드는 한개의 부모를 가지고 있다.
- 각 부모 노드는 한번에 한개의 자식 노드에게 뉴스를 전할 수 있고 뉴스를 전달하는데 걸리는 시간은 1분이다.
- 트리를 구성하고 있는 모든 노드가 뉴스를 전달 받을 때 까지 걸리는 최소 시간을 구하라.

## 입력
첫째 줄에 직원의 수 N이 주어진다. 둘째 줄에는 0번 직원부터 그들의 상사의 번호가 주어진다. 반드시 0번 직원 (오민식)의 상사는 -1이고, -1은 상사가 없다는 뜻이다. N은 50보다 작거나 같은 자연수이다.

## 출력
첫째 줄에 모든 소식을 전하는데 걸리는 시간의 최솟값을 출력한다.

## 예제 입력

```
3
-1 0 0
```  

## 예제 출력

```
2
```  

## 풀이
- 각 부모는 한번에 하나의 자식 노드에게 밖에 전달하지 못하기 때문에 자식노드의 깊이가 깊은 노드에 먼저 뉴스를 알려야 최소 시간을 구할 수 있다.
- 즉 뉴스를 전하는 시간은 트리의 깊이와 트리의 자식수에 비례하게 된다.
- Root 노드에서 부터 자식노드가 자신의 자식들에게 전달하는데 걸리는 최대 시간을 구한다.
	- 뉴스를 전하는데 걸리는 시간 = 서브트리의 깊이 + 자식수
- 부모 노드는 자신의 자식노드들이 뉴스를 전하는데 걸린 최대 시간을 구하고나서 이를 내림차순으로 정렬한다.
	- 전달시간이 오래 걸리는 자식노드를 먼저 방문하기 위함
- 부모 노드는 정렬된 순으로 방문 카운트 값을 증가해주며 자신이 모든 자식노드에 뉴스를 전달하는 최대 시간을 구하고 이를 리턴해준다.

```java
public class Main {
    // 출력 결과
    private int result;
    // 총 트리 노드 수
    private int count;
    // 노등 대한 자식 정보 저장
    private ArrayList<int[]>[] childListArray;

    public Main() {
        this.input();
        this.solution(0);
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.count = Integer.parseInt(reader.readLine());
            this.childListArray = new ArrayList[this.count];

            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            short sh;

            for(int i = 0; i < this.count; i++) {
                this.childListArray[i] = new ArrayList<>(this.count);
            }

            for(int i = 0; i < this.count; i++) {
                sh = Short.parseShort(token.nextToken());

                if(sh == -1) {
                } else {
                    this.childListArray[sh].add(new int[]{i, 0});
                }
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    // DFS 탐색을 해당 노드의 깊이와 자식수에 따른 최대 전달 시간반환
    public int solution(int node) {
        // 현재 노드에서 자식노드의 깊이에 따른 전달 시간
        int maxDept = 0;

        // 현재 노드의 자식 노드들
        ArrayList<int[]> childList = this.childListArray[node];
        int size = childList.size();
        int[] child;

        // 자식노드를 순차적으로 방문, 자식노드의 깊이 및 전달 시간을 구함
        for(int i = 0; i < size; i++) {
            child = childList.get(i);
            // 현재 노드에서 자식 노드에 전달하기 위해서는 자식 노드까지의 전달 시간 + 1
            child[1] = 1 + this.solution(child[0]);
        }

        // 전달 시간이 오래걸리는 순으로 자식 노드 정렬
        Collections.sort(childList, new Comparator<int[]>() {
            @Override
            public int compare(int[] o1, int[] o2) {
                return o2[1] - o1[1];
            }
        });

        // 오래된 순으로 탐색, 가장 깊이가 깊으면서 전달 시간이 오래 걸리는 노드를 먼저 방문해야함
        for(int i = 0; i < size; i++) {
            child = childList.get(i);
            // 해당 자식노드를 방문하는 순서는 탐색 순서에 비례, 자식 노드의 전달 시간에 현재 노드가 방문하는 순서에 따른 시간을 추가
            child[1] += i;
            // 현재 노드에서 전달 시간이 가장 오래걸리는 자식노드 + 자신이 자식노드를 방문하는 수를 통해 자신이 전달하는데 걸리는 최대 시간을 구함
            maxDept = Integer.max(maxDept, child[1]);
        }

        this.result = Integer.max(this.result, maxDept);

        return maxDept;
    }
}
```  

---
## Reference
[1135-뉴스 전하기](https://www.acmicpc.net/problem/1135)  
