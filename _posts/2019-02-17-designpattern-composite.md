--- 
layout: single
classes: wide
title: "디자인 패턴(Design Pattern) 컴퍼지트 패턴(Composite Pattern)"
header:
  overlay_image: /img/designpattern-bg.jpg
subtitle: '컴퍼지트 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Composite Pattern
---  

'컴퍼지트 패턴이 무엇이고, 어떠한 특징을 가지는지'

### 컴퍼지트 패턴이란
  - 여러 개의 객체들로 구성된 복합 객체와 단일 객체를 클라이언트에서 구별 없이 다루게 해주는 해턴
    - 전체 부분의 관계(ex. Dictionary-File)를 갖는 객체들 사이의 관계를 정의할 때 유용하다.
    - 또한 클라이언트는 전체와 부분을 구분하지 않고 동일한 인터페이스를 사용할 수 있다.
    - [구조 패턴 중 하나]({{site.baseurl}}{% link _posts/2019-02-08-designpattern-intro.md %})
  - ![컴퍼지트 패턴 예시1](/img/designpattern-composite-ex-1-classdiagram.png)
  - <img src="{{site.baseurl}}/path/img/designpattern-composite-ex-1-classdiagram.png">
  - 역할이 수행하는 작업
    - Component
      - 구체적인 부분
      - Leaf 클래스와 전체에 해당하는 Composite 클래스에 공통 인터페이스를 정의
    - Leaf
      - 구체적인 부분 클래스
      - Composite 객체의 부붐으로 설정
    - Composite
      - 전체 클래스
      - 복수 개의 Component를 갖도록 정의
      - 복수 개의 Leaf, 심지어 복수 개의 Composite 객체를 부분으로 가질 수 있음
      
### 예시
#### 컴퓨터에 추가 장치 지원하기
- ![컴퍼지트 패턴 컴퓨터1](/img/designpattern-composite-computer-1-classdiagram.png)
- ![옵저버 패턴 예시1](/img/designpattern-observer-ex-1-classdiagram.png)
- ![컴퍼지트 패턴 컴퓨터1](/img/designpattern-composite-computer-1-classdiagram.png)
- ![컴퍼지트 패턴 컴퓨터1](/img/designpattern-composite-computer-1-classdiagram.png)
- ![컴퍼지트 패턴 컴퓨터1](/img/designpattern-composite-computer-1-classdiagram.png)
- ![옵저버 패턴 예시1](/img/designpattern-observer-ex-1-classdiagram.png)
- ![옵저버 패턴 예시1](/img/designpattern-observer-ex-1-classdiagram.png)
- ![업저버 패턴 예시1](/img/designpattern-observer-ex-1-classdiagram.png)
- sdfsdfdsfs