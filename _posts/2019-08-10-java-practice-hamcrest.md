--- 
layout: single
classes: wide
title: "[Java 실습] Hamcrest"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Hamcrest 라이브러리를 사용해서 Java TDD 의 가독성을 높여보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Practice
    - Java
    - Junit
    - Hamcrest
    - TDD
    - Test
toc: true
---  

## Hamcrest 란
- 소프트웨어 테스트를 위한 Framework 이다.
- [Junit]({{site.baseurl}}{% link _posts/2019-08-01-java-practice-junit.md %}) 과 Mockito 와 연계해서 사용가능하다.
- `Matcher` 클래스를 통해 단위 테스트를 수행 결과를 판별한다.
- `assertThat` 을 사용해서 검증을 수행한다.
- Junit 의 검증 문과 Hamcrest 의 검증 문을 비교하면 아래와 같다.
	- Junit
		
		```java
		assertEquals(expected, actual);
		```  
		
	- Hamcrest
	
		```java
		assertThat(actual, is(eqaulTo(expected)));
		```  
		
- 위의 비교에서 알수 있듯이 Junit 을 사용하는 것보다 검증 문의 가독성이 향상된다.
- Hamcrest 에서는 `assertThat()` 을 사용하는데, 테스트 실패 시 보여 줄 메시지를 명시할 수 있다.

	```java
	assertThat("need check third party library", actual, expected);
	```  
	
- Hamcrest 를 사용하기 위해 아래와 같이 Maven 혹은 Gradle 에 의존성을 추가해 주면 된다.

	```xml
	<dependency>
		<groupId>org.hamcrest</groupId>
		<artifactId>hamcrest-all</artifactId>
		<version>1.3</version>
  	</dependency>
	```  
	
	```groovy
	dependencies {
		testCompile "org.hamcrest:hamcrest-all:1.3"
  	}
	```  
	
- Hamcrest 를 사용할 때는 주로 2개의 패키지를 static 으로 import 하고 사용한다.

	```java
	// hamcrest 의 assert 문 (assertThat())
	import static org.hamcrest.MatcherAssert.*;
	// hamcrest 의 Mactchers 이하의 함수들
	import static org.hamcrest.Matchers.*;
	```  
	
### hamcrest 패키지

package|desc
---|---
org.hamcrest.core|Object 또는 Value 관련 기본적인 Matcher
org.hamcrest.beans|Java Bean 과 프로퍼티 관련된 Matcher
org.hamcrest.collection|Array 와 Collection 관련 Matcher
org.hamcrest.number|숫자 관련 Matcher
org.hamcrest.object|Object 와 Class 관련 Matcher
org.hamcrest.text|문자열 관련 Matcher
org.hamcrest.xml|XML 관련 Matcher

## Core 
### anything()
- 어떤 오브젝트나 값이 사용되든 일치한다고 판별한다.

```java
@Test
public void object_Expected_Anything() {
	// given
	Object actual = new Object();

	// then
	assertThat(actual, is(anything()));

	// given
	actual = null;

	// then
	assertThat(actual, is(anything()));
}

@Test
public void value_Expected_Anything() {
	// given
	int intActual = 1;

	// then
	assertThat(intActual, is(anything()));

	// given
	double doubleActual = 0d;

	// then
	assertThat(doubleActual, is(anything()));
}
```  

### describedAs()
- 테스트 실패 시 보여 줄 추가적인 메시지를 다양한 방법으로 표현 할 수 있다.

```java
@Test
public void value_Expected_DescribedAsIs() {
	int actual = 111;
	int expected = 1121;

	assertThat(actual, describedAs("%0 need to equal %1", is(expected), actual, expected));
}
```  

- 아래와 같은 메시지로 출력된다.

```
java.lang.AssertionError: 
Expected: <111> need to equal <1121>
     but: was <111>
Expected :<111> need to equal <1121>
     
Actual   :<111>
```  

### is()
- actual 값이 Matcher 에 만족하는지 검사한다.
- 내부적으론 `equalTo` 와 동일하다.
- `assertThat()` 의 검증 문을 보다 가독성 있게 하는데 사용된다.
- `is()` 의 인자 값으로는 오브젝트, 값, Matcher 를 사용 할 수 있다.

```java
@Test
public void object_Expected_Is() {
	// given
	int intActual = 1111;

	// then
	assertThat(intActual, is(1111));

	// given
	double doubleActual = 11.22d;

	// then
	assertThat(doubleActual, is(equalTo(11.22d)));

	// given
	String strActual = "myString";

	// then
	assertThat(strActual, is("myString"));
}
```

## Logical
### allOf()
- 인자 값으로 사용된 모든 Matcher 를 만족하는지 검사한다. (`&&`)

```java
@Test
public void object_Expected_AllOf_Matcher() {
	// given
	int actual = 10;

	// then
	assertThat(actual, allOf(greaterThan(0), lessThan(100)));

	// given
	String strActual = "abcdef";

	// then
	assertThat(strActual, allOf(startsWith("ab"), endsWith("ef")));
}

@Test
public void object_Expected_AllOf_IterableMatcher() {
	// given
	String strActual = "abcdef";

	List<Matcher<? super String>> matchers = Arrays.asList(startsWith("ab"), endsWith("ef"));

	// then
	assertThat(strActual, allOf(matchers));
}
```  


### anyOf()
- 인자 값으로 사용 된 Matcher 중 하나를 만족하는지 검사한다. (`||`)

```java
// anyOf
@Test
public void object_Expected_AnyOf_Matcher() {
	// given
	int actual = 10;

	// then
	assertThat(actual, anyOf(is(10), is(100)));

	// given
	String strActual = "abcdef";

	// then
	assertThat(strActual, anyOf(startsWith("abbbbb"), endsWith("ef")));
}

// anyOf
@Test
public void object_Expected_AnyOf_IterableMatcher() {
	// given
	String strActual = "abcdef";

	List<Matcher<? super String>> matchers = Arrays.asList(startsWith("abbbbb"), endsWith("ef"));

	// then
	assertThat(strActual, anyOf(matchers));
}
```  

### not()
- Matcher 를 만족하지 않는지 검사한다.

```java
@Test
public void object_Expected_Not() {
	// given
	int actual = 10;

	// then
	assertThat(actual, not(100));
}

@Test
public void object_Expected_NotMatcher() {
	// given
	int actual = 10;

	// then
	assertThat(actual, not(greaterThan(100)));
}
```  

## Object, Value
### equalTo()
- 오브젝트, 값, 배열이 동일한지 검사한다. (Object.equals)

```java
@Test
public void object_Expected_EqualTo() {
	// given
	String actual = "myString~~";

	// then
	assertThat(actual, equalTo("myString~~"));

	// given
	actual = new String("myString~~");

	// then
	assertThat(actual, equalTo(new String("myString~~")));
}

@Test
public void array_Expected_EqualTo() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};
	String[] expected = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, equalTo(expected));

	// given
	actual = new String[]{"cc", "ab", "b", "cd", "abc"};
	expected = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, not(equalTo(expected)));

	// given
	actual = new String[]{"ab", "b", "cd", "abc"};
	expected = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, not(equalTo(expected)));
}
```  

### greaterThan(), greaterThanOrEqualTo(), lessThan(), lessThanOrEqualTo()
- 값이 큰지, 같거나 큰지, 작은지, 같거나 작은지 검사한다.

```java
@Test
public void value_Expected_Range() {
	// given
	int actual = 120;

	// then
	assertThat(actual, greaterThan(119));
	assertThat(actual, greaterThanOrEqualTo(120));
	assertThat(actual, lessThan(121));
	assertThat(actual, lessThanOrEqualTo(120));
}

@Test
public void object_Expected_Range() {
	// given
	String actual = "efg";

	// then
	assertThat(actual, greaterThan("a"));
	assertThat(actual, greaterThanOrEqualTo("efg"));
	assertThat(actual, lessThan("tgvg"));
	assertThat(actual, lessThanOrEqualTo("efg"));
}
```

### closeTo()
- Double, BigDecimal 의 값이 오차 범위를 포함해서 Matcher 에 만족하는 지 검사한다.

```java
@Test
public void double_Expected_CloseTo() {
	// given
	double actual = 11.123d;

	// then
	assertThat(actual, closeTo(11, 0.2d));
}

@Test
public void bigDecimal_Expected_CloseTo() {
	// given
	BigDecimal actual = new BigDecimal(1234.567);

	// then
	assertThat(actual, closeTo(new BigDecimal(1200), new BigDecimal(35)));
}
```  

### hasToString()
- 오브젝트의 문자열이 동일한지 검사한다.(Object.toString)

```java
@Test
public void int_Expected_EqualToString() {
	// given
	int actual = 1234567;

	// then
	assertThat(actual, hasToString("1234567"));
}
```  

### instanceOf(), isCompatibleType()
- 두 오브젝트의 타입이 같은지 검사한다.

```java
@Test
public void object_Expected_InstanceOf_String() {
	// given
	String actual = "string~~";

	// then
	assertThat(actual, instanceOf(String.class));
}

@Test
public void object_Expected_InstanceOf_NotString() {
	// given
	String actual = null;

	// then
	assertThat(actual, not(instanceOf(String.class)));
}
```  

### notNullValue()
- 오브젝트가 null 값이 아닌지 검사한다.

```java
@Test
public void object_Expected_NotNull() {
	// given
	String actual = "myString ~~~~";

	// then
	assertThat(actual, notNullValue());
}
```  

### nullValue()
- 오브젝트가 null 값 인지 검사한다.

```java
@Test
public void object_Expected_Null() {
	// given
	String actual = null;

	// then
	assertThat(actual, nullValue());
}
```  

## sameInstance(), theInstance()
- 오브젝트의 인스턴스가 동일한지 검사한다.

```java
@Test
public void object_Expected_SameInstance() {
	// given
	String actual = "myString ~~~";

	// then
	assertThat(actual, sameInstance("myString ~~~"));
}

@Test
public void object_Expected_NotSameInstance() {
	// given
	String actual = new String("myString");

	// then
	assertThat(actual, not(sameInstance(new String("myString"))));
}
```  

### isOneOf()
- 인자 값의 값 중 하나와 일치하는지 검사한다.

```java
@Test
public void object_Expected_IsOneOf() {
	// given
	int actual = 10;

	// then
	assertThat(actual, isOneOf(1, 2, 3, 10));
}
```  

### isIn()
- Collection 의 값과 일치한지 검사한다.

```java
@Test
public void object_Expected_IsIn() {
	// given
	String actual = "abc";

	// then
	assertThat(actual, isIn(new ArrayList<>(Arrays.asList("a", "b", "abc"))));
}
```  

## Beans
### hasProperty()
- 빈에 프로퍼티가 있는지와 빈의 프로퍼티의 값이 Matcher 를 만족하는지 검사한다.

```java
@Test
public void bean_Expected_HasProperty() {
	// given
	JTextField actual = new JTextField();
	actual.setText("myString");

	// then
	assertThat(actual, hasProperty("text"));
}

@Test
public void bean_Expected_HasPropertyAndValue() {
	// given
	JTextField actual = new JTextField();
	actual.setText("myString");

	// then
	assertThat(actual, hasProperty("text", is("myString")));
}
```  

### samePropertyValuesAs()
- 두 빈의 프로퍼티와 값이 모두 동일한지 검사한다.

```java
public class MyBean {
    private Integer integer;
    private String text;

    public Integer getInteger() {
        return integer;
    }

    public void setInteger(Integer integer) {
        this.integer = integer;
    }

    public String getText() {
        return text;
    }

    public void setText(String text) {
        this.text = text;
    }
}
```  

```java
@Test
public void bean_Expected_SamePropertyValue() {
	// given
	MyBean actual = new MyBean();
	actual.setInteger(10);
	actual.setText("myText");

	MyBean expected = new MyBean();
	expected.setInteger(10);
	expected.setText("myText");

	// then
	assertThat(actual, samePropertyValuesAs(expected));
}
```  

## Collections, Array, Iterable
### hasEntry()
- Map 에서 Macher 에 만족하는 Entry 가 있는지 검사한다.

```java
@Test
public void map_Expected_HasEntry() {
	// given
	HashMap<String, Integer> actual = new HashMap<>();
	actual.put("ab", 1);
	actual.put("cd", 2);

	// then
	assertThat(actual, hasEntry("ab", 1));
	assertThat(actual, hasEntry("cd", 2));
}

@Test
public void map_Expected_HashEntryMatcher() {
	// given
	HashMap<String, Integer> actual = new HashMap<>();
	actual.put("ab", 1);
	actual.put("cd", 2);

	// then
	assertThat(actual, hasEntry(startsWith("a"), lessThan(2)));
	assertThat(actual, hasEntry(endsWith("d"), greaterThanOrEqualTo(2)));
}
```  

### hasKey()
- Map 에서 Matcher 를 만족하는 Key 가 있는지 검사한다.

```java
@Test
public void map_Expected_HashKey() {
	// given
	HashMap<String, Integer> actual = new HashMap<>();
	actual.put("ab", 1);
	actual.put("cd", 2);

	// then
	assertThat(actual, hasKey("ab"));
	assertThat(actual, hasKey("cd"));
}

@Test
public void map_Expected_HashKeyMatcher() {
	// given
	HashMap<String, Integer> actual = new HashMap<>();
	actual.put("ab", 1);
	actual.put("cd", 2);

	// then
	assertThat(actual, hasKey(containsString("a")));
	assertThat(actual, hasKey(startsWith("c")));
}
```  

### hasValue()
- Map 에서 Matcher 를 만족하는 Value 가 있는지 검사한다.

```java
@Test
public void map_Expected_HasValue() {
	// given
	HashMap<String, Integer> actual = new HashMap<>();
	actual.put("ab", 1);
	actual.put("cd", 2);

	// then
	assertThat(actual, hasValue(1));
	assertThat(actual, hasValue(2));
}

@Test
public void map_Expected_HasValueMatcher() {
	// given
	HashMap<String, Integer> actual = new HashMap<>();
	actual.put("ab", 1);
	actual.put("cd", 2);

	// then
	assertThat(actual, hasValue(lessThan(2)));
	assertThat(actual, hasValue(greaterThanOrEqualTo(2)));
}
```  

### hasItem()
- Iterable 에서 Matcher 를 만족하는 값이 있는지 검사한다.

```java
@Test
public void iterable_Expected_HasItem() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, hasItem("a"));
	assertThat(actual, hasItem("abc"));
}

@Test
public void iterable_Expected_HasItemMatcher() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, hasItem(startsWith("a")));
	assertThat(actual, hasItem(endsWith("c")));
}
```  

### hasItems()
- Iterable 에서 Matcher 들을 만족하는 값이 있는지 검사한다.

```java
@Test
public void iterable_Expected_HashItems() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, hasItems("a", "abc"));
}

@Test
public void iterable_Expected_HasItemsMatcher() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, hasItems(startsWith("a"), endsWith("c")));
}
```  

### everyItem()
- Iterable 의 값들이 모두 Matcher 를 만족하는지 검사한다.

```java
@Test
public void iterable_Expected_EveryItemMatched() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "abc", "abcd", "abcde");

	// then
	assertThat(actual, everyItem(startsWith("a")));
	assertThat(actual, everyItem(containsString("a")));
	assertThat(actual, everyItem(not(containsString("f"))));
}
```  

### contains()
- Iterable 에서 각 인덱스에 대응되는 Matcher 를 만족하는지 검사한다.

```java
@Test
public void iterable_Expected_Contains() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, contains("a", "ab", "b", "cd", "abc"));
}

@Test
public void iterable_Expected_ContainsMatcher() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, contains(is("a"), endsWith("b"), notNullValue(), startsWith("c"), startsWith("ab")));
}
```  

### containsInAnyOrder()
- Iterable 의 모든 값이 하나씩의 Matcher 를 각각 만족하는지 검사한다.

```java
@Test
public void iterable_Expected_ContainsInAnyOrder() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, containsInAnyOrder("abc", "cd", "b", "ab", "a"));
}

@Test
public void iterable_Expected_ContainsInAnyOrderMatcher() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, containsInAnyOrder(endsWith("bc"), startsWith("c"), startsWith("b"), endsWith("ab"), is("a")));
}
```  

### emptyIterable()
- Iterable 이 비어있는지 검사한다.

```java
@Test
public void iterable_Expected_EmptyIterable() {
	// given
	Set<String> actual = new HashSet<>();

	// then
	assertThat(actual, emptyIterable());
}
```  

### emptyIterableOf()
- Iterable 이 비었는지와 타입을 검사한다.

```java
@Test
public void iterable_Expected_EmptyIterableOf() {
	// given
	Set<String> actual = new HashSet<>();

	// then
	assertThat(actual, emptyIterableOf(String.class));
}
```  

### iterableWithSize()
- Iterable 의 크기가 Matcher 를 만족하는지 검사한다.

```java
@Test
public void iterable_Expected_IterableWithSize() {
	// given
	Set<String> actual = new HashSet<>();
	actual.add("a");
	actual.add("b");

	// then
	assertThat(actual, iterableWithSize(2));
}

@Test
public void iterable_Expected_IterableWithSizeMatcher() {
	// given
	Set<String> actual = new HashSet<>();
	actual.add("a");
	actual.add("b");

	// then
	assertThat(actual, iterableWithSize(lessThan(3)));
}
```  

### hasSize()
- Collection 의 크기가 Matcher 에 만족하는지 검사한다.

```java
@Test
public void collection_Expected_HasSize() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "abc", "abcd", "abcde");

	// then
	assertThat(actual, hasSize(5));
}

@Test
public void collection_Expected_HasSizeMatcher() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "abc", "abcd", "abcde");

	// then
	assertThat(actual, hasSize(greaterThan(4)));
}
```  

### empty()
- Collection 이 비어있는지 검사한다.

```java
@Test
public void collection_Expected_Empty() {
	// given
	List<String> actual = Arrays.asList("a", "ab", "b", "cd", "abc");

	// then
	assertThat(actual, not(empty()));

	// given
	actual = new ArrayList<>();

	// then
	assertThat(actual, empty());
}
```  

### emptyCollectionOf()
- Collection 이 비었는지와 타입을 검사한다.

```java
@Test
public void collection_Expected_EmptyCollectionOf() {
	// given
	List<String> actual = new ArrayList<>();

	// then
	assertThat(actual, emptyCollectionOf(String.class));
}
```  

### array()
- 배열에서 각 인덱스에 대응되는 Matcher 를 만족하는지 검사한다.

```java
@Test
public void array_Expected_ArrayMatched() {
	// given
	String[] actual = new String[]{"one", "two", "three", "four"};

	// then
	assertThat(actual, array(startsWith("on"), containsString("w"), instanceOf(String.class), endsWith("our")));
}
```  

### arrayContaining()
- 배열에서 각 인덱스에 대응되는 Matcher 를 만족하는지 검사한다.

```java
@Test
public void array_Expected_arrayContaining() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, arrayContaining("a", "ab", "b", "cd", "abc"));
}

@Test
public void array_Expected_arrayContainingMatcher() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, arrayContaining(is("a"), endsWith("b"), notNullValue(), startsWith("c"), startsWith("ab")));
}
```  

### arrayContainingInAnyOrder()
- 배열에서 모든 값이 하나씩의 Matcher 를 각각 만족하는지 검사한다.

```java
@Test
public void array_Expected_arrayContainingInAnyOrder() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, arrayContainingInAnyOrder("abc", "cd", "b", "ab", "a"));
}

@Test
public void array_Expected_arrayContainingInAnyOrderMatcher() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, arrayContainingInAnyOrder(endsWith("bc"), startsWith("c"), startsWith("b"), endsWith("ab"), is("a")));
}
```  

### hasItemInArray()
- 배열에서 Matcher 를 만족하는 값이 있는지 검사한다.

```java
@Test
public void array_Expected_HasItemInArray() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, hasItemInArray("a"));
	assertThat(actual, hasItemInArray("abc"));
}

@Test
public void array_Expected_HasItemInArrayMatcher() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, hasItemInArray(startsWith("a")));
	assertThat(actual, hasItemInArray(endsWith("c")));
}
```  

### arrayWithSize()
- 배열의 크기가 Matcher 를 만족하는지 검사한다.

```java
@Test
public void array_Expected_ArrayWithSize() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, arrayWithSize(5));
}

@Test
public void array_Expected_ArrayWithSizeMatcher() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, arrayWithSize(greaterThan(4)));
}
```  

### emptyArray()
- 배열이 비었는지 검사한다.

```java
@Test
public void array_Expected_EmptyArray() {
	// given
	String[] actual = new String[]{"a", "ab", "b", "cd", "abc"};

	// then
	assertThat(actual, not(emptyArray()));

	// given
	actual = new String[]{};

	// then
	assertThat(actual, emptyArray());
}
```  

## String
### equalToIgnoringCase()
- 문자열을 대소문자 상관없이 같은지 검사한다.

```java
@Test
public void string_Expected_EqualToIgnoringCase() {
	// given
	String actual = "myString";

	// then
	assertThat(actual, equalToIgnoringCase("MYsTring"));
}
```  

### equalToIgnoringWhiteSpace()
- 문자열을 공백 상관없이 같은지 검사한다.

```java
@Test
public void string_Expected_EqualToIgnoringWhiteSpace() {
	// given
	String actual = "my String";

	// then
	assertThat(actual, equalToIgnoringWhiteSpace("    my    String   "));
}

@Test
public void string_Expected_NotEqualToIgnoringWhiteSpace() {
	// given
	String actual = "my String";

	// then
	assertThat(actual, not(equalToIgnoringWhiteSpace("    m      y    String   ")));
}
```  

### stringContainsInOrder()
- 문자열에서 부분문자열이 순서대로 매칭되는지 검사한다.

```java
@Test
public void string_Expected_StringContainsOrder() {
	// given
	String actual = "one is one, two is two, three is three, four is four";

	// then
	assertThat(actual, stringContainsInOrder(Arrays.asList("one", "two", "three", "four")));
}
```  

### containsString(), endsWith(), startsWith()
- 문자열에서 특정 문자열을 포함하는지, 끝나는지, 시작하는지 검사한다.

```java
@Test
public void string_Expected_SubString() {
	// given
	String actual = "one is one, two is two, three is three, four is four";

	// then
	assertThat(actual, startsWith("one"));
	assertThat(actual, endsWith("four"));
	assertThat(actual, containsString("two is two"));
}
```  

### isEmptyOrNullString()
- 문자열이 null 이거나, 비었는지 검사한다.

```java
@Test
public void string_Expected_IsEmptyOrNullString() {
	// given
	String actual = "";

	// then
	assertThat(actual, isEmptyOrNullString());

	// given
	actual = null;

	// then
	assertThat(actual, isEmptyOrNullString());
}
```  

### isEmptyString()
- 문자열이 빈문자열인지 검사한다.

```java
@Test
public void string_Expected_IsEmptyString() {
	// given
	String actual = "";

	// then
	assertThat(actual, isEmptyString());
}
```  

---
## Reference
[Hamcrest Tutorial](http://hamcrest.org/JavaHamcrest/tutorial)  
[Class Matchers](http://hamcrest.org/JavaHamcrest/javadoc/1.3/org/hamcrest/Matchers.html)  
[Hamcrest matchers tutorial](https://www.javacodegeeks.com/2015/11/hamcrest-matchers-tutorial.html)  
[Using Hamcrest for testing - Tutorial](https://www.vogella.com/tutorials/Hamcrest/article.html)  
[hamcrest 라이브러리](http://blog.naver.com/PostView.nhn?blogId=simpolor&logNo=221289242597&categoryNo=166&parentCategoryNo=0&viewDate=&currentPage=1&postListTopCurrentPage=1&from=postView)  
[hamcrest 로 가독성있는 jUnit Test Case 만들기](https://www.lesstif.com/pages/viewpage.action?pageId=18219426)  
[[Hamcrest] Using Hamcrest for testing](https://coronasdk.tistory.com/918)  
