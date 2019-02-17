--- 
layout: single
classes: wide
title: "디자인 패턴(Design Pattern) 팩토리 메서드 패턴(Factory Method Pattern)"
header:
  overlay_image: /img/designpattern-bg.jpg
subtitle: '팩토리 메서드 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
    - 디자인 패턴
    - Design Pattern
    - Composite Pattern
---  

'팩토리 메서드 패턴이 무엇이고, 어떠한 특징을 가지는지'

# 팩토리 메서드 패턴이란
- 객체, 생성 처리를 서브 클래스로 분리 해 처리하도록 캡슐화하는 패턴
	- 객체의 생성 코드를 별도의 클래스/메서드로 분리함으로써 객체 생성의 변화에 대비하는 데 유용하다.
	- 특정 기능의 구현은 개별 클래스를 통해 제공되는 것이 바람직한 설계다.
		- 기능의 변경이나 상황에 따른 기능의 선택은 해당 객체를 생성하는 코드의 변경을 초래한다.
		- 상황에 따라 적절한 객체를 생성하는 코드는 자주 중복될 수 있다.
		- 객체 생성 방식의 변화는 해당되는 모든 코드 부분을 변경해야 하는 문제가 발생한다.
	- [스크래치지 패턴]({{site.baseurl}}{% link _posts/2019-02-12-designpattern-strategy.md %}), [싱글턴 패턴]({{site.baseurl}}{% link _posts/2019-02-11-designpattern-singleton.md %}), [템플릿 메서드 패턴]({{site.baseurl}}{% link _posts/2019-02-11-designpattern-templatemethod.md %}) 를 사용한다.
	- [생성 패턴 중 하나]({{site.baseurl}}{% link _posts/2019-02-08-designpattern-intro.md %})
	