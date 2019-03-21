--- 
layout: single
classes: wide
title: "[풀이] 백준 1005 ACM Craft"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '위상정렬을 통해 건물이 지어지는 최소 시간을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Topological Sort
  - DFS
---  

# 문제
- 각 건물을 건설하기 위해서는 시작해서 완성될 때까지 Delay 가 존재한다.
- 1번 건물을 지으면 2, 3번 건물을 지을 수 있다.
- 2, 3번 건물을 모두 지으면 4번 건물을 지을 수 있다.
- 이때 특정건물을 가장 빨리 지을 때까지 걸리는 최소 시간을 구하라

## 입력
첫째 줄에는 테스트케이스의 개수 T가 주어진다. 각 테스트 케이스는 다음과 같이 주어진다. 첫째 줄에 건물의 개수 N 과 건물간의 건설순서규칙의 총 개수 K이 주어진다. (건물의 번호는 1번부터 N번까지 존재한다)  
둘째 줄에는 각 건물당 건설에 걸리는 시간 D가 공백을 사이로 주어진다. 셋째 줄부터 K+2줄까지 건설순서 X Y가 주어진다. (이는 건물 X를 지은 다음에 건물 Y를 짓는 것이 가능하다는 의미이다)  
마지막 줄에는 백준이가 승리하기 위해 건설해야 할 건물의 번호 W가 주어진다. (1 ≤ N ≤ 1000, 1 ≤ K ≤ 100000 , 1≤ X,Y,W ≤ N, 0 ≤ D ≤ 100000) 

## 출력
건물 W를 건설완료 하는데 드는 최소 시간을 출력한다. 편의상 건물을 짓는 명령을 내리는 데는 시간이 소요되지 않는다고 가정한다.  
모든 건물을 지을 수 없는 경우는 없다.  

## 예제 입력

```
2
4 4
10 1 100 10
1 2
1 3
2 4
3 4
4
8 8
10 20 1 5 8 7 1 43
1 2
1 3
2 4
2 5
3 6
5 7
6 7
7 8
7
```  

## 예제 출력

```
120
39
```  

# 풀이
- 각 건물을 짓기전에 선행 선물이 있는지 확인해봐야 한다.
- 하나의 건물을 지을 때 선행 건물이 모두 지어져야 해당 건물을 지을 수 있다.
	- 4번 건물을 지을 때 2, 3 번 건물이 모두 지어져야 한다.
	- 4번 건물을 지을 때 걸리는 시간은 2, 3번 중 가장 오래걸리는 시간에서 4번 건물이 걸리는 시간을 더하는 것이다.

## 해결방법
- 진입차수가 0인(바로 지을 수 있는) 건물 번호 부터 시작해서 DFS 탐색을 한다.
- 각 건물의 결과시간을 저장하는 곳에 각 건물을 지을 때 걸리는 시간을 넣어준다.
- DFS 탐색을 위해 진입차수가 0일 경우 queue에 넣어준다.
- queue 에서 건물 번호를 하나씩 빼서, 해당 건물의 다음 건물 번호를 탐색한다.
- 다음 건물 번호의 진입차수를 감소시키고 다음 건물의 시간을 갱신한다.
	- 다음 건물의 결과시간 = max(다음 건물의 결과시간, 지금 건물의 결과시간 + 다음 건물의 시간)
- 다음 건물의 진입 차수가 0 일 경우 해당 건물도 지을 수 있기 때문에 queue 에 넣어준다.

```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 각 건물 번호가 걸리는 시간
    private int[] delayTime;
    // 테스트 케이스 수
    private int testCaseCount;
    // 건물의 선행 관계 저장
    private List[] matrix;
    // 걸리는 시간을 알아내야 하는 건물 번호
    private int target;
    // 건물 개수
    private int buildingCount;
    // 건물 선행 관계 개수
    private int buildingCaseCount;
    // 각 건물 번호의 진입차수 (선행 되어야 하는 건물의 개수)
    private int[] indegree;

    public Main() {
        this.result = new StringBuilder();
        this.delayTime = null;
    }

    public static void main(String[] args) {
        Main main = new Main();
        main.input();
        main.output();
    }

    public void input(){
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            this.testCaseCount = Integer.parseInt(reader.readLine());
            StringTokenizer token;

            for(int i = 0; i < this.testCaseCount; i++ ) {
                token = new StringTokenizer(reader.readLine(), " ");
                buildingCount = Integer.parseInt(token.nextToken());
                buildingCaseCount = Integer.parseInt(token.nextToken());
                this.delayTime = new int[buildingCount + 1];
                this.matrix = new List[buildingCount + 1];
                this.indegree = new int[buildingCount + 1];

                token = new StringTokenizer(reader.readLine(), " ");
                for(int j = 1; j < buildingCount + 1; j++){
                    this.matrix[j] = new ArrayList<Integer>();
                    this.delayTime[j] = Integer.parseInt(token.nextToken());
                }

                for(int j = 0; j < buildingCaseCount; j++){
                    token = new StringTokenizer(reader.readLine(), " ");
                    int from = Integer.parseInt(token.nextToken());
                    int to = Integer.parseInt(token.nextToken());

                    this.matrix[from].add(to);
                    this.indegree[to]++;
                }

                this.target = Integer.parseInt(reader.readLine());
                this.solution();
            }

        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.print(this.result);
    }

    public void solution() {
        // 지을수 있는 건물의 번호를 넣는 큐
        LinkedList<Integer> queue = new LinkedList<>();
        // 각 건물 번호를 짓는데 걸리는 시간
        int[] resultTime = new int[this.buildingCount + 1];
        int current;

        // indegree 가 0 일때(건물을 짓기위한 선행 건물이 없을경우) queue 에 넣어준다.
        for(int i = 1; i <= this.buildingCount; i++) {
            resultTime[i] = this.delayTime[i];

            if(this.indegree[i] == 0) {
                queue.addLast(i);
            }
        }

        // queue 에 원소가 있을 때 까지
        while(!queue.isEmpty()) {
            // 현재 지을수 있는 건물 번호
            current = queue.removeFirst();

            // 현재 번호 건물을 지었을 경우 다음에 지을 수 있는 건물 번호를 탐색
            for(int next : (List<Integer>)this.matrix[current]) {
                // 다음에 지을 수 있는 건물번호의 진입차수 감소
                this.indegree[next]--;

                // 다음 건물 번호의 결과시간이랑 현재 건물의 시간 + 다음 건물의 시간을 비교한 것중 큰 것을 결과시간에 넣는다.
                resultTime[next] = Integer.max(resultTime[next], resultTime[current] + this.delayTime[next]);

                // 다음 건물번호의 진입차수가 0 이면 건물을 지을 수 있으므로 queue 에 넣는다.
                if(this.indegree[next] == 0 ){
                    queue.addLast(next);
                }
            }
        }

        this.result.append(resultTime[this.target]).append("\n");
    }
}
```  


---
## Reference
[1005-ACM Craft](https://www.acmicpc.net/problem/1005)  
