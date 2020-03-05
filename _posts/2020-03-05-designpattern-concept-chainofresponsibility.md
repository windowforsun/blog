--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Chain Of Responsibility Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '요청과 처리의 연결을 책임을 떠넘기는 방식으로 유연하게 만드는 Chain Of Responsibility 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Chain Of Responsibility
use_math : true
---  

## Chain Of Responsibility 패턴이란
- 책임 연쇄 패턴은 복수의 객체를 사슬(chain)처럼 연결해서, 목적에 맞는 객체를 선택해 처리를 수행하는 패턴이다.
- 자신이 담당하는 부분이 아니라면 자신이 처리하지 않고 다음 객체로 처리를 떠넘기는 방식이다.
- 요청부분과 처리 부분을 연결해서 유연한 구조로 구성이 가능하다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_chainofresponsibility_1.png)

- 패턴의 구성요소
	- `Handler` : 요구를 처리하는 처리자의 역할로 처리에 필요한 메소드를 정의한다. 다음 처리자를 가지고 있으면서, 자신이 처리할 부분이 아니면 다음 처리자에게 떠넘긴다.
	- `ConcreteHandler`: `Handler` 를 구현한 구체적인 처리자로 담당하는 부분의 처리를 구현한다.
	- `Client` : 처리를 요구하는 요구자의 역할로 `Handler` 에게 처리할 것들을 요구한다.
- 요구와 처리의 유연한 연결
	- 책임 연쇄 패턴의 가장 큰 특징은 다양한 요구를 처리할 때, 중앙집중식으로 처리하지 않고 각 처리자가 자신이 처리해야할 부분만 처리한다는 점에 있다.
- 동적인 처리구조
	- 다양한 종류의 처리를 구성할 때, 마지막 처리자에 새로운 처리자를 추가해 주는 방식으로 확장해 나갈 수 있다.
- 각 처리자는 독립적으로 자신의 처리만 수행한다.
	- 처리자는 자신이 처리해야 되는지에 대한 판별문을 통해, 자신이 처리하는 경우에만 처리과정을 실행 시키고 그렇지 않은 경우는 다음 처리자에게 떠넘기면 된다.
- 떠넘기는 과정에서 지연이 발생할 수 있다.
	- 계속해서 처리자를 찾는 과정이 중앙지중식과 같이 누군가 지정해주는 방법보다는 지연이 발생할 수 있다.

## 요청 처리하고 응답하기
- 요청이 들어오면 처리를 수행할 처리자를 선별하고, 처리를 수행한 처리자의 이름을 담아 응답해주는 요청-응답 과정을 구현한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_chainofresponsibility_2.png)

### Request

```java
public class Request {
    private int id;

    public Request(int id) {
        this.id = id;
    }

    public int getId() {
        return id;
    }
}
```  

- `Request` 는 요청의 형식을 나타내는 클래스이다.
- 요청에는 `id` 값이 있고, 처리자는 `id` 값을 통해 자신이 처리할 요청인지 판별한다.

### Response

```java
public class Response {
    private int id;
    private String receiverName;

    public Response(int id, String receiverName) {
        this.id = id;
        this.receiverName = receiverName;
    }

    public int getId() {
        return id;
    }

    public String getReceiverName() {
        return receiverName;
    }
}
```  

- `Response` 는 응답 형식을 나타내는 클래스이다.
- 응답에는 요청의 `id` 값과 처리자의 이름인 `receiverName` 을 통해 어떤 요청을 어떤 처리자가 처리했는지 알 수있도록 한다.

### Receiver

```java
public abstract class Receiver {
    private String name;
    private Receiver next;

    public Receiver(String name) {
        this.name = name;
    }

    public Receiver setNext(Receiver next) {
        this.next = next;
        return next;
    }

    public String getName() {
        return name;
    }

    public final Response receive(Request request) {
        Response response;

        if(this.resolve(request)) {
            response = this.done(request);
        } else if(this.next != null) {
            response = this.next.receive(request);
        } else {
            response = this.fail(request);
        }

        return response;
    }

    protected abstract boolean resolve(Request request);

    protected Response done(Request request) {
        return new Response(request.getId(), this.name);
    }

    protected Response fail(Request request) {
        return new Response(request.getId(), null);
    }
}
```  

- `Receiver` 는 처리자에 대한 체인구성 및 공통적인 처리, 처리자 판별 메소드를 정의한 추상 클래스이다.
- `Chain Of Responsibility` 패턴에서 `Handler` 역할을 수행한다.
- `name` 필드에는 처리자의 이름, `next` 필드는 `setNext(Receiver)` 를 통해 설정된 다음 처리자를 의미한다.
- `receive(Request)` 메소드는 [Template Method]({{site.baseurl}}{% link _posts/2020-01-05-designpattern-concept-templatemethod.md %})
패턴으로 구성돼 있고, 처리자를 판별하고 처리를 수행해 응답을 리턴한다.
- `done(Request)` 메소드는 처리가능한 요청일 때 수행하고, 현재 처리자의 이름을 응답값에 추가한다.
- `fail(Request)` 메소드는 구성된 처리자들 중 요청을 모두 처리할 수 없을 경우 수행하고, 처리자의 이름에 `null` 값을 설정한다.
- `resolve(Request)` 메소드에서 하위 처리자가 요청을 처리해야 하는지에 대한 판별문을 작성하는 메소드이다.

### OddReceiver, RangeReceiver, ListReceiver

```java
public class OddReceiver extends Receiver {
    public OddReceiver(String name) {
        super(name);
    }

    @Override
    protected boolean resolve(Request request) {
        return request.getId() % 2 == 1;
    }
}
```  

```java
public class RangeReceiver extends Receiver {
    private int min;
    private int max;

    public RangeReceiver(String name, int min, int max) {
        super(name);
        this.min = min;
        this.max = max;
    }

    @Override
    protected boolean resolve(Request request) {
        return this.min <= request.getId() && this.max >= request.getId();
    }
}
```  

```java
public class ListReceiver extends Receiver{
    private int[] idArray;

    public ListReceiver(String name, int ...id) {
        super(name);
        this.idArray = id;
    }

    @Override
    protected boolean resolve(Request request) {
        return Arrays.binarySearch(this.idArray, request.getId()) >= 0;
    }
}
```  

- `OddReceiver`, `RangeReceiver`, `ListReceiver` 는 `Receiver` 의 구현체로 실제 각 처리자를 의미하는 클래스이다.
- `Chain Of Responsibility` 패턴에서 `ConcreteHandler` 역할을 수행한다.
- `OddReceiver` 는 `id` 가 홀수일 경우만 처리, `RangeReceiver` 는 `id` 가 `min`, `max` 범위일 경우만 처리, `ListReceiver` 는 `id` 가 주어진 리스트에 있을 경우에만 처리한다.

### Receiver 의 처리과정
- 처리자 구성
	- `OddReceiver("Odd")`
	- `RangeReceiver("SmallRange", 1, 10)`
	- `RangeReceiver("BigRange", 50, 100)`
	- `ListReceiver("List", 20, 30, 40)`
- 요청
	- `Request(48)`
	
![그림 1]({{site.baseurl}}/img/designpattern/2/concept_chainofresponsibility_3.png)


### 테스트

```java
public class ChainOfResponsibilityTest {

    @Test
    public void ReceiverChain_Resolve() {
        // given
        Receiver oddReceiver = new OddReceiver("Odd");
        Receiver smallRangeReceiver = new RangeReceiver("SmallRange", 1, 10);
        Receiver bigRangeReceiver = new RangeReceiver("BigRange", 50, 100);
        Receiver listReceiver = new ListReceiver("List", 20, 30, 40);
        Request request = new Request(9);
        oddReceiver.setNext(smallRangeReceiver).setNext(bigRangeReceiver).setNext(listReceiver);

        // when
        Response actual = oddReceiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), is(oddReceiver.getName()));
    }

    @Test
    public void ReceiverChain_Fail() {
        // given
        Receiver oddReceiver = new OddReceiver("Odd");
        Receiver smallRangeReceiver = new RangeReceiver("SmallRange", 1, 10);
        Receiver bigRangeReceiver = new RangeReceiver("BigRange", 50, 100);
        Receiver listReceiver = new ListReceiver("List", 20, 30, 40);
        Request request = new Request(12);
        oddReceiver.setNext(smallRangeReceiver).setNext(bigRangeReceiver).setNext(listReceiver);

        // when
        Response actual = oddReceiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), nullValue());
    }

    @Test
    public void OddReceiver_Resolve() {
        // given
        Receiver receiver = new OddReceiver("ResolveOdd");
        Request request = new Request(3);

        // when
        Response actual = receiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), is(receiver.getName()));
    }

    @Test
    public void OddReceiver_Fail() {
        // given
        Receiver receiver = new OddReceiver("FailOdd");
        Request request = new Request(4);

        // when
        Response actual = receiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), nullValue());
    }

    @Test
    public void RangeReceiver_Resolve() {
        // given
        Receiver receiver = new RangeReceiver("ResolveRange", 10, 20);
        Request request = new Request(11);

        // when
        Response actual = receiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), is(receiver.getName()));
    }

    @Test
    public void RangeReceiver_Fail() {
        // given
        Receiver receiver = new RangeReceiver("FailRange", 10, 20);
        Request request = new Request(21);

        // when
        Response actual = receiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), nullValue());
    }

    @Test
    public void ListReceiver_Resolve() {
        // given
        Receiver receiver = new ListReceiver("ResolveList", 10, 20, 30);
        Request request = new Request(10);

        // when
        Response actual = receiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), is(receiver.getName()));
    }

    @Test
    public void ListResolver_Fail() {
        // given
        Receiver receiver = new ListReceiver("FailList", 10, 20, 30);
        Request request = new Request(11);

        // when
        Response actual = receiver.receive(request);

        // then
        assertThat(actual.getId(), is(request.getId()));
        assertThat(actual.getReceiverName(), nullValue());
    }
}
```  

---
## Reference

	