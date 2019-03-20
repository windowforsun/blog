--- 
layout: single
classes: wide
title: "[알고리즘] 백준 1003 피보나치 함수"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '피보나치 함수에서 0과 1이 몇개씩 나오는지 출력해보자.'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Fibonacci
  - DP
---  

# 문제

```
int fibonacci(int n) {
    if (n == 0) {
        printf("0");
        return 0;
    } else if (n == 1) {
        printf("1");
        return 1;
    } else {
        return fibonacci(n‐1) + fibonacci(n‐2);
    }
}
```  

위 함수는 N 번째 피보나치 수를 구하는 함수이다.  
fibonacci(3) 을 호출하면 아래와 같은 동작을 수행한다.
1. fibonacci(3)은 fibonacci(2)와 fibonacci(1) (첫 번째 호출)을 호출한다.
1. fibonacci(2)는 fibonacci(1) (두 번째 호출)과 fibonacci(0)을 호출한다.
1. 두 번째 호출한 fibonacci(1)은 1을 출력하고 1을 리턴한다.
1. fibonacci(0)은 0을 출력하고, 0을 리턴한다.
1. fibonacci(2)는 fibonacci(1)과 fibonacci(0)의 결과를 얻고, 1을 리턴한다.
1. 첫 번째 호출한 fibonacci(1)은 1을 출력하고, 1을 리턴한다.
1. fibonacci(3)은 fibonacci(2)와 fibonacci(1)의 결과를 얻고, 2를 리턴한다.  

1은 2번 출력되고, 0은 1번 출력된다. N이 주어졌을 때, fibonacci(N)을 호출했을 때, 0과 1이 각각 몇 번 출력되는지 구하는 프로그램을 작성하시오.


## 입력
첫째 줄에 테스트 케이스의 개수 T가 주어진다.  

각 테스트 케이스는 한 줄로 이루어져 있고, N이 주어진다. N은 40보다 작거나 같은 자연수 또는 0이다.

## 출력
각 테스트 케이스마다 0이 출력되는 횟수와 1이 출력되는 횟수를 공백으로 구분해서 출력한다.

## 예제 입력
3  
0  
1  
3  

## 예제 출력
1 0  
0 1  
1 2  

# 풀이
- 문제에서 제시된 코드를 바탕으로 피보나치 수열은 기본적으로 O(2^N) (정확하게는 2^N / 2) 의 시간복잡도를 갖는다.
- f(3) 일때 호출 스택
	- f(2)
		- f(1)
		- f(0)
	- f(1)	
- 총 4번의 함수 호출이 일어남을 확인할 수 있다.
- 즉 f(40) 의 경우에는 함수 호출 수가 너무 많아 StackOverFlow 에러가 발생하거나 아주 긴시간이 결려 문제를 해결 할 수 있다.

## 해결 방법
- [피보나치 수열](https://ko.wikipedia.org/wiki/%ED%94%BC%EB%B3%B4%EB%82%98%EC%B9%98_%EC%88%98)이란 앞 두수의 합이 바로 뒤의 수가 되는 수열이다.
- 문제에서 제시된 코드를 보면 2 와 같거나 클 경우에는 재귀적으로 함수를 호출하고, 작을 때는 0 과 1을 출력해주는 것을 확인 할 수 있다.
- 그리고 문제에서는 이 0 과 1이 출력되는 개수를 카운트해서 출력을 하면 된다.
- 여기서 중요한 부분은 0 의 개수와 1의 개수를 분리해서 생각하고 이를 나열 해보는 것이다.
- 0의 개수

	| N | f(N) 에서 0의 개수|
	|---|:---:|
	|0|1|
	|1|0|
	|2|1|
	|3|1|
	|4|2|
	|5|3|

- 1의 개수

	| N | f(N) 에서 1의 개수|
	|---|:---:|
	|0|0|
	|1|1|
	|2|1|
	|3|2|
	|4|3|
	|5|5|

- 숫자가 증가할 때 마다 개수의 증가가 피보나치 수열의 방식인 것을 확인 할 수 있다.
- 이제 0 의 개수를 저장하는 배열, 1의 개수를 저장하는 배열을 각각 선언해서 사용하는 DP 방식으로 문제를 해결하면 된다.
- solution 소스코드
	```java
    public void solution() {
        final int MAX = 40; // N 최대 크기
        int[] oneCountArray = new int[MAX + 1]; // 0, 1 의 개수를 저장하는 배열 
        int[] zeroCountArray = new int[MAX + 1]; // Index 는 0부터 시작하므로 최대 크기에 +1

        /**
         * 피보나치 수열에서 f(n) 은 f(n-1) + f(n-2) 이므로
         * n-1 과 n-2 를 구할 수 있도록 우선 몇개의 값을 넣어준다.
         */
        oneCountArray[0] = 0;
        oneCountArray[1] = 1;
        oneCountArray[2] = 1;

        zeroCountArray[0] = 1;
        zeroCountArray[1] = 0;
        zeroCountArray[2] = 1;

        int currentNum = 0;

        // 테스트 케이스 수만큼 배열 탐색
        for (int i = 0; i < this.testCaseCount; i++) {
            
            // 현재 구해야 하는 수
            currentNum = this.numList.get(i);

            // 0 1 2 값보다 클경우 수를 구한다.
            if (currentNum >= 3) {
                
                // 3부터 currentNum 까지 0, 1의 숫자 배열의 값을 피보나치 수열의 방식으로 구한다.
                for (int j = 3; j <= currentNum; j++) {
                    zeroCountArray[j] = zeroCountArray[j - 1] + zeroCountArray[j - 2];
                    oneCountArray[j] = oneCountArray[j - 1] + oneCountArray[j - 2];
                }
            }

            // 결과를 저장하는 변수에 넣는다.
            this.result.append(zeroCountArray[currentNum])
                    .append(" ")
                    .append(oneCountArray[currentNum])
                    .append("\n");
        }
    }
	```  
	
- 전체 소스코드
	
	```java
    public class Main {
        private int testCaseCount;
        private List<Integer> numList;
        private StringBuilder result;
    
        public Main() {
            this.testCaseCount = 0;
            this.numList = new ArrayList<Integer>();
            this.result = new StringBuilder();
        }
    
        public static void main(String[] args) {
            Main main = new Main();
            main.input();
            main.solution();
            main.output();
        }
    
        public void input() {
            BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
    
            try {
                this.testCaseCount = Integer.parseInt(reader.readLine());
                int num = 0;
    
                for (int i = 0; i < this.testCaseCount; i++) {
                    num = Integer.parseInt(reader.readLine());
                    this.numList.add(num);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    
        public void output() {
            System.out.print(this.result.toString());
        }
    
        public void solution() {
            final int MAX = 40; // N 최대 크기
            int[] oneCountArray = new int[MAX + 1]; // 0, 1 의 개수를 저장하는 배열
            int[] zeroCountArray = new int[MAX + 1]; // Index 는 0부터 시작하므로 최대 크기에 +1
    
            /**
             * 피보나치 수열에서 f(n) 은 f(n-1) + f(n-2) 이므로
             * n-1 과 n-2 를 구할 수 있도록 우선 몇개의 값을 넣어준다.
             */
            oneCountArray[0] = 0;
            oneCountArray[1] = 1;
            oneCountArray[2] = 1;
    
            zeroCountArray[0] = 1;
            zeroCountArray[1] = 0;
            zeroCountArray[2] = 1;
    
            int currentNum = 0;
    
            // 테스트 케이스 수만큼 배열 탐색
            for (int i = 0; i < this.testCaseCount; i++) {
    
                // 현재 구해야 하는 수
                currentNum = this.numList.get(i);
    
                // 0 1 2 값보다 클경우 수를 구한다.
                if (currentNum >= 3) {
    
                    // 3부터 currentNum 까지 0, 1의 숫자 배열의 값을 피보나치 수열의 방식으로 구한다.
                    for (int j = 3; j <= currentNum; j++) {
                        zeroCountArray[j] = zeroCountArray[j - 1] + zeroCountArray[j - 2];
                        oneCountArray[j] = oneCountArray[j - 1] + oneCountArray[j - 2];
                    }
                }
    
                // 결과를 저장하는 변수에 넣는다.
                this.result.append(zeroCountArray[currentNum])
                        .append(" ")
                        .append(oneCountArray[currentNum])
                        .append("\n");
            }
        }
    }

	```  
	


---
## Reference
[1003-피보나치 함수](https://www.acmicpc.net/problem/1003)  
