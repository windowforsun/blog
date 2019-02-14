--- 
layout: post
title: "디자인 패턴(Design Pattern) 데코레이터 패턴(Decorator Pattern)"
subtitle: '데코레이터 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
tags:
    - 디자인 패턴
    - Design Pattern
    - Decorator Pattern
---  
[참고](https://lee-changsun.github.io/2019/02/08/designpattern-intro/)  

### 데코레이터 패턴이란
- 객체의 결합을 통해 기능을 동적으로 유연하게 확장 할 수 있게 해주는 패턴
    - 기본 기능에 추가할 수 있는 기능의 종류가 많은 경우에 각 추가 기능을 Decorator클래스로 정의 한 후 필요한 Decorator객체를 조합함으로써 추가 기능의 조합을 설계하는 방식이다.
    - ex) 기본 도로 표시 기능에 차선 표시, 고통량 표시, 교차로 표시, 단속 카메라 표시의 4가지 추가 기능이 있을 때 추가 기능의 모든 조합은 15가지가 된다.
    - 데코레이터 패턴을 이용하여 필요 추가 기능의 조합을 동적으로 생성 할 수 있다.
    - 구조 패턴 중 하나
- ![]()
- 역할이 수행하는 작업
    - Component
        - 기본 기능을 뜻하는 ConcreteComponent와 추기 기능을 뜻하는 Decorator의 공통 기능을 정의
        - 클라이언트는 Component를 통해 실제 객체를 사용함
    - ConcreteComponent
        - 기본 기능을 구현하는 클래스
    - Decorator
        - 많은 수가 존재하는 구체적인 Decorator의 공통 기능을 제공
    - ConcreteDecoratorA, ConcreteDecoratorB
        - Decorator의 하위 클래스로 기본 기능에 추가되는 개별적인 기능을 뜻함
        - ConcreteDecorator 클래스는 ConcreteComponent 객체에 대한 참조가 필요한데, 이는 Decorator클래스에서 Component클래스로의 합성(Composition) 관계를 통해 표현됨
    
#### 참고
- 합성(Composition) 관계
    - 생성자에서 필드에 대한 객체를 생성하는 경우
    - 전체 객체의 라이프타임과 부분 객체의 라이프 타임은 의존적이다.
    - 전체 객체가 없어지면 부분 객체도 없어진다.
    
### 예시
#### 도로 표시 방법 조합하기
