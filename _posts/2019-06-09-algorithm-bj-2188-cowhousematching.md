--- 
layout: single
classes: wide
title: "[풀이] 백준 2188 축사 배정"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '소들이 원하는 방으로 배정할 수 있는 최대개수를 구하자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
  - Network Flow
  - Bipartite Matching
---  

# 문제

## 입력

## 출력

## 예제 입력

```
```  

## 예제 출력

```
```  

## 풀이

```java

// 전에 코드 참고하기 !!


import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.StringTokenizer;

// 2188
public class Main {
    public static ArrayList[] adjList;
    public static int[] cowArray;
    public static int[] houseArray;
    public static boolean[] isVisited;

    public static void main(String[] args) {
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        try {
            StringTokenizer token = new StringTokenizer(reader.readLine(), " ");
            int cowCount = Integer.parseInt(token.nextToken()), houseCount = Integer.parseInt(token.nextToken());
            int house, size;
            adjList = new ArrayList[cowCount + 1];
            cowArray = new int[cowCount + 1];
            houseArray = new int[houseCount + 1];
            isVisited = new boolean[cowCount + 1];

            for (int i = 0; i <= cowCount; i++) {
                adjList[i] = new ArrayList<Integer>();
            }

            for (int i = 1; i <= cowCount; i++) {
                token = new StringTokenizer(reader.readLine(), " ");
                size = Integer.parseInt(token.nextToken());

                for (int j = 0; j < size; j++) {
                    house = Integer.parseInt(token.nextToken());

                    adjList[i].add(house);
                }
            }

            int result = 0;

            for (int i = 1; i <= cowCount; i++) {
                if (cowArray[i] == 0) {
                    Arrays.fill(isVisited, false);
                    if (dfs(i)) {
                        result++;
                    }
                }
            }

            System.out.println(result);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static boolean dfs(int startCow) {
        isVisited[startCow] = true;
        ArrayList<Integer> array = adjList[startCow];
        int size = array.size(), currentHouse;

        for (int i = 0; i < size; i++) {
            currentHouse = array.get(i);
            if (houseArray[currentHouse] == 0 || !isVisited[houseArray[currentHouse]] && dfs(houseArray[currentHouse])) {
                cowArray[startCow] = currentHouse;
                houseArray[currentHouse] = startCow;

                return true;
            }
        }

        return false;
    }
}
```  

---
## Reference
[2188-축사 배정](https://www.acmicpc.net/problem/2188)  
