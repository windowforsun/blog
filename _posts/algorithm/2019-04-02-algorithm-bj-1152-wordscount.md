--- 
layout: single
classes: wide
title: "[풀이] 백준 1152 단어의 개수"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '문자열이 주어질 때 띄어쓰기로 구분하여 단어의 개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - String
---  

# 문제
- 영어 대소문자와 띄어쓰기만으로 구성된 문자열이 주어진다.
- 같은 단어가 여러번 나오더라도 카운트 값은 증가한다.
- 문자열에서 띄어쓰기로 구분된 단어의 개수를 구하라

## 입력
첫 줄에 영어 대소문자와 띄어쓰기로 이루어진 문자열이 주어진다. 이 문자열의 길이는 1,000,000을 넘지 않는다. 단어는 띄어쓰기 한 개로 구분되며, 공백이 연속해서 나오는 경우는 없다. 또한 문자열의 앞과 뒤에는 공백이 있을 수도 있다.

## 출력
첫째 줄에 단어의 개수를 출력한다.

## 예제 입력

```
The Curious Case of Benjamin Button
```  

## 예제 출력

```
6
```  

## 예제 입력

```
 Mazatneunde Wae Teullyeoyo
```  

## 예제 출력

```
3
```  

## 예제 입력

```
Teullinika Teullyeotzi 
```  

## 예제 출력

```
2
```  

## 풀이
### StringTokenizer 를 이용해서 해결하기
- Java StringTokenizer 는 token 문자열을 구분자로 기준 문자열을 나누게된다.
- str 이 기준 문자열이고 token 이 공백 문자열이라면 StringTokenizer 를 이용해서 아주 쉽게 답을 구해낼 수 있다.

### 문자열 탐색 이용하기
- isCount 라는 플래그를 둔다.
	- true 일경우 결과 값을 카운트 한다.
- 입력 받은 문자열을 0번째 부터 탐색하며 공백을 만났을 경우 isCount 플래그를 true 로 설정한다.
- 공백이 아닌 다른 문자의 경우 false 로 설정해서 카운트 하지 않도록 한다.
	
  
```java
public class Main {
    // 출력 결과 저장
    private int result;
    // 입력 받은 문자열
    private String inputStr;

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
            this.inputStr = reader.readLine();
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    // 문자열 탐색을 이용
    public void solution() {
        // 문자열 좌우 공백 제거
        this.inputStr = this.inputStr.trim();
        int len = this.inputStr.length();
        // true 가 될경우 카운트를 한다.
        boolean isCount = true;

        for(int i = 0; i < len; i++) {
            // true 일경우 결과값 카운트
            if(isCount) {
                this.result++;
            }
            
            if(this.inputStr.charAt(i) != ' ') {
                // 공백이 아닐 경우 카운트 하지 않음   
                isCount = false;
            } else {
                // 공백일 경우 다음 loop 에서 카운트
                isCount = true;
            }

        }
    }

    // StringTokenizer 를 이용
    public void solution2() {
        // 문자열 좌우 공백 제거
        this.inputStr = this.inputStr.trim();
        // 공백을 기준으로 토큰 객체 생성
        StringTokenizer token = new StringTokenizer(this.inputStr, " ");

        // 토큰의 개수를 결과값으로 설정
        this.result = token.countTokens();
    }
}
```  

---
## Reference
[1152-단어의 개수](https://www.acmicpc.net/problem/1152)  
