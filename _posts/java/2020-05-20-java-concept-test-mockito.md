--- 
layout: single
classes: wide
title: "[Java 개념] Mockito 프레임워크"
header:
  overlay_image: /img/java-bg.jpg
excerpt: '모의 객체생성으로 테스트 코드작성에 집중을 도와주는 Mockito 프레임웍에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - Test
    - Mockito
toc: true
use_math: true
---  

## Mockito
- `Mockito` 는 `mocking`(모의 환경) 을 기반으로 테스트를 제공하는 프레임워크로 간단하고 쉬운 API 를 통해 테스트를 작성할 수 있다.
- 테스트를 수행하다보면 비지니스 로직상 특정 조건이나 선행되야 하는 작업이 있을 수있다. 이러한 상황은 항상 테스트를 어렵게하는 특징 중 하나이다.
- `Mockito` 를 기반으로 `mocking` 을 하게되면 위와 같은 상황에서 벗어나 좀더 테스트에 집중해서 코드를 작성할 수 있다.
- `Mockito` 는 하나의 방법으로 객체를 `mocking` 하고, 특정 동작을 `stubbing` 할 수 있어, `TDD` 를 기반으로 코드를 작성할 때 편리함과 높은 생산성을 기대할 수 있다.
- `Mockito` 의 특징은 아래와 같다.
	1. 구현 클래스 뿐만아니라 인터페이스 또한 `mocking` 이 가능하다.
	1. `@Mock` 과 같은 손쉬운 `Annotation` 을 제공한다.
	1. 깔끔한 에러, 예외 확인이 가능하다.
	1. 순서에 따라 유연한 검증이 가능하다.
	1. 검증에 있어서 정확한 횟수나 최소, 최대와 같은 다양한 방식을 제공한다.
	1. 메소드 호출 인자에 대해서도 다양한 방식을 제공한다.(`anyObject()`, `anyString()` ..)
	1. 메소드 호출 인자에 대해서 커스텀한 `Matcher` 를 구성하거나, `hamcrest` 를 기반으로도 사용할 수 있다.
	
### 의존성
- [여기](https://mvnrepository.com/artifact/org.mockito/mockito-all)
에서 빌드 도구에 맞는 의존성을 추가할 수 있다.
- `Maven`

	```xml
	<!-- https://mvnrepository.com/artifact/org.mockito/mockito-all -->
	<dependency>
	    <groupId>org.mockito</groupId>
	    <artifactId>mockito-all</artifactId>
	    <version>1.10.19</version>
	    <scope>test</scope>
	</dependency>
	```  
	
- `Gradle`

	```groovy
	// https://mvnrepository.com/artifact/org.mockito/mockito-all
    testCompile group: 'org.mockito', name: 'mockito-all', version: '1.10.19'
	```  

### 관련 용어
- `Mock` : 생성된 모의(`mocking`) 객체를 의미한다.
- `Stub` : `Mock` 객체에서 수행가능한 메소드에 대해서 동작을 지정한다.
- `Verify` : `Mock` 객체의 메소드가 예상대로 호출되었는지 검증한다.

## Mockito 예제
- 원활한 `Mockito API` 사용을 위해 아래와 같이 임포트할 수 있다.

	```java	
	import static org.mockito.Mockito.*;
	```  
	
- 테스트 검증을 위해 추가적으로 [hamcrest](https://mvnrepository.com/artifact/org.hamcrest/hamcrest-all) 라이브러리를 사용했다.
- `Junit4` 를 사용해서 테스트 코드를 작성했다.
	
### Mock 객체 만들고 검증하기

```java
@Test
public void mockObject_호출검증() {
    // given
    List<String> mockList = mock(List.class);

    // when
    mockList.add("a");
    mockList.add("b");

    // then
    verify(mockList).add("a");
    verify(mockList).add("b");
}
```  

- `mock()` 메소드를 사용해서 원하는 `mocking` 객체를 생성할 수 있다.
- `verify()` 를 사용해서 `mocking` 객체에 대한 메소드 호출 검증을 수행할 수 있다.

### 동작 지정하기

```java
@Test
public void mockObject_동작지정_값리턴() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    when(mockList.get(0)).thenReturn("a");
    when(mockList.get(1)).thenReturn("b");
    when(mockList.size()).thenCallRealMethod();

    // then
    assertThat(mockList.get(0), is("a"));
    assertThat(mockList.get(1), is("b"));
    assertThat(mockList.size(), is(0));
    assertThat(mockList.get(99999), nullValue());
    verify(mockList).get(0);
    verify(mockList).get(1);
    verify(mockList).size();
    verify(mockList).get(99999);
}
```  

- `Mock` 객체의 메소드는 기본적으로 값을 리턴하는데 `stub` 이 지정되지 않은 경우 `Mock` 객체는 `null`, 원시타입이나 랩퍼타입은 해당타입의 기본값을 리턴하고, `Collections` 의 경우 빈 컬렉션을 리턴한다.
- `Mockito` 에서 `stub` 은 `when()` 메소드를 통해 지정할 수 있다.
- `stub` 은 오버라이드를 통해 동작을 재지정 할 수 있다. 하지만 많은 오버라이드는 테스트에 대한 가독성을 떨어뜨릴 수 있다.
- 한번 지정된 `stub` 은 해당되는 메소드를 몇번 호출하든지 `stub` 에 지정된 동작을 수행한다.

### 동작으로 예외 지정해서 발생시키기

```java
@Test(expected = RuntimeException.class)
public void mockObject_동작지정_예외발생() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    when(mockList.get(0)).thenThrow(new RuntimeException("my exception"));

    // then
    mockList.get(0);
}
```  

### 메소드 인자값 조건에 따라 매칭시키기

```java
@Test
public void mockObject_호출인자_모두허용() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    when(mockList.get(anyInt())).thenReturn("any");

    // then
    assertThat(mockList.get(0), is("any"));
    assertThat(mockList.get(1), is("any"));
    assertThat(mockList.get(11), is("any"));
    assertThat(mockList.get(111), is("any"));
//        verify(mockList, times(4)).get(anyInt());
    verify(mockList).get(0);
    verify(mockList).get(1);
    verify(mockList).get(11);
    verify(mockList).get(111);
}

@Test
public void mockObject_호출인자_조건사용() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    when(mockList.get(1)).thenReturn("1");
    when(mockList.get(2)).thenReturn("1");
    when(mockList.get(11)).thenReturn("11");
    when(mockList.get(12)).thenReturn("11");
    when(mockList.get(111)).thenReturn("111");
    when(mockList.get(112)).thenReturn("111");

    // then
    assertThat(mockList.get(1), is("1"));
    assertThat(mockList.get(2), is("1"));
    assertThat(mockList.get(11), is("11"));
    assertThat(mockList.get(12), is("11"));
    assertThat(mockList.get(111), is("111"));
    assertThat(mockList.get(112), is("111"));
    // org.mockito.hamcrest.MockitoHamcrest.*;
    verify(mockList, times(2)).get(intThat(lessThanOrEqualTo(2)));
    verify(mockList, times(4)).get(intThat(lessThanOrEqualTo(12)));
    verify(mockList, times(6)).get(intThat(lessThanOrEqualTo(112)));
}

@Test
public void mockObject_호출인자여러개_모두Matcher사용() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    when(mockList.subList(0, 1)).thenReturn(Arrays.asList("a", "b"));

    // then
    assertThat(mockList.subList(0, 1), contains("a", "b"));
    verify(mockList).subList(eq(0), anyInt());
    // 인자값에 Matcher 를 사용해야 할경우, 모든 인자값에 사용해야 한다.
//        verify(mockList).subList(0, anyInt());
}
```  

- 메소드의 인자값은 `stub`, `verify` 에서 커스텀 매칭이나, `hamcrest` 매칭을 사용해서 유연한 사용이 가능하다.
	- [ArgumentMatchers](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/ArgumentMatchers.html)
	- [MockitoHamcrest](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/hamcrest/MockitoHamcrest.html)
- 메소드 인자값에 대한 검증은 추후에 다룰 `ArgumentCaptor` 를 통해서도 가능하다.

### 메소드 동작 검증

```java
@Test
public void mockObject_호출횟수검증() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    when(mockList.get(0)).thenReturn("0");
    when(mockList.get(1)).thenReturn("1");
    when(mockList.get(2)).thenReturn("2");

    // then
    assertThat(mockList.get(0), is("0"));
    assertThat(mockList.get(1), is("1"));
    assertThat(mockList.get(2), is("2"));
    verify(mockList).get(0);
    verify(mockList, times(3)).get(anyInt());
    verify(mockList, atLeast(3)).get(anyInt());
    verify(mockList, atMost(3)).get(anyInt());
    verify(mockList, atMostOnce()).get(0);
    verify(mockList, atLeastOnce()).get(1);
    verify(mockList, never()).size();
}
```  

- `verify()` 의 2번째 인자의 기본값은 `times(1)` 이다.

### 리턴형이 void 인 메소드 예외 발생시키기

```java
@Test(expected = RuntimeException.class)
public void mockObject_동작지정_리턴void형예외발생() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    doThrow(new RuntimeException()).when(mockList).clear();

    // when
    mockList.clear();
}
```  

### 메소드 호출 순서 검증

```java
@Test
public void mockObject_단일객체_순서검증() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);

    // when
    mockList.add("a");
    mockList.add("b");
    mockList.add("c");

    // then
    InOrder inOrder = inOrder(mockList);
    inOrder.verify(mockList).add("a");
    inOrder.verify(mockList).add("b");
    inOrder.verify(mockList).add("c");
}
    
@Test
public void mockObject_여러객체_순서검증() {
    // given
    ArrayList<String> mockList_1 = mock(ArrayList.class);
    ArrayList<String> mockList_2 = mock(ArrayList.class);

    // when
    mockList_1.add("a");
    mockList_2.add("a");
    mockList_1.add("b");
    mockList_2.add("b");

    // then
    InOrder inOrder = inOrder(mockList_1, mockList_2);
    inOrder.verify(mockList_1).add("a");
    inOrder.verify(mockList_2).add("a");
    inOrder.verify(mockList_1).add("b");
    inOrder.verify(mockList_2).add("b");
}
```  

- 유연한 순서 검증을 제공하기 때문에, 모든 메소드에 순서에 대해 검증하기보다는 실제로 테스트가 필요한 부분에 대한 순서 검증을 할 수 있다.

### Mock 객체 메소드의 사용여부 검증

```java
@Test
public void mockObject_실행여부검증_아무것도호출되지않음() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);

    // then
    verifyNoInteractions(mockList);
}

@Test
public void mockObject_실행여부검증_특정호출이후호출되지않음() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);

    // when
    mockList.add("a");

    // then
    verify(mockList).add("a");
    verifyNoMoreInteractions(mockList);
}
```  

- `verifyNoMoreInteractions()` 의 경우 특정 시점 이후 부터 `Mock` 객체의 메소드의 사용여부를 검증할 수 있어서 모든 테스트에 적용하는 경우가 있지만 이는 바람직하지 않다. 
	- 추후 개발로 인해 코드가 변경된 경우 이는 유지보수 측면에서 난해한 테스트 코드가 될 수 있기 때문에 적절한 테스트에서만 사용하는 것을 권장한다.

### @Mock 을 통한 Mock 객체 생성

```java
@Mock
ArrayList<String> mockList;
@Test
public void mockObject_MockAnnotation_객체생성() {
    // given
    MockitoAnnotations.initMocks(this);
    // stubbing
    when(this.mockList.get(0)).thenReturn("a");
    when(this.mockList.get(1)).thenReturn("b");

    // then
    assertThat(this.mockList.get(0), is("a"));
    assertThat(this.mockList.get(1), is("b"));
    verify(this.mockList, times(2)).get(anyInt());
}
```  

- `@Mock` 은 간단하게 `Mock` 객체를 생성할 수 있는 방법이다.
- 테스트 코드 가독성이 높아 진다.
- `MockitoAnnotations.initMocks(this)` 해당 코드 호출해야 정상적으로 `Mock` 객체의 활성화가 가능하다.
- `MockitoJUnitRunner` 나 `MockitoRule` 을 통해 `Mock` 객체의 활성화도 가능하다.
- `Junit5` 이상의 경우 [여기](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html#45)를 참고한다.

### 같은 메소드 연속호출 동작 지정하기

```java
@Test
public void mockObject_동작지정_연속호출() {
    // given
    LinkedList<String> mockList = mock(LinkedList.class);
    // stubbing
    when(mockList.getFirst())
            .thenReturn("a")
            .thenReturn("b")
            .thenReturn("c")
    ;

    // then
    assertThat(mockList.getFirst(), is("a"));
    assertThat(mockList.getFirst(), is("b"));
    assertThat(mockList.getFirst(), is("c"));
    assertThat(mockList.getFirst(), is("c"));
    verify(mockList, times(4)).getFirst();
}

@Test
public void mockObject_동작지정_연속호출요약() {
    // given
    LinkedList<String> mockList = mock(LinkedList.class);
    // stubbing
    when(mockList.getFirst())
        .thenReturn("a", "b", "c");

    // then
    assertThat(mockList.getFirst(), is("a"));
    assertThat(mockList.getFirst(), is("b"));
    assertThat(mockList.getFirst(), is("c"));
    assertThat(mockList.getFirst(), is("c"));
    verify(mockList, times(4)).getFirst();
}
```  

- 구현된 코드의 구조상 같은 메소드에 같은 인자에 대해서 호출 횟수에 따라서 리턴 값이 다른 경우가 있다.
	- ex) `Iterator`, ...
- `stub` 을 사용할때 리턴 구문을 `chaining` 을 통해 구성하거나, 여러개의 값을 넣어주면 된다.


### 메소드 동작 콜백을 통해 지정하기

```java
@Test
public void mockObject_Callback사용해서_동작지정() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    when(mockList.get(0)).thenAnswer(
            new Answer<String>(){
                @Override
                public String answer(InvocationOnMock invocation) throws Throwable {
                    Object[] args = invocation.getArguments();
                    Object mock = invocation.getMock();
                    return "answer args is " + Arrays.toString(args);
                }
            }
    );

    // when
    assertThat(mockList.get(0), is("answer args is [0]"));
    verify(mockList).get(0);
}
```   

- `stub` 에 대한 구문은 `thenReturn()`, `thenThrow()` 와 같은 간단한 구문을 권장한다.
- 특정 동작에 대한 처리가 필요한 경우 `thenAnswer()` 을 콜백을 통해 사용할 수 있다.

### doReturn(), doThrow(), doAnswer(), doNothing(), doCallRealMethod()

```java
@Test(expected = RuntimeException.class)
public void mockObject_동작지정_리턴void형예외발생() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    // stubbing
    doThrow(new RuntimeException()).when(mockList).clear();

    // when
    mockList.clear();
}
```  

- 위 5가지 메소드는 아래와 같은 경우에 사용 가능하다.
	- 리턴형이 `void` 인 경우
	- 추후에 다룰 `spy` 객체
	- 테스트 중  `stub` 의 동작을 변경하는 경우
- 자세한 설명은 아래 링크에서 확인 가능하다.
	- [doReturn(Object)](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html#doReturn-java.lang.Object-)
	- [doThrow(Throwable ...)](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html#doThrow-java.lang.Throwable...-)
	- [doThrow(Class)](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html#doThrow-java.lang.Class-)
	- [doAnswer(Answer)](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html#doAnswer-org.mockito.stubbing.Answer-)
	- [doNothing()](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html#doNothing--)
	- [doCallRealMethod()](https://javadoc.io/static/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html#doCallRealMethod--)

### 실제 객체와 Mock 객체의 역할을 모두 수행하는 Spy

```java
@Test
public void mockObject_실제객체를통해생성_mockObject처럼사용가능() {
    // given
    ArrayList<String> list = new ArrayList<>();
    ArrayList<String> spyList = spy(list);
    // stubbing (특정 메소드에 대해서 stub 가능)
    when(spyList.size()).thenReturn(999);

    // when
    spyList.add("a");
    spyList.add("b");

    // then
    assertThat(spyList.get(0), is("a"));
    assertThat(spyList.get(1), is("b"));
    assertThat(spyList.size(), is(999));
    verify(spyList).get(0);
    verify(spyList).get(1);
    verify(spyList).size();
}
```  

- `spy` 는 `mocking` 하려는 객체의 부분적인 `mocking` 을 하는 개념이다.
- 실제 객체의 인스턴스로 생성하고, `stub` 이 지정되지 않은 메소드는 실제 객체의 메소드로 동작한다.

### ArgumentCaptor 로 호출된 메소드 인자값 검증하기

```java
@Test
public void mockObject_호출메소드파라미터검증() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    ArgumentCaptor<Integer> argumentCaptor = ArgumentCaptor.forClass(Integer.class);
    // stubbing
    when(mockList.get(0)).thenReturn("a");

    // then
    assertThat(mockList.get(0), is("a"));
    verify(mockList).get(argumentCaptor.capture());
    assertThat(0, is(argumentCaptor.getValue()));
}
```  

- 테스트 검증 과정에서 `ArgumentCaptor` 를 사용해서 호출된 메소드의 인자값에 대해 다양한 방식으로 검증을 수행할 수 있다.
- 이러한 방식으로 통해 인자값에 대한 검증을 좀더 깔끔하고, 쉽게 수행할 수 있어 추천하는 방식이다.
- `ArgumentCaptor` 는 호출된 메소드의 인자값에 대한 검증을 하는 역할이기 때문에, `stub` 의 인자값에는 사용하지 않는게 좋다.
	- `ArgumentCaptor` 의 검증 부분은 외부에서 별도로 검증문을 작성하는 방식이기 때문에, `stub` 사용에 있어서는 가독성을 떨어뜨릴 수 있다.
- `ArgumentCaptor` 비슷한 기능을 하는 `ArgumentMatcher` 와 비교될 수 있지만, 호출된 메소드의 인자 검증에는 `ArgumentCaptor` 가 적합하고 `verify` 에는 `ArgumentMatcher` 가 적합하다.
- 1.8 버전부터 지원한다.

### 실제 객체의 메소드 호출하기

```java
@Test
public void mockObject_특정메소드_실제호출() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    when(mockList.get(0)).thenReturn("a");
    when(mockList.size()).thenCallRealMethod();

    // then
    assertThat(mockList.get(0), is("a"));
    assertThat(mockList.size(), is(0));
    verify(mockList).get(0);
    verify(mockList).size();
}
```  

- 객체에 대해서 `mocking` 을 부분적으로 적용하는 방법은 앞서 설명한 `spy` 가 있다. 같은 기능을 제공하지만 좀 더 간단하고 명시적인 사용을 위해 `partial mocking` 이 있다.
- `Mock` 객체를 생성하는 것처럼 생성후, `stub` 을 할때 실제 메소드를 호출할 부분에서 `thenCallRealMethod()` 를 지정해 주면된다.
- 1.8 버전부터 지원한다.

### Mock 객체 리셋

```java
@Test
public void mockObject_설정리셋() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    when(mockList.get(0)).thenReturn("a");
    when(mockList.get(1)).thenReturn("b");

    // when
    reset(mockList);

     // then
    assertThat(mockList.get(0), nullValue());
    assertThat(mockList.get(1), nullValue());
}
```  

- 테스트를 위해 다양한 다양한 설정을 한 `Mock` 객체는 특정 시점에 깔끔하게 리셋을 해야하는 경우가 있다면 `reset()` 을 통해 이를 수행하 수 있다.
- `reset()` 을 사용하기 전에 너무 많은 행동을 한개의 테스트에서 검증하려 하는건 아닌지 등의 수행하려는 테스트에 대해서 한번 더 고민이 필요할 수 있다. 
- 1.8 버전부터 지원한다.


### 메소드 타임아웃 검증

```java
@Test
public void mockObject_타임아웃_검증() {
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    when(mockList.get(0)).thenReturn("a");
    when(mockList.get(1)).thenReturn("b");

    // then
    assertThat(mockList.get(0), is("a"));
    assertThat(mockList.get(1), is("b"));
    verify(mockList, timeout(5).times(2)).get(anyInt());
}
```  

- 메소드가 실행될 때 소요되는 타임아웃 시간을 검증 할 수 있다.
- 동시성을 가진 테스트에서 유용할 수있다.

### BDD 스타일 테스트

```java
@Test
public void mockObject_BDD형식테스트() {
    // org.mockito.BDDMockito
    // given
    ArrayList<String> mockList = mock(ArrayList.class);
    given(mockList.get(0)).willReturn("a");

    // when
    String actual = mockList.get(0);

    // then
    then(mockList).should().get(0);
    assertThat(actual, is("a"));
}
```  

- `org.mockito.BDDMockito` 패키지를 사용해서 `BDD(Behavior Driven Development)` 스타일의 테스트를 할 수 있다.

### @Capture

```java
@Captor
ArgumentCaptor<Integer> captor;

@Test
public void mockObject_CaptureAnnotation() {
    // given
    MockitoAnnotations.initMocks(this);
    ArrayList<String> mockList = mock(ArrayList.class);
    when(mockList.get(0)).thenReturn("a");
    when(mockList.get(1)).thenReturn("b");

    // then
    assertThat(mockList.get(0), is("a"));
    assertThat(mockList.get(1), is("b"));
    verify(mockList, times(2)).get(this.captor.capture());
    assertThat(this.captor.getAllValues(), containsInAnyOrder(0, 1));
}
```  

- `ArgumentCaptor` 의 객체를 자동설정 해준다.
- 사용을 위해서는 `MockitoAnnotations.initMocks()` 과 같은 `@Mock` 과 동일 한 설정이 필요하다.

### @Spy

```java
@Spy
ArrayList<String> spyList;

@Test
public void mockObject_SpyAnnotation_실제객체생성() {
    // given
    MockitoAnnotations.initMocks(this);
    this.spyList.add("a");
    when(this.spyList.size()).thenReturn(999);

    // then
    assertThat(this.spyList.get(0), is("a"));
    assertThat(this.spyList, hasSize(999));
    verify(this.spyList).get(0);
    verify(this.spyList).size();
}
```  

- `spy()` 대신 사용할 수 있다.
- 사용을 위해서는 `MockitoAnnotations.initMocks()` 과 같은 `@Mock` 과 동일 한 설정이 필요하다.

### @InjectMock

```java
class PlusOperation {
    public int execute(int a, int b) {
        return a + b;
    }
}
class MinusOperation {
    public int execute(int a, int b) {
        return a - b;
    }
}
class Calculator {
    private PlusOperation plus;
    private MinusOperation minus;

    public Calculator(PlusOperation plus, MinusOperation minus) {
        this.plus = plus;
        this.minus = minus;
    }

    // or setter
    // getter


    public PlusOperation getPlus() {
        return plus;
    }

    public MinusOperation getMinus() {
        return minus;
    }
}

@Mock
PlusOperation plus;
@Mock
MinusOperation minus;
@InjectMocks
Calculator calculator;
@Test
public void mockObject_InjectMocksAnnotation_자동주입() {
    // given
    MockitoAnnotations.initMocks(this);
    when(this.plus.execute(1, 1)).thenReturn(11);
    when(this.minus.execute(2, 2)).thenReturn(22);

    // when
    assertThat(this.calculator.getPlus().execute(1, 1), is(11));
    assertThat(this.calculator.getMinus().execute(2,2), is(22));
    verify(this.plus).execute(1, 1);
    verify(this.minus).execute(2, 2);
}

```  

- 객체의 필드에 사용자가 정의한 객체가 있을 때 `@InjectMock` 을 사용하면 쉽고 한번에 `Mock` 혹은 `Spy` 객체를 설정할 수 있다.


---
## Reference
[Features And Motivations](https://github.com/mockito/mockito/wiki/Features-And-Motivations)  
[Mockito (Mockito 3.1.0 API)](https://javadoc.io/doc/org.mockito/mockito-core/3.1.0/org/mockito/Mockito.html)  