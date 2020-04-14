--- 
layout: single
classes: wide
title: "[Java 개념] 참조(Reference)"
header:
  overlay_image: /img/java-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
toc: true
use_math: true
---  

## 개요
- [JVM](https://windowforsun.github.io/blog/java/java-concept-jvm-architecture/#runtime-data-area)
에서 보면 사용자가 선언하고 할당한 변수에 대한 값이 `Stack` 과 `Heap` 에 저장된다는 것을 알 수 있다.
- `Java` 에서 변수는 `Primitive` 타입과 `Reference` 타입이 있는데, 그 차이에 관련된 설명과 저장되는 장소에 대해 살펴본다.
- `Wrapper Class`, `Immutable Object` 관련해서도 알아본다.

## Stack, Heap 의 관계
- 메모리의 영역으로 변수의 값과 관련된 데이터가 저장되는 `Stack` 과 `Heap` 의 관계를 간단하게 그림으로 표현하면 아래와 같다.

![그림 1]({{site.baseurl}}/img/java/concept_reference_1.png)

## Stack
- [JVM-Stack](https://windowforsun.github.io/blog/java/java-concept-jvm-architecture/#jvm-language-stacksstack-area)
을 보면 `Stack` 에는 로컬 변수, 리턴 값, 매개변수 등이 저장된다고 하는데, 더욱 자세히 알아본다.
- `Stack` 은 `Thread` 단위로 할당되는 메모리 공간이다.
- `Heap` 에 생성된 객체의 인스턴스 참조를 위한 값이 저장된다.
- `Primitive` 타입의 변수 값이 저장된다.
	- byte, short, int, long, float, double, boolean, char
- `Stack` 에는 로컬 변수가 저장되는데, 이런 로컬 변수는 `visibility` 라는 특성을 가진다.
	- `scope` 와 관련된 개념으로, 구문에서 `{}` 단위로 변수가 사용될 수 있는 범위를 특정 짓는다.



---
## Reference
