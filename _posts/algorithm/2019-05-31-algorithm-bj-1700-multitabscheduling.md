--- 
layout: single
classes: wide
title: "[풀이] 백준 1700 멀티탭 스케줄링"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '가장 효율적으로 멀티탭을 사용해보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Greedy
---  

# 문제
- N 개의 멀티탭에 K 번의 순서로 전자제품을 연결해서 사용해야 한다.
- 멀티텝이 다 찼을 경우에는 하나의 플러그를 빼고 이번에 사용할 플러그를 연결한다.
- 이런 상황에서 플러그를 빼는 횟수를 최소화 하는 방법을 구하라.
- 3구 멀티탭과 아래와 같은 전자제품을 순서대로 사용한다.
	1. 키보드
	1. 헤어드라이기
	1. 핸드폰 충전기
	1. 디지털 카메라 충전기
	1. 키보드
	1. 헤어드라이기
- 디지털 카메라 충전기를 사용 할때 핸드폰 충전기를 뽑게 되면 1번으로 멀티탭을 사용할 수 있다.

## 입력
첫 줄에는 멀티탭 구멍의 개수 N (1 ≤ N ≤ 100)과 전기 용품의 총 사용횟수 K (1 ≤ K ≤ 100)가 정수로 주어진다. 두 번째 줄에는 전기용품의 이름이 K 이하의 자연수로 사용 순서대로 주어진다. 각 줄의 모든 정수 사이는 공백문자로 구분되어 있다. 

## 출력
하나씩 플러그를 빼는 최소의 횟수를 출력하시오. 

## 예제 입력

```
2 7
2 3 2 3 1 2 7
```  

## 예제 출력

```
2
```  

## 풀이
- 매 순서마다 가장 효율적인 선택을 통해 멀티탭의 플러그를 뽑아야 한다.
- 사용할 순서를 이미 모두 알고 있기 때문에, 아래와 같은 조건을 통해 뽑아낼 플러그를 선정할 수 있다.
	- 이후에 사용할 계획이 없는 전자제품
	- 사용하기 까지 가장 순서가 뒤에 있는 전자제품

```java
public class Main {
    public Main() {
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args) {
        new Main();
    }

    private int result;
    private int plugCount;
    private int deviceCount;
    private LinkedList<Integer> remainDeviceList;
    private ArrayList<LinkedList<Integer>> devicePriorityArrayList;


    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int device;

            this.plugCount = Integer.parseInt(token.nextToken());
            this.deviceCount = Integer.parseInt(token.nextToken());
            this.remainDeviceList = new LinkedList<>();
            this.devicePriorityArrayList = new ArrayList();

            for (int i = 0; i <= this.deviceCount; i++) {
                this.devicePriorityArrayList.add(new LinkedList<>());
            }

            token = new StringTokenizer(reader.readLine(), " ");
            for (int i = 0; i < this.deviceCount; i++) {
                device = Integer.parseInt(token.nextToken());
                this.remainDeviceList.addLast(device);

                this.devicePriorityArrayList.get(device).addLast(i + 1);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // 플러그에 연결된 디바이스 배열
        int[] plugArray = new int[this.plugCount];
        int maxPriorityDeviceIndex, maxPriority, priority, pluggingDevice, pluggedCount = 0;
        // 사용중인 디바이스
        int[] isPlugged = new int[this.deviceCount + 1];

        // 초기 상태 설정
        while (pluggedCount < this.plugCount) {
            pluggingDevice = this.remainDeviceList.removeFirst();
            if (isPlugged[pluggingDevice] == 0) {
                plugArray[pluggedCount++] = pluggingDevice;
                isPlugged[pluggingDevice] = 1;
            }
            if (!this.devicePriorityArrayList.get(pluggingDevice).isEmpty()) {
                this.devicePriorityArrayList.get(pluggingDevice).removeFirst();
            }
        }

        while (!this.remainDeviceList.isEmpty()) {
            // 새로 연결할 디바이스
            pluggingDevice = this.remainDeviceList.removeFirst();

            // 플러그에 연결되있지 않으면
            if (isPlugged[pluggingDevice] == 0) {
                maxPriorityDeviceIndex = 100;
                maxPriority = 0;

                // 사용 계획이 없거나, 사용하기 까지 가장 먼 디바이스 선정
                for (int j = 0; j < this.plugCount; j++) {
                    if (this.devicePriorityArrayList.get(plugArray[j]).isEmpty()) {
                        priority = 0;
                    } else {
                        priority = this.devicePriorityArrayList.get(plugArray[j]).getFirst();
                    }

                    // 사용하기 까지 가장 먼 디바이스
                    if (maxPriority < priority) {
                        maxPriorityDeviceIndex = j;
                        maxPriority = priority;
                    }

                    // 사용 계획이 없는 디바이스
                    if (priority == 0) {
                        maxPriorityDeviceIndex = j;
                        break;
                    }
                }

                // 선정된 디바이스 플러그에서 제거후 연결할 디바이스 플러그에 연결
                isPlugged[plugArray[maxPriorityDeviceIndex]] = 0;
                plugArray[maxPriorityDeviceIndex] = pluggingDevice;
                isPlugged[pluggingDevice] = 1;
                this.result++;

            }

            // 사용된 디바이스는 각 디바이스 마다 저장된 사용 순서에서 제거
            if (!this.devicePriorityArrayList.get(pluggingDevice).isEmpty()) {
                this.devicePriorityArrayList.get(pluggingDevice).removeFirst();
            }
        }
    }
}
```  

---
## Reference
[1700-멀티탭 스케줄링](https://www.acmicpc.net/problem/1700)  
