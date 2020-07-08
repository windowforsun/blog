--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Memento Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '현재 인스턴스의 상태를 저장하고 이를 컨트롤 할 수 있는 Memento 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Memento
use_math : true
---  

## Memento 패턴이란
- `Memento` 는 기념물, 기억이라는 의미를 가진 것과 같이, 인스턴스의 상태를 저장하고 컨트롤을 캡슐화의 파괴에 빠지지 않고 구성할 수 있는 패턴이다.
- 인스턴스 상태를 저장하고 컨트롤 한다는 것은 아래와 같은 동작을 의미한다.
	- undo(실행취소)
	- redo(재 실행)
	- history (작업 이력)
	- snapshot(현재 상태저장)
- 캡슐화의 파괴는 의도하지 않은 액세스가 허용됨에 따라 클래스 내부 코드를 의존하는 코드가 다양한 경로로 분산돼 클래스의 수정을 어렵게 만드는 것을 의미한다.
	
### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_memento_1.png)

- `Originator` : 작성자의 역할로 자신의 현재 상태를 저장하고 싶을 때 Memento 를 생성하거나, Memento 로 상태를 되돌리는 동작을 수행한다.
- `Memento` : 기념품(기억)의 역할로 `Originator` 의 내부 정보를 담고, 해당 정보는 `Originator` 외에는 공개가 되지 않도록 하는 클래스이다.
	- `wide interface`(넓은 인터페이스) : `Oroginator` 가 사용하는 인터페이스로, 해당 객체의 내부 정보에 접근 가능한 인터페이스를 의미한다.
	- `narrow interface`(좁은 인터페이스) : 외부에서 사용하는 인터페이스로, 해당 객체의 정보를 제한적으로 접근하도록 하는 인터페이스를 의미한다.
- `CareTaker` : 관리인의 역할로 `Originator` 의 상태를 실제로 컨트롤하는 클래스이다. `Orginator` 의 상태인 `Memento` 에 직접 접근은 불가능하지만 `Originator` 에서 제공하는 인터페이스를 통해 동작을 수행한다.

### 인터페이스와 접근제어자
- Java 에서는 아래와 같은 4개의 접근 제어자를 제공한다.

	접근 제어자|설명
	---|---
	public|모든 클래스에서 접근 가능하다.
	protected|동일한 패키지 및 하위 클래스에서 접근 가능하다.
	없음|동일한 패키지에서만 접근 가능하다.
	private|동일 클래스에서만 접근 가능하다.
	
- `Memento` 패턴은 이런 접근 제어자를 통해 `Originator` 의 상태를 담고있는 `Memento` 클래스의 정보가 외부로 노출되지 않도록 제어한다.

### Memento 의 관리
- 인스턴스의 상태를 저장하는 `Memento` 는 한개 이상 저장하면서 다양한 방식으로 인스턴스의 상태를 저장해서 관리 할 수있다.

### Memento 유효기간
- `Memento` 의 저장공간이 메모리가 아니라, 별도의 공간이라면 유효기간을 둬서 관리할 수 있다.
- 인스턴스의 상태는 개발과정에서 바뀔 수 있기 때문에, 저장된 `Memento` 와 모순이 발생할 수 있다.

### CareTaker, Originator 의 역할
- `CareTaker` 는 상태를 컨트롤 하는 시점을 결정한다.
	- 언제 undo, redo 를 할지 판별한다.
	- 어느 시점에 `Memento` 를 생성할지 결정한다.
	- `Originator` 의 기능을 컨트롤한다.
- `Originator` 는 상태 컨트롤 동작을 수행한다.
	- undo, redo, `Memento` 생성 및 설정을 실제로 수행한다.
- `CareTaker`, `Originator` 는 추가 개발에 있어서도 각 역할에 해당하는 부분만 변경하면 된다.

## 텍스트 에디터
- 글을 추가로 작성할 수는 있지만, 지우는 동작이 없는 에디터가 있다고 가정한다.
- 에디터에는 undo, redo 기능을 제공한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_memento_2.png)

### Memento

```java
public class Memento {
    String str;

    Memento(String str) {
        this.str = str;
    }

    public int getLength() {
        return this.str.length();
    }
}
```  

- `Memento` 는 `Page` 인스턴스의 상태를 저장하는 클래스이다.
- `Memento` 패턴에서 `Memento` 역할을 수행한다.
- `Page` 의 상태 값인 `str` 필드와 생성자는 `default` 접근 제어자를 통해 같은 패키지에서만 접근 가능하다.
- 외부 패키지에서는 `getLength()` 메소드를 통해 상태의 문자열 길이값을 알 수 있다.
- `Memento` 인스턴스의 생성은 생성자를 통해 `Page` 의 상태 값인 `str` 을 인자값을 받아 생성한다.

### Page

```java
public class Page {
    private StringBuilder str;

    public Page() {
        this.str = new StringBuilder();
    }

    public void addStr(String str) {
        this.str.append(str);
    }

    public Memento createMemento() {
        return new Memento(this.str.toString());
    }

    public void restoreMemento(Memento memento) {
        if(memento == null) {
            memento = new Memento("");
        }

        this.str = new StringBuilder(memento.str);
    }

    public String getStr() {
        return this.str.toString();
    }
}
```  

- `Page` 클래스는 글을 작성하는 하나의 페이지를 나타내는 클래스이다.
- `Memento` 패턴에서 `Originator` 역할을 수행한다.
- 작성되는 문자열은 `str` 필드에 `addStr()` 메소드를 사용해서 저장한다.
- `createMemento()` 메소드를 통해 현재 자신의 상태를 스냅샷 찍는 것처럼 `Memento` 인스턴스를 생성한다.
- `restoreMemento()` 메소드는 `Memento` 인스턴스를 인자값으로 받아 현재 인스턴스의 상태를 `Memento` 에 저장된 상태로 설정한다.

### Editor

```java
public class Editor {
    private Page page;
    private LinkedList<Memento> undoList;
    private LinkedList<Memento> redoList;

    public Editor() {
        this.page = new Page();
        this.undoList = new LinkedList<>();
        this.redoList = new LinkedList<>();
    }

    public String addStr(String str) {
        this.page.addStr(str);
        this.undoList.addLast(this.page.createMemento());
        return this.page.getStr();
    }

    public String undo() {
        if(!this.undoList.isEmpty()) {
            this.redoList.addLast(this.undoList.removeLast());
            this.page.restoreMemento(this.undoList.peekLast());
        }

        return this.page.getStr();
    }

    public String redo() {
        if(!this.redoList.isEmpty()) {
            this.page.restoreMemento(this.redoList.peekLast());
            this.undoList.addLast(this.redoList.removeLast());
        }

        return this.page.getStr();
    }
}
```  

- `Editor` 클래스는 텍스트를 작성하는 에디터를 나타내는 클래스이다.
- `Memento` 패턴에서 `CareTaker` 역할을 수행한다.
- `page` 필드는 현재 에디터에서 사용할 페이지를 나타내는 필드이다.
- `undoList`, `redoList` 는 `Memento` 의 리스트로 각각 undo 와 redo 기능을 사용할 떄 사용하는 필드 값이다.
- `addStr()` 은 `Page` 에 문자열을 추가하는 메소드인데 아래와 같이 동작한다.
	1. `Page` 의 `addStr()` 메소드를 사용해서 문자열을 추가한다.
	1. 현재 `Page` 의 스냅샷인 `Memento` 인스턴스를 `createMemento()` 메소드를 사용해서 생성한다.
	1. 생성된 `Memento` 를 `undoList` 의 마지막에 추가한다.
- `undo()` 는 수행한 결과를 되돌리는 메소드로 아래와 같이 동작한다.
	1. `undoList` 가 비었는지 검사하고 비었다면, 되돌릴 상태가 없으므로 현재 `Page` 의 문자열을 리턴한다.
	1. 비어있지 않다면, `undoList` 에서 마지막 (가장 최근) 상태를 가져오고 지운다.
	1. 가져온 마지막 상태를 `redoList` 에 추가한다.
	1. `undoList` 에서 마지막(가장 최근) 상태를 `Page` 에 `restoreMemento()` 메소드를 통해 설정한다.
- `redo()` 는 실행 취소(undo)한 결과를 다시 재실행하는 메소드로 아래와 같이 동작한다.
	1. `redoList` 가 비었는지 검사하고 비었다면, 실행 취소할 상태가 없으므로 현재 `Page` 의 문자열을 리턴한다.
	1. `redoList` 에서 마지막(가장 최근) 상태를 가져와서 `Page` 에 `restoreMemento()` 메소드를 통해 설정한다.
	1. 가져온 마지막 상태를 `undoList` 의 마지막에 추가한다.
	1. `redoList` 의 마지막 상태 값을 지운다.
	
### Memento 의 처리과정

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_memento_3.png)
	
### 테스트

```java
public class MementoTest {
    @Test
    public void Editor_addStr() {
        // given
        Editor editor = new Editor();
        editor.addStr("str1");

        // when
        String actual = editor.addStr("str2");

        // then
        assertThat(actual, is("str1str2"));
    }

    @Test
    public void Editor_Undo() {
        // given
        Editor editor = new Editor();
        editor.addStr("str1");
        editor.addStr("str2");

        // when
        String actual = editor.undo();

        // then
        assertThat(actual, is("str1"));
    }

    @Test
    public void Editor_Redo() {
        // given
        Editor editor = new Editor();
        editor.addStr("str1");
        editor.addStr("str2");
        editor.undo();

        // when
        String actual = editor.redo();

        // then
        assertThat(actual, is("str1str2"));
    }

    @Test
    public void Editor_Empty_Undo() {
        // given
        Editor editor = new Editor();

        // when
        String actual = editor.undo();

        // then
        assertThat(actual, isEmptyString());
    }

    @Test
    public void Editor_Empty_Redo() {
        // given
        Editor editor = new Editor();

        // when
        String actual = editor.redo();

        // then
        assertThat(actual, isEmptyString());
    }
}
```  

---
## Reference

	