--- 
layout: single
classes: wide
title: "[Java 실습] Jackson Polymorphic Serialize/Deserialize"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Jackson 라이브러리로 다형성 클래스 Serialize/Deserialize 하기'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Practice
    - Java
    - Jackson
    - Polymorphism
    - Lombok
---  

## 환경
- Java 8
- Spring Boot 2
- Maven

## 다형성(Polymorphism)
- 하나의 인터페이스 또는 추상 클래스로 표현될 수 있으면서, 다른 구현을 가진다.
- 다형성을 통해 트리 형태로 구조화 된 객체 인스턴스를 `Jackson` 라이브러를 통해 직렬화/역직렬화 방법에 대해 알아본다.

## 사전 작업
- `Jackson` 라이브러리 의존성 추가해준다.

	```xml
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.9.8</version>
    </dependency>
	```  
	
- 소스코드 작성에 사용할 [`Lombok` 라이브러리]({{site.baseurl}}{% link _posts/2019-04-16-java-practice-lombok.md %})도 추가해준다.

	```xml
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.4</version>
    </dependency>
	```  
	
## 예제
- 다형성을 띄고 있는 트리 형태의 클래스 구조는 아래와 같다.

	![그림 1]({{site.baseurl}}/img/spring/practice-jackson-lombok-polymorphism-1.png)

- 위 구조를 소스코드로 구현하면 아래와 같다.

	```java
    public abstract class Parent {
        protected int parentInt;
        protected String parentString;
    
        public abstract String classInfo();
    }
    
    public class ChildA extends Parent {
        protected int childAInt;
        protected String childAString;
        
        @Override
        public String classInfo() {
            return ChildA.class.toString();
        }
    }
    
    public class ChildB extends Parent{
        protected double childBDouble;
        protected String childBString;
    
        @Override
        public String classInfo() {
            return ChildB.class.toString();
        }
    }

	```  
	
- `Jackson` 과 `Lombok` Annotation 을 적용 시키면 아래와 같은 코드가 된다.

	```java
	@JsonTypeInfo(
            use = JsonTypeInfo.Id.NAME,
            include = JsonTypeInfo.As.PROPERTY,
            property = "className"
    )
    @JsonSubTypes({
            @JsonSubTypes.Type(value=ChildA.class, name="ChildA"),
            @JsonSubTypes.Type(value=ChildB.class, name="ChildB")
    })
    @AllArgsConstructor
    @Getter
    @Setter
    @NoArgsConstructor
    @EqualsAndHashCode
    @ToString
    public abstract class Parent {
        protected int parentInt;
        protected String parentString;
    
        public abstract String classInfo();
    }
	```  
		
	- `Parent` 를 상속받는 하위 클래스에 대한 명시를 `@JsonTypeInfo`, `@JsonSubTypes` 를 통해 해준다.
	- 현재 `Parant` 클래스에서 하위클래스를 명시의 Id 는 `NAME` 으로 설정된 상태이다.
	- `Json` 에서 `"className":"ChildA"` 의 필드가 있으면 `ChildA` 로 처리하게 된다.
	
	```java
	@EqualsAndHashCode(callSuper = true)
    @ToString(callSuper = true)
    @Getter
    @Setter
    @NoArgsConstructor
    public class ChildA extends Parent {
        protected int childAInt;
        protected String childAString;
    
        @Builder
        public ChildA(int parentInt, String parentString, int childAInt, String childAString) {
            super(parentInt, parentString);
            this.childAInt = childAInt;
            this.childAString = childAString;
        }
    
        @Override
        public String classInfo() {
            return ChildA.class.toString();
        }
    }
	```   
	
	```java
	@EqualsAndHashCode(callSuper = true)
    @ToString(callSuper = true)
    @Getter
    @Setter
    @NoArgsConstructor
    public class ChildB extends Parent{
        protected double childBDouble;
        protected String childBString;
    
        @Builder
        public ChildB(int parentInt, String parentString, double childBDouble, String childBString) {
            super(parentInt, parentString);
            this.childBDouble = childBDouble;
            this.childBString = childBString;
        }
    
        @Override
        public String classInfo() {
            return ChildB.class.toString();
        }
    }
	```  
	
	- 하위 클래스들은 원활한 객체 생성을 위해 `@Builder` 를 이용했다.
	
- 테스트 코드는 아래와 같다.

	```java
	public class JacksonPolymorphismTest {
        private ObjectMapper objectMapper = new ObjectMapper();
    
        @Test
        public void childA_json_to_object_byName() throws Exception{
            String jsonStr = "{\n" +
                    "  \"className\" : \"ChildA\",\n" +
                    "  \"parentInt\" : 1,\n" +
                    "  \"parentString\" : \"parent\",\n" +
                    "  \"childAInt\" : 2,\n" +
                    "\t\"childAString\" : \"childA\"\n" +
                    "}";
    
            ChildA childA = this.objectMapper.readValue(jsonStr, ChildA.class);
    
            assertThat(childA.classInfo(), is(ChildA.class.toString()));
            assertThat(childA.getParentInt(), is(1));
            assertThat(childA.getParentString(), is("parent"));
            assertThat(childA.getChildAInt(), is(2));
            assertThat(childA.getChildAString(), is("childA"));
        }
    
        @Test
        public void childB_json_to_object_byName() throws Exception{
            String jsonStr = "{\n" +
                    "  \"className\" : \"ChildB\",\n" +
                    "  \"parentInt\" : 11,\n" +
                    "  \"parentString\" : \"parent\",\n" +
                    "  \"childBDouble\" : 22.22,\n" +
                    "\t\"childBString\" : \"childB\"\n" +
                    "}";
    
            ChildB childB = this.objectMapper.readValue(jsonStr, ChildB.class);
    
            assertThat(childB.classInfo(), is(ChildB.class.toString()));
            assertThat(childB.getParentInt(), is(11));
            assertThat(childB.getParentString(), is("parent"));
            assertThat(childB.getChildBDouble(), is(22.22));
            assertThat(childB.getChildBString(), is("childB"));
        }
    
        @Test
        public void  childA_object_to_json_to_object_byName() throws Exception {
            ChildA expected = ChildA.builder()
                    .parentInt(1)
                    .parentString("parent")
                    .childAInt(2)
                    .childAString("childA")
                    .build();
    
            String jsonStr = this.objectMapper.writeValueAsString(expected);
            ChildA childA = this.objectMapper.readValue(jsonStr, ChildA.class);
    
            assertThat(childA, is(expected));
            assertThat(childA.classInfo(), is(expected.classInfo()));
        }
    
        @Test
        public void  childB_object_to_json_to_object_byName() throws Exception {
            ChildB expected = ChildB.builder()
                    .parentInt(11)
                    .parentString("parent")
                    .childBDouble(22.22)
                    .childBString("childB")
                    .build();
    
            String jsonStr = this.objectMapper.writeValueAsString(expected);
            ChildB childB = this.objectMapper.readValue(jsonStr, ChildB.class);
    
            assertThat(childB, is(expected));
            assertThat(childB.classInfo(), is(expected.classInfo()));
        }
    }
	```  
	
- Id 값을 `CLASS` 로 사용하는 경우 클래스 소스코드는 아래와 같다.

	```java
	@JsonTypeInfo(
            use = JsonTypeInfo.Id.CLASS,
            include = JsonTypeInfo.As.PROPERTY,
            property = "@class"
    )
    @JsonSubTypes({
            @JsonSubTypes.Type(value=ChildAByClass.class, name="ChildAByClass"),
            @JsonSubTypes.Type(value=ChildBByClass.class, name="ChildBByClass")
    })
    @AllArgsConstructor
    @Getter
    @Setter
    @NoArgsConstructor
    @EqualsAndHashCode
    @ToString
    public abstract class ParentByClass {
        protected int parentInt;
        protected String parentString;
    
        public abstract String classInfo();
    }
	```  
	
	- `"@class":"<패키지경로>.ChildAByClass"`  의 필드가 있으면 `ChildA` 로 처리한다.
	- `ChildAByClass`, `ClassBByClass` 는 `ParentByClass` 를 상속 받는 다는 점을 제외하면 `ChildA`, `ChildB` 와 같다.
	
- Id 가 `CLASS` 인 경우를 테스트하는 코드는 아래와 같다.

	```java
	public class JacksonPolymorphismTest {
        private ObjectMapper objectMapper = new ObjectMapper();
        
        @Test
        public void childAByClass_json_to_object_byClass() throws Exception {
            String jsonStr = "{\n" +
                    "  \"@class\" : \"com.example.demo.ChildAByClass\",\n" +
                    "  \"parentInt\" : 1,\n" +
                    "  \"parentString\" : \"parent\",\n" +
                    "  \"childAInt\" : 2,\n" +
                    "\t\"childAString\" : \"childAByClass\"\n" +
                    "}";
    
            ChildAByClass childAByClass = this.objectMapper.readValue(jsonStr, ChildAByClass.class);
    
            assertThat(childAByClass.classInfo(), is(ChildAByClass.class.toString()));
            assertThat(childAByClass.getParentInt(), is(1));
            assertThat(childAByClass.getParentString(), is("parent"));
            assertThat(childAByClass.getChildAInt(), is(2));
            assertThat(childAByClass.getChildAString(), is("childAByClass"));
        }
    
        @Test
        public void childBByClass_json_to_object_byClass() throws Exception {
            String jsonStr = "{\n" +
                    "  \"@class\" : \"com.example.demo.ChildBByClass\",\n" +
                    "  \"parentInt\" : 11,\n" +
                    "  \"parentString\" : \"parent\",\n" +
                    "  \"childBDouble\" : 22.22,\n" +
                    "\t\"childBString\" : \"childBByClass\"\n" +
                    "}";
    
            ChildBByClass childBByClass = this.objectMapper.readValue(jsonStr, ChildBByClass.class);
    
            assertThat(childBByClass.classInfo(), is(ChildBByClass.class.toString()));
            assertThat(childBByClass.getParentInt(), is(11));
            assertThat(childBByClass.getParentString(), is("parent"));
            assertThat(childBByClass.getChildBDouble(), is(22.22));
            assertThat(childBByClass.getChildBString(), is("childBByClass"));
        }
    
        @Test
        public void childAByClass_object_to_json_to_object_byClass() throws Exception {
            ChildAByClass expected = ChildAByClass.builder()
                    .parentInt(1)
                    .parentString("parent")
                    .childAInt(2)
                    .childAString("childAByClass")
                    .build();
    
            String jsonStr = this.objectMapper.writeValueAsString(expected);
            ChildAByClass childAByClass = this.objectMapper.readValue(jsonStr, ChildAByClass.class);
    
            assertThat(childAByClass, is(expected));
            assertThat(childAByClass.classInfo(), is(expected.classInfo()));
        }
    
        @Test
        public void childBByClass_object_to_json_to_object_byClass() throws Exception {
            ChildBByClass expected = ChildBByClass.builder()
                    .parentInt(2)
                    .parentString("parent")
                    .childBDouble(22.22)
                    .childBString("childBByClass")
                    .build();
    
            String jsonStr = this.objectMapper.writeValueAsString(expected);
            ChildBByClass childBByClass = this.objectMapper.readValue(jsonStr, ChildBByClass.class);
    
            assertThat(childBByClass, is(expected));
            assertThat(childBByClass.classInfo(), is(expected.classInfo()));
        }
    }
	```  


---
## Reference
[Polymorphic Deserialization with Jackson](https://igorski.co/java/polymorphic-deserialization-jackson/)  
[Inheritance with Jackson](https://www.baeldung.com/jackson-inheritance)  
[POLYMORPHISM AND INHERITANCE WITH JACKSON](https://octoperf.com/blog/2018/02/01/polymorphism-with-jackson/)  
[Jackson Polymorphic Deserialization](https://medium.com/@david.truong510/jackson-polymorphic-deserialization-91426e39b96a)  
[Polymorphic endpoints with Jackson](http://publicstaticmain.blogspot.com/2016/02/polymorphic-endpoints-with-jackson.html)  
