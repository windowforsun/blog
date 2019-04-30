--- 
layout: single
classes: wide
title: "[풀이] 백준 1302 베스트셀러"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '책중 가장 많이 팔린 책을 알아내자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Sorting
---  

# 문제
- 책 이름이 입력으로 들어올 때 가장 많이 팔린 책의 제목을 출력한다.

## 입력
첫째 줄에 오늘 하루 동안 팔린 책의 개수 N이 주어진다. 이 값은 1,000보다 작거나 같은 자연수이다. 둘째부터 N개의 줄에 책의 제목이 입력으로 들어온다. 책의 제목의 길이는 50보다 작거나 같고, 알파벳 소문자로만 이루어져 있다.

## 출력
첫째 줄에 가장 많이 팔린 책의 제목을 출력한다. 만약 가장 많이 팔린 책이 여러 개일 경우에는 사전 순으로 가장 앞서는 제목을 출력한다.

## 예제 입력

```
5
top
top
top
top
kimtop
```  

## 예제 출력

```
top
```  

## 풀이

```java
public class Main {
    public TestHelper testHelper = new TestHelper();

    class TestHelper {
        private ByteArrayOutputStream out;

        public TestHelper() {
            this.out = new ByteArrayOutputStream();

            System.setOut(new PrintStream(this.out));
        }

        public String getOutput() {
            return this.out.toString().trim();
        }
    }

    private StringBuilder result;
    private int bookCount, max;
    private HashMap<String, Integer> bestsellerMap;
    private ArrayList<String> bestSellerList;

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
            String bookName;
            int count;

            this.bookCount = Integer.parseInt(reader.readLine());
            this.bestsellerMap = new HashMap<>();
            this.bestSellerList = new ArrayList<>();
            this.result = new StringBuilder();
            this.max = 0;

            for(int i = 0; i < this.bookCount; i++) {
                bookName = reader.readLine();
                count = 0;

                if(this.bestsellerMap.containsKey(bookName)) {
                    count = this.bestsellerMap.get(bookName);
                }
                count += 1;

                if(this.max < count) {
                    this.max = count;
                    this.bestSellerList = new ArrayList<>();
                    this.bestSellerList.add(bookName);
                } else if(this.max == count) {
                    this.bestSellerList.add(bookName);
                }

                this.bestsellerMap.put(bookName, count);
            }
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public void output() {
        System.out.println(this.result);
    }

    public void solution() {
        Collections.sort(this.bestSellerList);

        this.result.append(this.bestSellerList.get(0));
    }
}
```  

---
## Reference
[1302-베스트셀러](https://www.acmicpc.net/problem/1302)  
