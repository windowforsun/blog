--- 
layout: single
classes: wide
title: "[Spring 실습] Java로 POJO 구성하기 (Annotation)"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Annotation 을 이용해 POJO 관리하기'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Annotation
    - A-Component
    - A-ComponentScan
    - POJO
    - IoC
    - spring-core
---  

# 목표
Spring IoC 컨테이너에서 Annotation 을 이용해 POJO 를 관리하자

# 방법
- POJO 클래스를 설계한다.
- @Configuration, @Bean 을 붙인 자바 설정 클래스를 만들거나, @Component, @Repository, @Service, @Controller 등을 붙인 자바 컴포넌트를 구성한다.
- IoC 컨터이너는 Annotation 을 붙인 자바 클래스를 스캐닝(탐색)하여 애플리케이션의 이리부인 것처럼 POJO 인스턴스(Bean)을 구성한다.

# 예제
- 다목적 시퀀스(Sequence) 생성기 애플리케이션을 개발한다.
- 시퀀스 유형별로 접두어(Prefix), 접미어(Suffix), 초깃값은 따로 있어서 생성기 인스턴스를 여러개 만들어야 한다.
- Bean을 생성하는 POJO 클래스를 자바 설정으로 작성한다.

```java
public class SequenceGenerator {
	private String prefix;
	private String suffix;
	private int initial;
	private final AtomicInteger counter = new AtomicInteger();
	
	public SequenceGenerator(){
	}
	
	public void setPrefix(String prefix) {
		this.prefix = prefix;
	}
	
	public void setSuffix(String suffix) {
		this.suffix = suffix;
	}
	
	public void setInitial(int initial) {
		this.initial = initial;
	}
	
	public String getSequence() {
		StringBuilder builder = new StringBuilder();
		builder.append(this.prefix)
				.append(this.initial)
				.append(this.counter.getAndIncrement())
				.append(this.suffix);
		return builder.toString();
	}
}
```  

- 시퀀스 요건에 따라 prefix, suffix, initial 세 프로퍼티를 지닌 SequenceGenerator 클래스이다.
	- 인스턴스에서 getSequence() 메서드를 호출하면 그때마다 prefix와 suffix가 조합된 마지막 시퀀스를 리턴한다.

## 설정 클래스에서 @Configuration 과 @Bean 을 붙여 POJO 생성하기
- 자바 설정 클래스에 초깃값을 지정하면 IoC 컨테이너에서 POJO 인스턴스를 정의할 수 있다.

```java
@Configuration
public class SequenceGeneratorConfiguration {

    @Bean
    public SequenceGenerator sequenceGenerator() {

        SequenceGenerator seqgen = new SequenceGenerator();
        seqgen.setPrefix("30");
        seqgen.setSuffix("A");
        seqgen.setInitial(100000);
        return seqgen;
    }
}
```  

- SequenceGeneratorConfiguration 클래스의 @Configuration은 이 클래스가 설정 클래스임을 스프링에게 알린다.
- 스프링은 @Configuration 이 달린 클래스를 보면 일단 그 안에서 빈 인스턴스 정의부, @Bean(빈 인스턴스를 생성해 반환해주는) 자바 메서드를 찾는다.
- 설정 클래스의 메서드에 @Bean 을 붙이면 그 메서드와 동일한 이름의 빈이 생성된다.
- 빈 이름을 따로 명시하려면 @Bean 의 name 속성에 적는다.
	- @Bean(name ="myBean")는 myBean 이라는 이름의 빈을 만든다.
	
## IoC 컨테이너를 초기화하여 Annotation 스캐닝하기
- Annotation 을 붙인 자바 클래스를 스캐닝하려면 우선 IoC 컨테이너를 인스턴스화 해야 스프링이 @Configuration, @Bean 을 읽고 IoC 컨테이너에서 빈 인스턴스를 가지고 올 수 있다.
- 스프링은 기본 구현체인 빈 팩토리(Bean Factory) 이와 호환되는 고급 구현체인 애플리케이션 컨텍스트(Application Context) 두 가지 IoC 컨테이너를 제공한다.
	- 설정 파일은 두 컨테이너 모두 동일하다.
- ApplicationContext 는 기본 기능에 출실하면서도 BeanFactory 보다(스프링을 애플릿이나 모바일 기기에서 실행하는 등) 발전된 기능을 지니고 있으므로 리소스에 제약을 받는 상황이 아니라면 가급적 애플리케이션 컨텍스트를 사용하는 것이 좋다.
- BeanFactory 와 ApplicationContext 는 각각 인터페이스로 설정되어 있다.
- ApplicationContext 는 BeanFactory 의 하위 인터페이스여서 호환성이 보장된다.
- ApplicationContext 는 인터페이스이므로 구현체가 필요하다.
	- 스프링은 여러 구현체가 있지만, 그 중 유연한 AnnotationConfigApplicationContext 를 권장한다.

```java
ApplicationContext context = new AnnotationConfigApplicationContext(SequenceGeneratorConfiguration.class);
```  

- ApplicationContext 를 인스턴스화한 이후에 객체 레퍼런스(여기선 context)는 POJO 인스턴스 또는 빈에 액세스하는 창구 역할을 수행한다.

## IoC 컨테이너에서 POJO 인스턴스/빈 가져오기
- 설정 클래스에서 선언된 빈을 빈 팩토리 또는 ApplicationContext 에서 가져오려면 유일한 빈이름을 getBean() 메서드의 인수로 호출한다.
- getBean() 메서드는 java.lang.Object 형을 반환하므로 실제 타입에 맞게 캐스팅 해야 한다.

```java
SequenceGenerator generator = (SequenceGenerator) context.getBean("sequenceGenerator");
```  

- 캐스팅을 안하려면 getBean() 메서드의 두 번째 인수에 빈 클래스 명을 지정한다.

```java
SequenceGenerator generator = context.getBean("sequenceGenerator", SequenceGenerator.class);
```  

- 해당 빈의 클래스 명에 해당되는 빈이 하나 뿐이라면 이름을 생략할 수 있다.

```java
SequenceGenerator generator = context.getBean(SequenceGenerator.class);
```  

- 위와 같은 방식으로 POJO 인스턴스/빈을 스프링 외부에서 생성자를 이용해 일반적인 객체처럼 사용할 수 있다.
- 아래는 시퀀스 생성기를 실행하는 Main 클래스이다.

```java
public class Main {

    public static void main(String[] args) {
        ApplicationContext context =
                new AnnotationConfigApplicationContext(SequenceGeneratorConfiguration.class);

        SequenceGenerator generator = context.getBean(SequenceGenerator.class);

        System.out.println(generator.getSequence());
        System.out.println(generator.getSequence());
    }
}
```  

- 출력결과

```
301000000A
301000001A
```  

## POJO 클래스에 @Component 를 붙여 DAO 빈 생성하기
- 위 예제에서는 자바 설정 클래스에서 값을 하드코딩해 스프링 빈을 인스턴스화 했다.
- 실제로 POJO 는 대부분 DB 나 유저 입력을 활용해 인스턴스로 만든다.
- 이에 맞춰 Domain 클래스 및 DAO(Data Access Object) 패턴을 이용해 POJO 를 생성 한다.
- Sequence Domain 클래스

```java
public class Sequence {

    private final String id;
    private final String prefix;
    private final String suffix;

    public Sequence(String id, String prefix, String suffix) {
        this.id = id;
        this.prefix = prefix;
        this.suffix = suffix;
    }

    public String getId() {
        return id;
    }

    public String getPrefix() {
        return prefix;
    }

    public String getSuffix() {
        return suffix;
    }

}
```  

- DB 액세스를 처리하는 DAO 인터페이스

```java
public interface SequenceDao {

    Sequence getSequence(String sequenceId);

    int getNextValue(String sequenceId);
} 
```  

- getSequence() 메서드는 주어진 ID로 DB 테이블을 찾아 POJO 나 Sequence 객체를 반환한다.
- getNextValue() 메서드는 다음 시퀀스 값을 반환한다.
- SequenceDao 인터페이스를 구현하는 SequenceDaoImpl 클래스

```java
@Component("sequenceDao")
public class SequenceDaoImpl implements SequenceDao {

    private final Map<String, Sequence> sequences = new HashMap<>();
    private final Map<String, AtomicInteger> values = new HashMap<>();

    public SequenceDaoImpl() {
        sequences.put("IT", new Sequence("IT", "30", "A"));
        values.put("IT", new AtomicInteger(10000));
    }

    @Override
    public Sequence getSequence(String sequenceId) {
        return sequences.get(sequenceId);
    }

    @Override
    public int getNextValue(String sequenceId) {
        AtomicInteger value = values.get(sequenceId);
        return value.getAndIncrement();
    }
}
```  

- 편의상 Sequence 인스턴스 및 value 값을 저장은 Map에 하드 코딩 하였다.
- SequenceDaoImpl 클래스에 @Component("sequenceDao") 를 붙이면 스프링은 이 클래스를 이용해 POJO를 생성한다.
	- @Component 에 넣은 sequenceDao 의 이름으로 빈 인스턴스 ID가 설정된다.
	- 값이 없다면 소문자로 시작하는 클래스명을 빈 인스턴스 ID로 설정한다.
		- SequenceDaoImpl 클래스로 생성한 빈의 이름은 sequenceDaoImpl 이 된다.
- getSequence() 메서드는 sequenceId 에 해당하는 현재 시퀀스 값을, getNextValue() 메서드는 그다음 시퀀스 값을 반환한다.
- @Component 는 스프링이 발견할 수 있게 POJO 에 붙이는 범용 Annotation 이다.
- 스프링에는 Persistent(영속화), Service, Presentation(표현)으로 구성된 3 Layer(계층)가 있다.
	- @Repository, @Service, @Controller 는 각각 이 3 Layer 를 가리키는 Annotation 이다.
- POJO 의 쓰임새가 명확하지 않을 땐 @Component 를 붙여도 되지만, 특정 용도에 맞는 부가 기능(혜택)을 위해서 구체적으로 명시하는 것이 좋다.
	- @Repository 는 발생한 예외를 DataAccessException 으로 감싸 던지므로 디버깅 시 유리하다.

## Annotation 을 스캐닝하는 필터로 IoC 컨터이너 초기화 하기
- 기본적으로 스프링은 @Configuration, @Bean, @Component, @Repository, @Service, @Controller 가 달린 클래스를 모두 스캐닝 한다.
- 하나 이상의 포함/제외 필터를 적용해서 스키닝 과정을 커스터마이징 할 수 있다.
- 특정 Annotation 을 붙인 POJO 를 스프링 ApplicationContext 에서 넣거나 뺄 수 있다.
- 스프링이 지원하는 필터 표현식은 네 종류이다.
	- annotation, assignable 은 각각 필터 대상 Annotation 타입 및 클래스/인터페이스를 지정한다.
	- regex, aspectj 는 각각 정규표현식과 AspectJ 포인트컷 표현식으로 클래스를 매치하는 용도로 쓰인다.
	- use-default-filters 속성으로 기본 필터를 해제할 수도 있다.

```java
@Configuration
@ComponentScan(
        includeFilters = {
                @ComponentScan.Filter(
                        type = FilterType.REGEX,
                        pattern = {"com.apress.springrecipes.sequence.*Dao", "com.apress.springrecipes.sequence.*Service"})
        },
        excludeFilters = {
                @ComponentScan.Filter(
                        type = FilterType.ANNOTATION,
                        classes = {org.springframework.stereotype.Controller.class}) }
                )
public class SequenceGeneratorConfiguration {

}
```  

- 위와 같이 선언할 경우 com.apress.springrecipes.sequence 패키지에 속한 클래스 중 이름에 Dao, Service 가 포함된 것들은 모두 넣고 @Controller 를 붙인 클래스는 제외 하게 된다.
- includeFilters 에 추가할 경우 Annotation 이 선언되지 않은 클래스도 스프링이 자동 감지하게 된다.

## IoC 컨테이너에서 POJO 인스턴스/빈 가져오기
- 적용한 예제를 실행 시키는 Main 클래스 이다.

```java
public class Main {

    public static void main(String[] args) {

        ApplicationContext context =
                new AnnotationConfigApplicationContext("com.apress.springrecipes.sequence");

        SequenceDao sequenceDao = context.getBean(SequenceDao.class);

        System.out.println(sequenceDao.getNextValue("IT"));
        System.out.println(sequenceDao.getNextValue("IT"));
    }
}
```  

- 출력결과

```
10000
10001
```  


---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  

