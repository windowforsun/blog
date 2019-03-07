--- 
layout: single
classes: wide
title: "[Spring 실습] POJO 레퍼런스를 자동연결(@Autowired)"
header:
  overlay_image: /img/spring-bg.jpg
subtitle: 'POJO 레퍼런와 자동 연결을 이용해 다른 POJO 와 연동해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - Autowired
    - POJO
---  

'POJO 레퍼런와 자동 연결을 이용해 다른 POJO 와 연동해보자'

# 목표
- 애플리케이션을 구성하는 POJO/빈 인스턴스들은 서로 함께 동작을 수행 한다.
- Annotation 을 붙여 POJO 레퍼런스와 자동연결을 한다.

# 방법
- 자바 설정 클래스에 정의된 POJO/빈 인스턴스들 사이의 참조 관계는 표준 자바 코드로도 가능하다.
- 필드, setter, 생성자 또는 다른 메서드에 @Autired 를 붙이면 POJO 레퍼런스를 자동 연결 할 수 있다.

# 예제
생성자, 필드, 프로퍼티로 자동 연결하는 방법을 차례로 소개하고 마지막에 자동 연결 관련 이슈의 해결 방법을 제시한다.

## 자바 설정 클래스에서 POJO 참조하기
- 자바 설정 클래스에 POJO 인스턴스를 정의하면, POJO 참조가 가능하다.
- 아래와 같이 빈 메서드를 호출해 빈 을 참조할 수 있다.

```java
@Configuration
public class SequenceConfiguration {

    @Bean
    public DatePrefixGenerator datePrefixGenerator() {
        DatePrefixGenerator dpg = new DatePrefixGenerator();
        dpg.setPattern("yyyyMMdd");
        return dpg;
    }

    @Bean
    public SequenceGenerator sequenceGenerator() {
        SequenceGenerator sequence = new SequenceGenerator();
        sequence.setInitial(100000);
        sequence.setSuffix("A");
        sequence.setPrefixGenerator(datePrefixGenerator());
        return sequence;
    }
}
```  

- SequenceGenerator 클래스의 prefixGenerator 프로퍼티를 DatePrefixGenerator 빈 인스턴스로 설정했다.
- DatePrefixGenerator 빈을 생성하고, 해당 빈 인스턴스를 참조하고 싶은 곳에(prefixGenerator) datePrefixGenerator() 를 일반 메서드 호출 하는 것처럼 사용하면 된다.


## POJO 필드에 @Autowired 를 붙여 자동 연결하기
- 서비스 객체를 생성하는 서비스 클래스는 실제로 자주 쓰이는 Best practice(모범 사례)로, DAO를 직접 호출 하는 대신 일종의 Facade(관문)을 두는 것이다.
- 서비스 객체는 내부적으로 DAO와 연동하며 시퀀스 생성 요청을 처리한다.

```java
@Service
public class SequenceService {

    @Autowired
    private SequenceDao sequenceDao;

    public SequenceService() {}

    public SequenceService(SequenceDao sequenceDao) {
        this.sequenceDao=sequenceDao;
    }

    public void setSequenceDao(SequenceDao sequenceDao) {
        this.sequenceDao = sequenceDao;
    }

    public String generate(String sequenceId) {
        Sequence sequence = sequenceDao.getSequence(sequenceId);
        int value = sequenceDao.getNextValue(sequenceId);
        return sequence.getPrefix() + value + sequence.getSuffix();
    }
}
```  

- SequenceService 클래스는 @Service Annotation 을 붙였기 때문에 스프링 빈으로 등록된다.
- 별도의 이름을 지정하지 않았으므로 클래스명 그대로 sequenceService 가 된다.
- sequenceDao 프로퍼티에 @Autowired 가 있기 때문에 sequenceDao 빈(SeqenceDaoImple 클래스)이 프로퍼티에 자동 연결된다.
- 배열형 프로퍼티에 @Autowired 를 붙이면 스프링은 매치된 빈을 모두 찾아 자동 연결한다.
- PrefixGenerator[] 프로퍼티에 @Autowired 를 적용하면 PrefixGenerator 와 타입이 호환되는 빈을 모두 찾아 자동 연결이 가능하다.

```java
public class SequenceGenerator {
	@Autowired
	private PrefixGenerator[] prefixGenerators;
}
```  

- IoC 컨테이너에 선언된 PrefixGenerator 와 타입이 호환되는 빈이 여러개 있어도 prefixGenerators 배열에 자동으로 추가된다.
- Type-Safe 컬렉션에 @Autowired 를 붙이면 스프링은 이컬렉션과 타입이 호환되는 빈을 모두 찾아 자동으로 연결한다.

```java
public class SequenceGenerator {
	@Autowired
	private List<PrefixGenerator> prefixGeneratorList;
}
```  

- 아래 처럼 Type-Safe한 java.util.Mapp 에 @Autowired 를 붙이면 스프링은 타입 호환되는 빈을 모두 찾아 빈 이름-빈 인스턴스(Key-Value) 로 매핑 시켜 추가한다.

```java
public class SequenceGenerator {
	@Autowired
	private Map<String, PrefixGenerator> prefixGeneratorMap;
}
```  

## @Autowired 로 POJO 메서드와 생성자를 자동 연결하기, 자동 연결을 선택적으로 적용하기
- @Autowired 는 POJO Setter 메서드에도 직접 적용 할 수 있다.
- prefixGenerator 프로퍼티 Setter 메서드에 @Autowired 를 붙이면 prefixGenerator 와 타입이 호환 되는 빈이 연결된다.

```java
public class SequenceGenerator {
	@Autowired
	public void setPrefixGenerator(PrefixGenerator prefixGenerator) {
		this.prefixGenerator = prefixGenerator;
	}
}
```  

- 스프링은 기본적으로 @Autowired 를 붙인 필수 프로퍼티에 해당하는 빈을 찾지 못하면 예외를 던진다.
- 선택적인 프로퍼티는 @Autowired 의 required 속성값을 false 로 지정해 스프링이 빈을 못찾더라도 그냥 지나치게 한다.

```java
public class SequenceGenerator {
	@Autowired(required = false)
	public void setPrefixGenerator(PrefixGenerator prefixGenerator) {
		this.prefixGenerator = prefixGenerator;
	}
}
```  

- @Autowired 는 메서드 인수의 이름과 개수에 상관없이 적용할 수 있다.
- 스프리은 각 메서드의 인수형과 호환되는 빈을 찾아 연결한다.

```java
public class SequenceGenerator {
	@Autowired
	public void myOwnCustomInjection(PrefixGenerator perfixGenerator) {
		this.prefixGenerator = prefixGenerator;
	}
}
```  

- 생성자에도 @Auwired 를 붙여 자동 연결 할 수 있다.
- 스프링은 생성자 인수가 몇 개든 각 인수형과 호환되는 빈을 연결한다.

```java
@Service
public class SequenceService {
	private final SequenceDao sequenceDao;
	
	@Autowired
	public SequenceService(SequenceDao sequenceDao) { 
		this.sequenceDao = sequenceDao;
	}
	
}
```  

- Spring 4.3 버전 부터 생성자가 하나뿐인 클래스의 생성자는 Autowired(자동연결) 하는 것이 기본이므로 굳이 @Autowired 를 붙이지 않아도 된다.

## Annotation 으로 모호한 자동 연결 명시하기
- 타입을 기준으로 자동 연결하면 IoC 컨테이너에 호환 타입이 여러개 존재하거나 프로퍼티가(Array, List, Map) 등의 그룹형이 아닐 경우 제대로 연결되지 않는다.
- 타입이 같은 반이 여러개 라면 @Primary, @Qualifier 로 해결 할 수 있다.
- @Primary 로 모호한 자동연결 명시하기
	- 스프링에서는 @Primary 를 붙여 Candidate(부호) 빈을 명시한다.
	- 여러 빈이 자동 연결 대상일 때 특정한 빈에 우선권을 부여하는 것이다.
	
	```java
	@Component
	@Primary
	public class DatePrefixGenerator implements PrefixGenerator {
	
	    @Override
	    public String getPrefix() {
	        DateFormat formatter = new SimpleDateFormat("yyyyMMdd");
	        return formatter.format(new Date());
	    }
	}
	```  

	- PrefixGenerator 인터페이스의 구현체인 DatePrefixGenerator 클래스에 @Primary 를 붙였기 때문에 PrefixGenerator 형의 빈 인스턴스가 여러개라도 스프링은 @Primary 를 붙인 클래스의 빈 인스턴스를 자동 연결한다.
- @Qualifier 로 모호한 자동 연결 명시하기
	- @Qualifier 에 이름을 주어 후보 빈을 명시할 수도 있다.

	```java
	public class SequenceGenerator {
		@Autowired
		@Qualifier("datePrefixGenerator")
		private PrefixGenerator prefixGenerator;
		
	}
	```  
	
	- 스프링은 IoC 컨테이너에서 이름이 datePrefixGenerator 인 빈을 찾아 prefixGenerator 프로퍼티에 연결한다.
	- @Qualifier 는 메서드 인수를 연결하는 쓰임새도 있다.
	
	```java
	public class SequenceGenerator {
		@Autowired
		public void myOwnCustomInjectionName(@Qualifier("datePrefixGenerator") PrefixGenerator prefixGenerator) {
			this.prefixGenerator = prefixGenerator;
		}
		
	}
	```  
	
## 여러 곳에 분산된 POJO 참조 문제 해결하기
- 애플리케이션 규모가 커질수록 모든 POJO 설정을 하나의 자바 설정 클래스에 담아두기 어렵기 때문에 POJO 기능에 따라 여러 자바 설정 클래스로 나누어 관리한다.
- 자바 설정 클래스가 여러개 공존하면 상이한 클래스에 정의된 POJO 를 자동으로 연결하거나 참조하는 일이 생각보다 간단하지 않다.
- 한 가지 방법은 자바 설정 클래스가 위치한 경로마다 ApplicationContext 를 초기화 하는 것이다.
- 각 자바 설정 클래스에 선언된 POJO 를 컨텍스트와 레퍼런스로 읽으면 POJO 간 자동 연결이 가능하다.

```java
AnnotationConfigurationContext context = new AnnotationConfigurationContext(PrefixConfiguration.class, SequenceGeneratorConfiguration.class);
```  

- @Import 로 구성 파일을 나누어 임포트 하는 방법도 있다.

```java
@Configuration
public class PrefixConfiguration {

    @Bean
    public DatePrefixGenerator datePrefixGenerator() {
        DatePrefixGenerator dpg = new DatePrefixGenerator();
        dpg.setPattern("yyyyMMdd");
        return dpg;
    }
}

@Configuration
@Import(PrefixConfiguration.class)
public class SequenceConfiguration {

    @Value("#{datePrefixGenerator}")
    private PrefixGenerator prefixGenerator;

    @Bean
    public SequenceGenerator sequenceGenerator() {
        SequenceGenerator sequence = new SequenceGenerator();
        sequence.setInitial(100000);
        sequence.setSuffix("A");
        sequence.setPrefixGenerator(prefixGenerator);
        return sequence;
    }
}
```  

- sequenceGenerator 빈에서는 반드시 prefixGenerator 빈을 설정해야 한다.
- 같은 설정 클래스에는 없고 다른 설정 클래스인 PrefixConfiguration 에 PrefixGenerator 가 정의되어 있다.
- @Import(PrefixConfiguration.class) 를 붙이면 PrefoxConfiguration 클래스에 정의한 POJO 를 모두 현재 설정 클래스의 스코프로 가져올 수 있다.
- 그리고 @Value 와 SpEL 을 써서 PrefixConfiguration 클래스에 선언된 datePrefixGenerator 빈을 prefixGenerator 필드에 Injection 시킨다.
- 결과적으로 sequenceGenerator 빈에서 prefixGenerator 빈을 가져다 쓸 수 있게 된 것이다.


---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  

	
	

