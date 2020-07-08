--- 
layout: single
classes: wide
title: "[풀이] 백준 1158 조세퍼스 문제"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '조세퍼스 순열을 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Linked List
---  

# 문제
- 1 ~ N 까지의 숫자가 원을 이루고 있다.
- 순서대로 K 번째 숫자를 제거한다.
- 한 숫자가 제거되면 남은 숫자로 이루어진 원을 따라 K 번째 숫자를 제거한다.
- 이 과정은 N 개의 숫자가 모두 제거 될때까지 반복한다.
- 원을 이루고 있는 숫자를 제거하는 순서를 조세퍼스 순열이라고 한다.
- 이 조세퍼스 순열을 구하라.


## 입력
첫째 줄에 N과 K가 빈 칸을 사이에 두고 순서대로 주어진다. (1 ≤ K ≤ N ≤ 5,000)

## 출력
예제와 같이 조세퍼스 순열을 출력한다.

## 예제 입력

```
7 3
```  

## 예제 출력

```
<3, 6, 2, 7, 5, 1, 4>
```  

## 풀이
- N 개의 숫자에서 K 번째의 숫자를 제거한다고 할때 제거하는 인덱스를 구하는 수식은 아래와 같다.
	- index = ((0 + K)  % N) - 1
- 조세퍼스 순열은 전에 제거된 index 부터 다시 K 번째 숫자를 제거하게 된다.
	- index = ((index + K) % N) - 1
- N 개의 숫자는 순열을 반복해서 구할때 마다 줄어든다.
	- N -= 1;
- index 가 음수가 나올 경우에는 현재 숫자 리스트의 마지막 인덱스로 설정한다.
  
```java
public class Main {
    // 출력 결과 저장
    private StringBuilder result;
    // 숫자의 개수
    private int count;
    // 삭제하는 순서
    private int index;

    public Main() {
        this.result = new StringBuilder();
        this.input();
        this.solution();
        this.output();
    }

    public static void main(String[] args){
        new Main();
    }

    public void input() {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try{
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");

            this.count = Integer.parseInt(token.nextToken());
            this.index = Integer.parseInt(token.nextToken());
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        LinkedList<Integer> linkedList = new LinkedList<>();
        int removeIndex = 0, size = this.count, num;

        // 링크드리스트에 1 ~ N 까지의 값을 넣어준다.
        for(int i = 1; i <= this.count; i++) {
            linkedList.addLast(i);
        }

        this.result.append("<");

        // 링크드리스트에 원소가 있을 때까지 반복한다.
        while(true) {
            // 삭제해야 하는 링크드리스트 인덱스
            removeIndex = ((removeIndex + this.index) % size) - 1;

            // 0보다 작을 경우 마지막 링크드리스트 인덱스 값으로 설정
            if(removeIndex < 0) {
                removeIndex = size - 1;
            }

            // 삭제할 인덱스 값 삭제 및 결과값에 저장
            num = linkedList.remove(removeIndex);
            this.result.append(num);

            if(linkedList.isEmpty()) {
                break;
            } else {
                this.result.append(", ");
            }

            // 원소 하나를 삭제할 때마다 크기도 감소
            size--;
        }

        this.result.append(">");
    }
}
```  

---
## Reference
[1158-조세퍼스 문제](https://www.acmicpc.net/problem/1158)  
