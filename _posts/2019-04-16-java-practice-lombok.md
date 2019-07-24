--- 
layout: single
classes: wide
title: "[Java 실습] Lombok 을 사용해 보자"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Lombok 을 사용해서 유용한 Annotation 을 통해 생산성을 높이자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
	- Java
    - Spring
    - Practice
    - Lombok
---  

# 목표
- Lombok 의 Annotation 을 사용해서 반복적인 작업에 대한 처리에 대한 생산성을 높인다.

# 방법
## Lombok 이란
- Java 기반에서 기계적으로 작성하는 VO, DTO, Entity 관련 작업을 보다 쉽게 하게 해주는 도구이다.
- Getter, Setter, ToString, hashCode 메소드 작업에 대한 Class 코드를 깔끔하게 작성할 수 있다.
- Spring 프로젝트에서 사용할 경우 JPA 환경과 함께 일관화 되고 가독성이 좋은 Application 을 작성 할 수 있다.
- 단점으로는 협업 모든 인원이 lombok 을 사용해야 한다는 것과 Annotation 을 사용할 경우 소스코드 분석이 난해해진다는 점이 있다.
	
# 예제
- Lombok 사용을 위해 의존성을 추가한다.
- Maven(pom.xml)

	```xml
	<dependency>
	    <groupId>org.projectlombok</groupId>
	    <artifactId>lombok</artifactId>
	    <version>1.16.20</version>
	    <!--<scope>provided</scope>-->
	</dependency>
	```  
- Gradle(build.gradle)

	```
	dependencies {
		compile "org.projectlombok:lombok:1.16.20"
	}
	```  
	
## Intellij Lombok 사용 설정
- File -> Settings -> Plugins -> Browse Repositories -> lombok 검색

![intellij lombok plugin 설치]({{site.baseurl}}/img/spring/lombok-installplugin-1.png)

- 이미 설치된 상태라서 보이지 않지만 Install 을 눌러 설치한다.

- File -> Settings -> Build, Execution, Deployment -> Compiler -> Build project automatically 체크

![lombok 사용 설정 build project automatically]({{site.baseurl}}/img/spring/lombok-buildprojectautomatically-1.png)

- File -> Settings -> Build, Execution, Deployment -> Compiler -> Annotation Processors -> Enable annotation processing 체크

![lombok 사용 설정 enable annotation processing]({{site.baseurl}}/img/spring/lombok-enableannotationprocessing-1.png)

- Applicatio 수행 시에도 컴파일이 되도록 설정을 변경한다.
	- Find Action -> Registry 검색 -> compiler.automake.allow.when.app.running 체크
	
![lombok 사용 설정 registry]({{site.baseurl}}/img/spring/lombok-registry-1.png)

## Lombok 의 Annotation
### Getter/Setter 자동 생성
- @Getter/@Setter Annotation 을 사용하면 해당 클래스 프로퍼티들의 Getter 와 Setter 가 자동으로 생성된다.

```java
@Getter
@Setter
public class Person {
	private String name;
	private boolean male;
}
```  

```java
public class Person {
	@Getter
	@Setter
	private String name;
}
```  

- String 형식의 name 프로퍼티에 선언해 주면 getName(), setName() 메서드가 자동으로 생성된다.
- boolean 형식의 isMale 프로퍼티에 선언해 주면 isMale(), setMale() 메서드가 자동으로 생성된다.
- Class 레벨 선언 및 Field 레벨 선언 모두 가능하다.

```java
Person person = new Person();
person.setName("windowforsun");
person.setMale(true);

System.out.println(person.getName + ", " + person.isMale());
```  

### 생성자 재동 생성
- @NoArgsConstructor Annotation 은 파라미터가 없는 기본 생성자를 생성 해준다.
- @AllArgsConstructor Annotation 은 모든 프로퍼티 값을 파라미터로 받는 생성자를 생성해 준다.
- @RequiredArgsConstructor Annotation 은 final 이나 @NonNull 인 필드 값만 파라미터로 받는 생성자를 생성해 준다.

```java
@NoArgsConstructor
@RequiredArgsConstructor
@AllArgsConstructor
public class Person {
	private String name;
	private boolean male;
	@NonNull
	private String phone;
}
```  

```java
Person person1 = new Person();
Person person2 = new Person("01012341234");
Person person3 = new Person("windowforsun", true, "01012341234");
```  

### ToString 메서드 자동 생성
- @ToString Annotation 을 사용해서 toString() 메서드를 자동으로 생성할 수 있다.
- exclude 속성을 사용해서 특정 필드를 toString() 메서드 결과에서 제외 시킬 수 있다.

```java
@ToString
@Getter
@Setter
public class Obj {
	private String id;
}


@ToString(exclude = "phone", callSuper = true)
@Getter
@Setter
public class Person extends Obj {
	private String name;
	private boolean male;
	private String phone;
}
```  

```java
Person person = new Person();
person.setName("windowforsun");
person.setMale(true);
person.setPhone("01012341234");
person.setId("id");

System.out.println(person);
```  

```
Person(super=Obj(id=id), name=windowforsun, male=true)
```  

### equals, hashCode 자동 생성
- @EqualsIsAndHashCode Annotation 을 사용해서 자동으로 메서드 오버라이딩이 가능하다.

```java
@Getter
@Setter
public class Obj {
	private String id;
}

@EqualsIsAndHashCode(callSuper = true)
@Getter
@Setter
public class Person extends Obj {
	private String name;
	private boolean male;
	private String phone;
}
```  

- callSuper 속성을 통해 equals, hashCode 메소드 자동 생성시 부모 클래스의 필드까지 포함 할지에 대한 설정을 할 수 있다.
- callSuper = true 일 경우 부모 클래스 필드 값들도 동일한지 체크하며, callSuper = false 일 경우에는 자신의 클래스 필드 값만 고려한다.

```java
Person person1 = new Person();
person1.setId("id1");
person1.setName("windowforsun");
person1.setMale(true);
person1.setPhone("01012341234");

Person person2 = new Person();
person2.setId("id2");
person2.setName("windowforsun");
person2.setMale(true);
person2.setPhone("01012341234");

person1.equals(person2);
// 부모 클래스 필드인 id 값이 다르므로 false 이지만 callSuper = false 일 경우 true 값을 반환한다.
```  

### @Data
- @Data Annotation 은 위에서 설명한 @Getter, @Setter, @RequiredArgsConstructor, @ToString, @EqualsAndHashCode 를 한번에 설정해주는 Annotation 이다.

```java
@Data
public class Person {
	// ...
}
```  

- 클래스 레벨에 @Data Annotation 을 붙여주면, 모든 필드 대상으로 Getter, Setter 가 자동으로 생성된다.
- final 또는 @NonNull 필드 값을 파라미터로 받는 생성자가 만들어진다.
- toString, equals, hashCode 메소드가 자동으로 만들어진다.

### 빌더 자동 생성
- 다수의 필드를 가지는 복잡한 클래스의 경우, 생성자 대신 빌더를 사용하면 생산성을 높일 수 있다.
- 빌더를 사용하기 위해서는 빌더 패턴을 작성해야 하는데 코드량이 상당하기 때문에 @Builder Annotation 을 통해 쉽게 사용 할 수 있다.

```java
@Builder
public class Person {
	private String name;
	private boolean male;
	private String phone;
	@Singular
	private List<String> hobbies;
}
```  

- Collection 형식의 필드에 @Singular Annotation 을 선언하면 모든 원소를 한번에 넘기지 않고 원소를 하나씩 추가할 수 있다.

```java
Person person = Person.builder()
	.name("windowforsun")
	.male(true)
	.phone("01012341234")
	.hobbies("soccer")
	.hobbies("movie")
	.hobbies("music");
```  

- 상속 구조에서는 아래와 같은 방법으로 빌더를 사용할 수 있다.

	```java
	@Getter
	@Setter
	@AllArgsConstructor
	public class Parent  {
	    protected long parentLong;
	    protected int parentInt;
	    protected String parentString;
	}
	```  
	
	```java
	@Getter
	@Setter
	public class Child  {
	    protected long childLong;
	    protected int childInt;
	    protected String childString;
	    
	    @Builder
	    public Child(long parentLong, int parentInt, String parentString, long childLong, int childInt, String childString) {
	    	super(parentLong, parentInt, parentString);
	    	this.childLong = childLong;
	    	this.childInt = childInt;
	    	this.childString = childString;
	    }
	}
	```  
	
	```java
	Parent p = Parent.builder()
        .parentLong(1l)
        .parentInt(1)
        .parentString("parentString")
        .build();
	```  
	
	```java
	Child p = Child.builder()
        .parentLong(1l)
        .parentInt(1)
        .parentString("parentString")
        .childLong(2l)
        .childInt(2)
        .childString("childString")
        .build();
	```  
	
	- `Jackson` 라이브러리를 사용해서 Json Serialize/Deserialze 를 할 경우 자식 클래스에 `@JsonDeserialize` Annotation 을 추가해 준다.
	

### 로그 자동 생성
- @Log Annotation 을 사용하면 자동으로 log 필드를 만들고, 해당 클래스의 이름으로 Logger 객체를 생성하여 할당한다.
- @Log 뿐만 아니라, @Slf4j, @Log4j, @Log4j2 등 다양한 Logging Framework 에 대응하는 Annotation 을 제공한다.

```java
@Log
public class LoggingSample {
	public void writeLog() {
		log.info("Test");
	}
}
```  

### Null 체크
- @NonNull Annotation 을 변수에 붙이면 자동으로 null 체크를 해준다.
- 해당 변수가 null 일 경우 NullPointException 예외를 일으킨다.

```java
public class Person {
	@NonNull
	private String name;
}
```  

```java
person.setName(null); // NullPointException 발생
```  

### 자동 자원 닫기
- IO 처리, JDBC 등 try-catch-finally 의 finally 를 통해 close() 메서드를 호출 하게 된다.
- @Cleanup Annotation 을 사용하면 해당 자원이 자동으로 닫히는 것이 보장된다.

```java
@Cleanup Connection conn = DriverManager.getConnection(url, user, passwd)
```  

### 예외 처리 생략
- CheckedException 의 조건으로 코드 작성 시 마다 예외 처리 관련 코드가 들어가게 된다.
- @SneakyThrows Annotation 을 사용하면 명시적인 예외 처리를 생략 할 수 있다.

```java
@SneakyThrows(IOException.class)
public void print() {
	BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
	System.out.println(reader.readLine());
}
```  

### 동기화
- Java 의 synchronized 키워드를 메소드에 선언해 객체 레벨에서 Lock 이 걸려 동기화 관련 문제들이 발생 할 수 있다.
- @Synchronized Annotation 을 사용하면 가상으 필드 레벨에서 좀 더 안전하게 Lock 을 걸어 사용할 수 있다.

```java
@Synchronized
public void say() {
	System.out.println("hi");
}
```  

### 불변 클래스
- 한번 샌성해서 변경할 수 없는 불변 객체를 만들기 위한 클래스는 @Data Annotation 대신, @Value Annotation 을 사용하면 된다.

```java
@Value
public class ImmutableClass {
	// ...
}
```  

---
## Reference
[Lombok 사용 시 IntelliJ 셋팅](http://blog.egstep.com/java/2018/01/12/intellij-lombok/)  
[Java Lombok 사용 및 설치](https://niceman.tistory.com/99)  
[[자바] 알아두면 유용한 Lombok 어노테이션](http://www.daleseo.com/lombok-useful-annotations/)  
[[자바] 알아두면 유용한 Lombok 어노테이션](http://www.daleseo.com/lombok-popular-annotations/)  
