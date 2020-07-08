--- 
layout: single
classes: wide
title: "[Spring 실습] Junit, TestNG 를 통해 단위 테스트 하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Junit 과 TestNG 를 사용해서 프로젝트의 단위 테스트를 진행해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - spring-test 
    - Junit
    - A-Test  
---  

# 목표
- 자동화된 테스트를 작성해서 Java Application 이 제대로 동작하는지 반복적으로 검증한다.

# 방법
- Junit 과 TestNG 는 Java Platform 에서 유명한 Test Framework 이다.
- 모두 테스트할 메서드에 @Test 를 붙여 public 메서드를 테스트 케이스로 실행한다.
	
# 예제
- 이자를 계산해주는 프로그램을 만든다.
- 아래와 같은 인터페이스가 있다.

```java
public interface IntersestCalculator {
	void setRate(double rate);
	double calculate(double amound, double year);
}
```  

- 위 인터페이스를 구현한 이자를 계산하는 클래스 이다.

```java
public class SimpleInterestCalculator implements InterestCalculator {
	private double rate;
	
	@Override
	public void setRate(double rate) {
		this.rate = rate;
	}
	
	@Override
	public double calculatr(double amount, double year) {
		if(amount < 0 || year < 0) {
			throw new IllegalArgumentException("Amount or year must be positive");
		}
		return amount * year * this.rate;
	}
}
```  

- 위 이자를 계산하는 클래스를 Junit 과 TestNG 를 사용해서 유닛 테스트를 해본다.

## Junit
- 테스트 케이스는 public 메서드에 @Test 를 붙인 메서드가 된다.
- 테스트 데이터는 @BeforeClass  static 메서드에 설정하고 테스트가 끝나면 @AfterClass static 메서드에서 리소스를 해제한다.
	- @BeforeClass/@AfterClass 는 static 해당 테스트 클래스에서 시작할때, 끝날 때 한번씩만 호출 된다.
- 모든 테스트 케이스 전후에 꼭 한번 실행할 로직은 @Before/@After 를 붙인 public 메서드에 구현 한다.
	- @Before/@After 는 @Test 가 붙은 테스트 케이스의 시작과 끝 부분에 매번 한번 씩 호출 된다.
- org.junit.Assert 클래스에 선언된 assertion 을 호출해서 결과 값이나 테스트 케이스에 대한 검증을 수행 할 수 있다.
	- 직접 호출하기 위해 `import static org.junit.Assert.*` 를 선언해 `assertTrue(true)` 와 같이 바로 호출 한다.
- Junit 에 대한 의존성 추가하기
	- Maven(pom.xml)
	
		```xml
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>4.3.18</version>
        </dependency>
		<dependency>
			<groupId>junit</groupId>
			<atifactId>junit</atifactId>
			<version>4.12</version>
		</dependency>
		```  
		
	- Gradle(build.gradle)
	
		```
		dependencies {
			testCompile "org.springframework:spring-test:4.3.18"
			testCompile "junit:junit:4.12"
		}
		```  
		
- 이자 계산기의 Junit 테스트 케이스는 아래와 같다.

```java
import static org.junit.Assert.*;

public class SimpleInterestCalculatorJunit4Tests {
	private static InterestCalculator interestCalculator;
	
	@BeforeClass
	public static void setUpBeforeClass() {
		interestCalculator = new SimpleInterestCalculator();
	}
	
	@Before
	public void setUp() {
		interestCalculator.setRate(0.05);
	}
	
	@Test
	public void calculator() {
		double interest = interestCalculator.calculate(10000, 2);
		assertEquals(interest, 10000.0, 0);
	}
	
	@Test(expected = IllegalArgumentException.class)
	public void illegalCalculate() {
		interestCalculator.calculate(-10000, 2);
	}
}
```  

- 테스트 케이스에서 @Test Annotation 에서 expected 속성에 예외 타입을 적어 에외에 대한 테스트도 가능 하다.

## TestNG
- TestNG 는 전용 클래스와 Annotation 타입을 사용하는 차이 외에는 Junit 과 동일하다.
- 다른 부분은 테스트 클래스에서 한번씩만 호출되는 @BeforeClass/@AfterClass 의 메서드가 static 아니다.
	- @Test 가 붙은 테스트 케이스의 시작과 끝부분에 호출 되는 Annotation 은 @BeforeMethod/@AfterMethod 이다.
- TestNG 의존성 추가
	- Maven(pom.xml)
		
		```xml
		<dependency>
			<groupId>org.testng</groupId>
			<artifactId>testng</artifactId>
			<version>6.11</version>
		</dependency>
		```  
		
	- Gradle(build.gradle)
	
		```
		dependencies {
			testCompile "org.testng:testng:6.11"
		}
		```  
		
- TestNG 를 사용한 테스트 코드는 아래와 같다.

```java
import static org.testng.Assert.*;

public class SimpleInterestCalculatorTestNGTests {
	private interestCalculator interestCalculator;
	
	@BeforeClass
	public void setUpBeforeClass() {
		this.interestCalculator = new SimpleInterestCalculator();
	}
	
	@BeforeMethod
	public void setUp() {
		this.interestCalculator.setRate(0.05);
	}
	
	@Test
	public void calculator() {
		double interest = this.interestCalculator.calculate(10000, 2);
		
		assertEquals(interest, 1000.0);
	}
	
	@Test(expectedException = IllgegalArgumentException.class)
	public void illegalCalculate() {
		this.interestCalculator.calculate(-10000, 2);
	}
}
```  

- TestNG 의 특징은 데이터 주도 테스트를 지원한다는 점이다.
- TestNG 는 테스트 데이터와 테스트 로직을 깔끔하게 분리할수 있으므로 데이터 세트만 바꿔가면서 테스트 케이스를 여러 번 실행 할 수 있다.
- 데이터 세트는 @DataProvider 를 붙인 데이터 공급자 메서드가 제공한다.

```java
import static org.testng.Assert.*;

public class SimpleInterestCalculatorTestNGTests {
	private InterestCalculator interestCalculator;
	
	@BeforeMethod
	public void setUp() {
		this.interestCalculator = new SimpleInterestCalculator();
		this.interestCalculator.setRate(0.05);
	}
	
	@DataProvider(name = "lagal")
	public Object[][] createLegalInterestParameter() {
		return new Object[][]{new Object[]{10000, 2, 1000.0}};
	}
	
	@DataProvider(name = "illegal")
	public Object[][] createIllegalInterestParameter() {
		return new Object[][] {
			new Object[] {-10000, 2},
			new Object[] {10000, -2},
			new Object[] {-1000, -2}
		};
	}
	
	@Test(dataProvider = "legel")
	public void calculator(double amount, double year, double result) {
		double interest = this.interestCalculator.calculator(amount, year);
		assertEquals(interest, result);
	}
	
	@Test(
		dataProvider = "illegal",
		expectedExceptions = IllegalArgumentException.calss)
	public void illegalCalculator(double amount, double year) {
		this.interestCalculator.calculator(amount, year);
	}
}
```  

- 테스트 케이스인 calculator() 메서드는 1회, illegalCalculator() 메서드는 3회 실행되고, 데이터 공급자 illegal 은 3개 데이터 세트를 반환한다.

## Spring Junit 단위 테스트 하기

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations ={"classpath:<bean-config-file-path>.xml"})
public class SomethingTest {
	
}
```  

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = ConfigClass.class)
public class SomethingTest {
	
}
```  

---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
[새내기 개발자의 JUnit 여행기](http://www.nextree.co.kr/p11104/)  
