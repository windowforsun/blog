--- 
layout: single
classes: wide
title: "[Java 실습] Annotation 과 Custom Annotation"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java Annotation 과 사용자 정의 Annotation 을 정의하고 사용해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Practice
    - Java
    - Annotation
---  

## Annotation 이란
- JDK 5 부터 추가된 기능이다.
- Annotation 은 주석이라는 의미를 가지고 있지만, 일반적인 주석과는 다르게 특정 의미를 포함하고 있는 주석이다.
- 클래스, 메서드, 변수, 인자에 추가할 수 있다.
- Annotation 은 메타데이터이기 때문에, 로직에 비지니스 로직에 직접적인 영향은 주지 않는다.
- 실행 흐름을 변경하거나, 반복적인 코딩을 더욱 깔끔하게 구성할 수 있다.

## Annotation 의 타입
### Marker Annotation

```java
@MyMarkerAnnotation
```  

- `@Ovrride`, `@Deprecated` 와 같은 Annotation 처럼 표시만 하는 것을 뜻한다.
- Marker Annotation 의 구현체를 보면 메서드가 없는 것을 확인 할 수 있다.

```java
public @interface MyMarkerAnnotation {
}
```  

```java
@MyMarkerAnnotation
public class MyClass {
}
```
	
### SingleValue Annotation

```java
@MySingleValueAnnotation(value = 10)
```  

- 하나의 값만 입력받을 수 있는 Annotation 이다.
- SingeValue Annotation 의 구현체를 보면 하나의 메서드만 있는 것을 확인할 수 있다.

```java
public @interface MySingleValueAnnotation {
	int value();
}
```  

```java
@MySingleValueAnnotation(value = 10)
public class MyClass {
}
```  
	
### MultiValue Annotation

```java
@MyMultiValueAnnotation(value = 10, name = "myName", roles = {"user", "admin"})
```

- 여러 값을 입력 받을 수 있는 Annotation 이다.
- 구현체에는 여러 메서드가 선언되어 있다.
- 각 메서드의 기본값은 `default` 키워드를 통해 선언할 수 있다.

```java
public @interface MyMultiValueAnnotation {
	int id();
	String name() default "name";
	String[] roles() default {"anonymous"};
}
```  

```java
@MyMultiValueAnnotation(value = 10, name = "myName", roles = {"user", "admin"})
public class MyClass {
}
```  

## Java Built-in Annotation
- Java 에서 기본적으로 제공하는 Annotation 에 대해 알아본다.

### @Override
-  Compiler 에게 상위 클래스의 메서드를 오버라이드 함을 알려준다.
- 오버라이드시에 필수적으로 선언해야하는 Annotation 은 아니지만, Compiler 가 오류를 발생시켜 잘못된 오버라이드를 잡을 수 있다.

### @SuppressWarnings
- Compiler 에게 특정 경고를 무시하도록 한다.
- `deprecation`, `unchecked`, `unused` 등 다양한 경고를 무시할 수 있다.
- `all` 을 사용하면 모든 경고를 무시한다.

### @SafeVarargs
- `Varargs` 는 가변 인자를 뜻한다.

	```java
	public void some(String... strs) {
	
	}
	```  
	
- Varargs 에 대한 `unchecked` 경고를 발생시키지 않는다.

### @Deprecated
- 더 이상 사용하면 안된다는 의미를 가지고 있는 Annotation 이다.
- Compiler 는 해당 Annotation 이 선언것을 사용할 경우 경고를 발생시키니다.

### @FunctionalInterface
- Interface 에서 한 개의 Method 만 가지도록 한다.
- Java8 부터 도입된 Lamda Expression 과 관련된 Annotation 이다.
- 한 개의 Method 만 가진 Interface 의 경우 Method 의 이름을 생략할 수 있다.

### @Target
- Annotation 을 정의 할때 사용하는 Annotation 이다.
- Annotation 이 적용되는 범위를 나타낸다.
- `ElementType.TYPE` : class, interface, enum 에 적용된다.
- `ElementType.FIELD` : 클래스 필드 변수 에 적용된다.
- `ElementType.METHOD` : Method 에 적용된다.
- `ElementType.PARAMETER` : Method 인자 에 적용된다.
- `ElementType.CONSTRUCTOR` : 생성자에 적용된다.
- `ElementType.LOCAL_VARIABLE` : 로컬 변수에 적용된다.
- `ElementType.ANNOTATION_TYPE` : Annotation 타입에만 적용된다.
- `ElementType.PACKAGE` : 패키지에 적용된다.
- `ElementType.TYPE_PARAMETER` : Java8 에 추가된 것으로, Generic Type 변수에 적용된다.
- `ElementType.TYPE_USE` : Java8 에 추가된 것으로, 어떤 타입에도 적용된다.(extends, implements, 객체 생성 등)
- `ElementType.MODULE` : Java9 에 추가된 것으로, 모듈에 적용된다.

### @Retention
- Annotation 을 정의 할때 사용하는 Annotation 이다.
- Annotation 이 어떻게 저장되는지를 뜻한다.
- `RetentionPolicy.SOURCE` : source level 이고, compiler 는 인식하지 않는다.
- `RetentionPolicy.CLASS` : compiler 는 인식하지만, JVM 은 인식하지 않는다.
- `RetentionPolicy.RUNTIME` : JVM 이 인식한다.

### @Document
- Annotation 을 정의 할때 사용하는 Annotation 이다.
- Annotation 이 선언되면 문서화 되어야 한다.

### @Inherited
- Annotation 을 정의 할때 사용하는 Annotation 이다.
- 상속관계에서 Annotation 은 상속되지 않지만, 해당 Annotation 을 사용하면 상위 클래스의 Annotation 도 상속받게 된다.

### @Repeatable
- Annotation 을 정의 할때 사용하는 Annotation 이다.
- 선언된 Annotation 을 반복적으로 사용 가능하다.

	```java
	@MyAnnotation(value = 10)
	@MyAnnotation(value = 100)
	public class MyClass {
	}
	```  
	
## Custom Annotation
- Class Field 에 값을 주입하는 Custom Annotation 을 구현한다.

### int 값을 주입하는 DefaultInteger Annotation

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface DefaultInteger {
    int value() default 100;
}
```  

### String 값을 주입하는 DefaultString Annotation

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface DefaultString {
    String str() default "default";
}
```  

### Annotation 을 사용할 ExamClass

```java
public class ExamClass {
    @DefaultInteger
    private int value;
    @DefaultInteger(value = 200)
    private int value2;
    @DefaultString
    private String name;
    @DefaultString(str = "str2")
    private String name2;

    public int getValue() {
        return value;
    }

    public int getValue2() {
        return value2;
    }

    public String getName() {
        return name;
    }

    public String getName2() {
        return name2;
    }
}
```  

### 값을 주입하는 Annotation 수행관련 클래스
- 값 주입 Annotation 의 Abstract Class

	```java
	public abstract class SetFieldAnnotation<A> {
        // 각 값 설정 Annotation 에 맞게 동작을 수행함
        public <T> T setField(T targetObj, Field field) {
            A annotation = this.getAnnotation(field);
    
            if(annotation != null && field.getType() == this.getType()) {
                field.setAccessible(true);
    
                try {
                    field.set(targetObj, this.getData(annotation));
                } catch(IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
    
            return targetObj;
        }
    
        // 현재 필드에서 사용하는 Annotation 객체 반환
        public abstract A getAnnotation(Field field);
    
        // Annotation 이 설정하는 값의 타입
        public abstract Class getType();
    
        // Annotation 이 설정하는 값
        public abstract Object getData(A annotation);
    }
	```  
	
- int 값을 주입하는 DefaultInteger 의 구현체

	```java
	public class DefaultIntegerSetFieldAnnotationImpl extends SetFieldAnnotation<DefaultInteger> {
		@Override
		public DefaultInteger  getAnnotation(Field field) {
			return field.getAnnotation(DefaultInteger.class);
		}
	
		@Override
		public Class getType() {
			return int.class;
		}
	
		@Override
		public Object getData(DefaultInteger annotation) {
			return annotation.value();
		}
	}
	```  
	
- String 값을 주입하는 DefaultString 의 구현체

	```java
	public class DefaultStringSetFieldAnnotationImpl extends SetFieldAnnotation<DefaultString> {
		@Override
		public DefaultString getAnnotation(Field field) {
			return field.getAnnotation(DefaultString.class);
		}
	
		@Override
		public Class getType() {
			return String.class;
		}
	
		@Override
		public Object getData(DefaultString annotation) {
			return annotation.str();
		}
	}
	```  
	
- Annotation 사용 객체에 Annotation 을 적용하는 Handler 클래스

	```java
	public class AnnotationHandler {
        // Annotation 구현체를 Annotation Type 과 매핑
        private HashMap<Class, SetFieldAnnotation> setFieldAnnotationHashMap;
    
        public AnnotationHandler() {
            this.setFieldAnnotationHashMap = new HashMap<>();
            this.setFieldAnnotationHashMap.put(DefaultInteger.class, new DefaultIntegerSetFieldAnnotationImpl());
            this.setFieldAnnotationHashMap.put(DefaultString.class, new DefaultStringSetFieldAnnotationImpl());
        }
    
        // 값을 설정하는 Annotation 을 targetObj 에 적용 한다.
        private <T> T setFieldAnnotation(T targetObj) {
            // targetObj 의 필드
            Field[] fields = targetObj.getClass().getDeclaredFields();
            for(Field field : fields) {
                // field 에 선언된 Annotation 들
                Annotation[] annotations = field.getAnnotations();
    
                for(Annotation annotation : annotations) {
                    // 선언된 Annotation 이 값 설정 Annotation 인 경우
                    if(this.setFieldAnnotationHashMap.containsKey(annotation.annotationType())) {
                        // 값 설정 Annotation 수행
                        this.setFieldAnnotationHashMap.get(annotation.annotationType()).setField(targetObj, field);
                    }
                }
            }
            return targetObj;
        }
    
        // Annotation 을 사용하는 클래스의 객체를 Annotation 을 적용한 상태로 반환
        public <T> Optional<T> getInstance(Class targetClass) {
            Optional optional = Optional.empty();
            Object obj;
    
            try {
                //  클래스 객체 생성
                obj = targetClass.newInstance();
                // 값 설정 Annotation 적용
                obj = setFieldAnnotation(obj);
                optional = Optional.of(obj);
            } catch(InstantiationException | IllegalAccessException e) {
                e.printStackTrace();
            }
    
            return optional;
        }
    }
	```  
	
### Custom Annotation Test

```java
public class CustomAnnotationTest {
    private AnnotationHandler annotationHandler;

    @Before
    public void setUp() {
        this.annotationHandler = new AnnotationHandler();

    }
    @Test
    public void classStringFieldDefault() {
        ExamClass examClass = this.annotationHandler.getInstance(ExamClass.class)
                .map(o -> (ExamClass)o)
                .orElse(new ExamClass());

        assertThat(examClass.getValue(), is(100));
        assertThat(examClass.getValue2(), is(200));
        assertThat(examClass.getName(), is("default"));
        assertThat(examClass.getName2(), is("str2"));
    }
}
```


---
## Reference
[Overview of Java Built-in Annotations](https://www.baeldung.com/java-default-annotations)  
[스프링 #6. Custom Annotation](https://zamezzz.tistory.com/270)  
[자바 커스텀 어노테이션 만들기](https://advenoh.tistory.com/21)  
[[IT/트랜드] Java의 진화 ⑥ (Annotation – 1 편)](https://daitso.kbhub.co.kr/56470/)  
[Java에서 커스텀 어노테이션(Annotation) 만들고 사용하기](https://elfinlas.github.io/2017/12/14/java-custom-anotation-01/)  
