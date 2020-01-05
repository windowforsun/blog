--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Template Method Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '로직을 공통화 시키면서, 구체화 시킬 수 있는 Template Method 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - DesignPattern
tags:
  - DesignPattern
  - Template Method
---  

## Template Method 패턴이란
- 템플릿이란 정해진 틀을 의미하고, 우리는 템플릿이라는 정해진 틀(규칙)을 두고 변경 될 수 있는 부분만 바꿔, 무언가를 손쉽게 만들어 낼 수 있다. 
		
	![그림 1]({{site.baseurl}}/img/designpattern/2/concept_templatemethod_1.png)
	
	- 우리는 모양자(템플릿)을 사용해서 쉽게 모양을 그릴 수 있다.
	- 그리는 도구를 연필, 펜, 붓 등으로 변경하면서 모양을 그릴 수 있다.
- `Template Method` 패턴이란 상위 클래스가 템플릿 역할을 하는 메서드가 정의되어 있고, 하위 클래스에서 변경될 수 있는 부분에 대해 정의하는 패턴을 뜻한다.
- 상위 클래스에서 특정 로직의 큰 뼈대(템플릿 메서드)를 구축해 놓으면, 하위 클래스에서는 특정 경우마다 변경 될 수 있는 사항들을 구현해 완성된 알고리즘의 결정은 하위 클래스에서 하게 된다.
- `Template Method` 패턴은 상위 클래스에서 큰 뼈대를 작성하는 만큼, 로직을 공통화 할 수 있다.
- `Template Method` 은 아래 클래스의 상속 관계의 특징을 바탕으로 구성된 패턴이다.
	- 상위 클래스에서 정의된 메서드를 하위 클래스에서 사용 할 수 있다.
	- 하위 클래스에 약간의 메서드를 기술해서 새로운 기능을 추가할 수 있다.
	- 하위 클래스에서 메서드를 오버라이딩하면 동작을 변경할 수 있다.
	- 하위 클래스에서 메서드가 구현되길 기대한다.
	- 하위 클래스에 대해 그 메서드의 구현을 요청한다.
- `Template Method` 패턴을 구성할 때 상위 클래스에서 공통 로직을 담는 메서드와 하위 클래스에서 구현하는 추상 클래스 부분을 잘 분배하는 것이 중요하다.
	- 공통 로직 메서드의 역할이 커지면 하위 클래스의 자유도에 제약이 있을 수 있다.
	- 공통 로직 메서드의 역할이 작으면 하위 클래스의 자유도는 높지만, 구현이 어렵고 중복이 많아 질 수 있다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_templatemethod_2.png)

- 패턴의 구성요소
	- `AbstractClass` : 템플릿 메서드를 구현하는 추상 클래스이다. 그리고 템플릿 메서드에서 사용 가능한 추상 메서드를 선언해 하위 클래스에서 구현할 수 있도록 한다.
	- `ConcreteClass` : `AbstractClass` 를 상속 받아 선언된 추상 메서드를 구체적으로 구현한다. `ConcreteClass` 에서 구체적으로 구현된 추상 메서드들은 `AbstractClass` 의 템플릿 메서드에서 사용 된다.

## Prefix, Suffix, Token 으로 구성된 문자열 만들기
- 문자열을 넣으면 정해진 규칙에 따라 Prefix, Suffix, Token 을 붙여 문자열을 만들어 내는 로직을 구현한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_templatemethod_3.png)

### AbstractStringMaker

```java
public abstract class AbstractStringMaker {
    public abstract String prefix();
    public abstract String suffix();
    public abstract String token();

    public String make(String str) {
        StringBuilder builder = new StringBuilder();
        String token = this.token();

        builder.append(this.prefix())
                .append(token)
                .append(str)
                .append(token)
                .append(this.suffix());

        return builder.toString();
    }
}
```  

- `AbstractStringMaker` 는 공통 로직과 추상 메서드가 정의된 추상 클래스이다.
- `Template Method` 패턴에서 `AbstractClass` 역할을 한다.
- 추상 클래스인 `prefix()`, `suffix()`, `token()` 는 하위 클래스에 구체적인 구현을 맡기고 있다.
- 공통 로직이 작성된 `make()` 메서드는 추상 메서드를 사용해서 만들고자 하는 문자열을 만들어 낸다.
	- 문자열은 `prefix` + `token` + `str` + `token` + `suffix` 로 구성 된다.

### NumberStringMaker

```java
public class NumberStringMaker extends AbstractStringMaker {
    @Override
    public String prefix() {
        return "111";
    }

    @Override
    public String suffix() {
        return "999";
    }

    @Override
    public String token() {
        return "555";
    }
}
```  

- `NumberStringMaker` 는 `AbstractStringMaker` 를 상속 받아 추상 메서드를 구현하는 클래스이다.
- `Template Method` 패턴에서 `ConcreteClass` 역할을 수행한다.
- `NumberStringMaker` 는 `prefix()`, `suffix()`, `token()` 메서드에서 숫자로 구성된 문자열을 리턴한다.

### AlphabetStringMaker

```java
public class AlphabetStringMaker extends AbstractStringMaker {
    @Override
    public String prefix() {
        return "AAA";
    }

    @Override
    public String suffix() {
        return "ZZZ";
    }

    @Override
    public String token() {
        return "KKK";
    }
}
```  

- `AlphabetStringMaker` 도 `AbstractStringMaker` 를 상속 받아 추상 메서드를 구현하는 클래스이다.
- `Template Method` 패턴에서 `ConcreteClass` 역할을 수행한다.
- `AlphabetStringMaker` 는 `prefix()`, `suffix()`, `token()` 메서드에서 알파벳으로 구성된 문자열을 리턴한다.


### 테스트

```java
public class TemplateMethodTest {
    @Test
    public void TemplateMethod_NumberStringMaker() {
        // given
        String str = "2020";
        AbstractStringMaker abstractStringMaker = new NumberStringMaker();

        // when
        String actual = abstractStringMaker.make(str);

        // then
        assertThat(actual, is("1115552020555999"));
    }

    @Test
    public void TemplateMethod_AlphabetStringMaker() {
        // given
        String str = "New";
        AbstractStringMaker abstractStringMaker = new AlphabetStringMaker();

        // when
        String actual = abstractStringMaker.make(str);

        // then
        assertThat(actual, is("AAAKKKNewKKKZZZ"));
    }
}
```  

---
## Reference

	