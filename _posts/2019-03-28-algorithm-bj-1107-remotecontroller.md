--- 
layout: single
classes: wide
title: "[풀이] 백준 1107 리모컨"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '고장난 리모컨으로 특정 채널로 가는 최소 버튼 횟수를 구하라'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Divide and Conquer
  - Math
  - Brute Force
  - DFS
---  

# 문제
- 0 ~ 9 까지 숫자, +, - 버튼이 있는 리모컨에서 몇개의 숫자 버튼이 고장 났다.
- `+` 버튼은 현재 채널에서 +1, `-` 버튼은 현재 채널에서 -1 이된다.
- 숫자 버튼은 3을 누르면 3번 채널로, 2 5를 누르면 25번 채널, 1 2 3을 누르면 123번 채널로 이동한다.
- 이동 하려는 채널이 N 번일 때, 현재 100번 채널에서 N 번 채널까지 이동할 수 있는 최소 버튼의 개수를 구하라.

## 입력
첫째 줄에 수빈이가 이동하려고 하는 채널 N (0 ≤ N ≤ 500,000)이 주어진다.  둘째 줄에는 고장난 버튼의 개수 M (0 ≤ M ≤ 10)이 주어진다. 고장난 버튼이 있는 경우에는 셋째 줄에는 고장난 버튼이 주어지며, 같은 버튼이 여러번 주어지는 경우는 없다.

## 출력
첫째 줄에 채널 N으로 이동하기 위해 버튼을 최소 몇 번 눌러야 하는지를 출력한다.

## 예제 입력

```
5457
3
6 7 8
```  

## 예제 출력

```
6
```  

## 예제 입력

```
100
5
0 1 2 3 4
```  

## 예제 출력

```
0
```  

## 예제 입력

```
500000
8
0 2 3 4 6 7 8 9
```  

## 예제 출력

```
11117
```  

# 풀이
- N 번 채널로 이동할 수 있는 방법은 아래와 같은 경우의 수가 있다.
	1. `+`, `-` 버튼으로만 이동
	1. 0 ~ 9 숫자 버튼으로 바로 이동
	1. 위의 두가지 경우를 사용하여 이동
- 이동할 수 있는 채널은 무한대 이지만 문제에서 유추할 수 있는 최대 채널은 1000000 번으로 생각할 수 있다.
	- N 이 가질수 있는 최대값인 500000 번 채널을 이동하려할 때, 0번에서 500000번을 `+` 로 이동하거나, 1000000번에서 500000번을 `-` 로 이동할 수 있는 경우가 같기 때문이다.
- 탐색은 DFS 탐색을 사용한다.
	- 채널의 첫째자리를 정하게되면 계속해서 뒤에 가능한 자리수를 붙여가며 탐색한다.
- 탐색시에 중복 연산을 줄이기 위해서 Memoization 을 사용한다.
	- 최대 채널의 개수만큼 배열을 선언하고 탐색한 채널을 설정해서 이후에 해당 채널은 다시 탐색하지 않도록 한다.
- 불필요한 채널 탐색을 줄이기 위해 첫째자리의 경우, 인접한 숫자나 탐색이 필요한 숫자만 첫째자리로 탐색을 수행한다.
	- 0 ~ 9번의 버튼이 있을 때, N = 500, 고장난 버튼이 1 2 3 일때, 첫째자리로 탐색 가능한 배열은 S = (4, 5, 6) 이다.
		- 첫째자리(5) 가 고장난 버튼이 아니라면 배열에 추가한다. S = (5)
		- 탐색은 오른쪽으로 한번, 왼쪽으로 한번 수행한다. S = (4, 5, 6)
			- 오른쪽으로 탐색하여 6, 왼쪽으로 탐색하여 4를 얻을 수 있다.
	- N = 1111, 고장난 버튼이 1, 2, 3, 4 일때, 첫째자리로 탐색 가능한 배열은 S = (0, 5, 9) 가 된다.
		- 첫째자리가 1혹은 0 이라면 9번 버튼을 추가한다.
		- 999 번채널에서 + 로 이동한 경우가 최소 횟수가 되기 때문이다.
	- N = 999, 고장난 버튼이 7, 8, 9 일때, 첫째자리로 탐색 가능한 배열은 S = (0, 1, 6) 가 된다.
		- 첫째자리가 9라면 1번 버튼를 추가한다.
		- 1000 채널에서 - 로 이동한 경우가 최소 횟수가 되기 때문이다.

```java
public class Main {
    // 리모컨 버튼의 총 개수
    final int BUTTON_MAX = 10;
    final int CHANNEL_MAX = 1000001;

    // 사용할 수 있는 최대 채널 개수
    private int channelMax;
    // 결과값
    private int result;
    // 고장난 버튼의 개수
    private int brokenCount;
    // 고장난 버튼 0: 정상, 1: 고장
    private int[] numberButtons;
    // 이동하려는 채널 N
    private int targetChannel;
    // 이동하려는 채널의 자리수
    private int targetChannelLen;
    // 이미 검사한 채널
    private int[] channelCache;

    public Main() {
        this.result = 0;

        this.input();
        this.solution();
        this.output();
    }


    public static void main(String[] args) {
        Main main = new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            this.targetChannel = Integer.parseInt(reader.readLine());
            this.targetChannelLen = (this.targetChannel + "").length();
            this.brokenCount = Integer.parseInt(reader.readLine());
            this.numberButtons = new int[BUTTON_MAX];
            this.channelMax = (this.targetChannel == 0 ? 1 : this.targetChannel) * 10 + 1;

            if(this.channelMax > CHANNEL_MAX) {
                this.channelMax = CHANNEL_MAX;
            }

            this.channelCache = new int[this.channelMax];

            if (this.brokenCount > 0) {
                StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

                for (int i = 0; i < this.brokenCount; i++) {
                    this.numberButtons[Integer.parseInt(token.nextToken())] = 1;
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        // +, - 버튼만으로 이동했을 때의 버튼의 개수
        this.result = Math.abs(this.targetChannel - 100);

        // 맨처음 자리수에서 이동할 수 있는 버튼 배열
        int[] adjButtons = this.getAdjButtons(0);

        // 모든 버튼을 탐색
        for (int i = 0; i < BUTTON_MAX; i++) {
            // 해당 버튼이 이동가능하다면
            if (adjButtons[i] == 1) {
                // 해당 버튼 숫자를 맨 앞자리로 DFS 탐색
                this.dfs(i);
            }
        }
    }

    // 해당 자리수에서 이동 가능한 버튼
    public int[] getAdjButtons(int channelIndex) {
        // 이동가능한 버튼 저장 1: 이동가능, 0: 불가능
        int[] array = new int[BUTTON_MAX];
        // 버튼 자리수에 위치한 버튼 숫자
        int currentChannelIndexNum = (this.targetChannel + "").charAt(channelIndex) - '0';

        // 고장 나지 않았다면 이동 가능으로 표시
        if (this.numberButtons[currentChannelIndexNum] == 0) {
            array[currentChannelIndexNum] = 1;
        }

        if (this.brokenCount != 10) {
            // 버튼 숫자를 기준으로 오른쪽으로 탐색
            int i = currentChannelIndexNum + 1;

            while (true) {
                if (i > 9) {
                    i = 0;
                }

                if (this.numberButtons[i] == 0) {
                    array[i] = 1;
                    break;
                }

                i++;
            }

            i = currentChannelIndexNum - 1;

            // 버튼 숫자를 기준으로 왼쪽으로 탐색
            while (true) {
                if (i < 0) {
                    i = 9;
                }

                if (this.numberButtons[i] == 0) {
                    array[i] = 1;
                    break;
                }

                i--;
            }
        }

        // 버튼 숫자가 0, 1 일경우 9가 탐색해야 할 경우가 있기 때문에 추가
        if ((currentChannelIndexNum == 0 || currentChannelIndexNum == 1) && this.numberButtons[9] == 0) {
            array[9] = 1;
        }

        // 버튼 숫자가 9일 경우 1을 탐색해야 할 경우가 있기 때문에 추가
        if (currentChannelIndexNum == 9 && this.numberButtons[1] == 0) {
            array[1] = 1;
        }

        return array;
    }

    public void dfs(int currentChannel) {
        // 현재 탐색채널을 문자열로 변경
        String strCurrentChannel = currentChannel + "";
        // 현재 탐색채널의 자리수
        int currentChannelLen = strCurrentChannel.length();
        // 탐색할 다음 채널
        int nextChannel;

        // 탐색하는 채널의 자리수는 targetChannel + 1 자리수 까지만 탐색
        if (currentChannelLen > this.targetChannelLen + 1) {
            return;
        }

        // 현재 채널이 최대 최대 채널보다 클경우 탐색하지 않음
        if (this.channelMax <= currentChannel) {
            return;
        }

        // 채널 자리수가 현재 버튼의 최소값보다 클경우 탐색하지 않음
        if (currentChannelLen > this.result) {
            return;
        }

        // 현재 탐색 채널을 기준으로 버튼 개수 연산
        int gap = Math.abs(this.targetChannel - currentChannel) + currentChannelLen;

        // 현재 버튼의 최소값보다 작을 경우 결과값 갱신
        if (this.result > gap) {
            this.result = gap;
        }

        for (int i = 0; i < BUTTON_MAX; i++) {
            // 사용 가능한 버튼이라면
            if (this.numberButtons[i] == 0) {
                // 뒷자리에 추가
                nextChannel = currentChannel * 10 + i;

                // 최대 채널 보다 작거나 탐색 하지 않은 채널이면
                if (this.channelMax > nextChannel && this.channelCache[nextChannel] == 0) {
                    // 탐색한 채널로 설정
                    this.channelCache[nextChannel] = 1;

                    // 다음 탐색 채널로 다시 DFS 탐색
                    this.dfs(nextChannel);
                }
            }
        }
    }
}
```  

---
## Reference
[1107-리모컨](https://www.acmicpc.net/problem/1107)  
