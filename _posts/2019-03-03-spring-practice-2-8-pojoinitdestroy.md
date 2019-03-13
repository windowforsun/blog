--- 
layout: single
classes: wide
title: "[Spring 실습] Annotation 으로 POJO 초기화/폐기(@PostConstruct, @PreDestroy)"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Annotation 을 사용해 POJO 초기화/폐기를 컨트롤하자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - A-PostConstruct
    - A-PreDestroy
    - A-Lazy
    - A-DependOn
    - IoC
    - spring-core
---  

# 목표
- 특정 POJO 는 사용하기 전에 특정 초기화 작업이 필요할 수 있다.
- 예를 들어 파일을 여려거나, 네트워크/DB 접속, 메모리 할당 등 이 될 수 있다.
- 이런 POJO 가 소멸할 때에도 동일하게 소멸 작업이 필요한 경우가 많다.
- IoC 컨테이너에서 빈을 초기화 및 폐기하는 로직을 컨트롤 해보자.

# 방법
- 자바 설정 클래스의 @Bean 정의부에서 initMethod, destroyMethod 속성을 설정하면 스프링은 이들을 각각 초기화, 폐기 콜백 메서드로 인지한다.
- POJO 메서드에 각각 @PostConstruct, @PreDestroy 를 붙여도 동일한 작업을 수행한다.
- 스프링에서는 @Lazy 를 붙여 Lazy Initialization(주어진 시점까지 빈 생성으르 미루는 기법)을 할 수 있고, @DependsOn 으로 빈을 생성하기ㅣ 전에 다른 빈을 먼저 생성하도록 강제 할 수 있다.

# 예제
- POJO 초기화/폐기 메서드는 @Bean 으로 정의한다.
- 쇼핑몰 애플리케이션에서 체크아웃 기능을 구현하려고 한다.
- 카트에 담긴 상품 및 체크아웃 시각을 텍스트 파일로 기록하는 기능을 Cashier 클래스에 추가한 코드이다.

```java
public class Cashier {

    private String fileName;
    private String path;
    private BufferedWriter writer;

    public void setFileName(String fileName) {
        this.fileName = fileName;
    }

    public void setPath(String path) {
        this.path = path;
    }

    public void openFile() throws IOException {

        File targetDir = new File(path);
        if (!targetDir.exists()) {
            targetDir.mkdir();
        }

        File checkoutFile = new File(path, fileName + ".txt");
        if (!checkoutFile.exists()) {
            checkoutFile.createNewFile();
        }

        writer = new BufferedWriter(new OutputStreamWriter(
                new FileOutputStream(checkoutFile, true)));
    }

    public void checkout(ShoppingCart cart) throws IOException {
        writer.write(new Date() + "\t" + cart.getItems() + "\r\n");
        writer.flush();
    }

    public void closeFile() throws IOException {
        writer.close();
    }
}
```  

- checkout() 메서드를 호출 할대마다 날짜와 카트 항목을 openFile() 의 텍스트 파일에 덧붙인다.
- 기록을 마치면 closeFile() 메서드는 파일을 닫고 시스템 리소스를 반납한다.
- Cachier 빈 생성 이전에는 openFile() 메서드, 폐기 직전에 closeFile() 메서드를 각각 실행하도록 자바 설정 파일을 작성한다.

```java
@Configuration
@ComponentScan("com.apress.springrecipes.shop")
public class ShopConfiguration {
	// ...

    @Bean(initMethod = "openFile", destroyMethod = "closeFile")
    public Cashier cashier() {

        String path = System.getProperty("java.io.tmpdir") + "/cashier";
        Cashier c1 = new Cashier();
        c1.setFileName("checkout");
        c1.setPath(path);
        return c1;
    }
}
```  

- @Bean 의 initMethod, destroyMethod 속성에 각각 초기화, 폐기 작업을 수행할 메서드를 지정했다.

## @PostConstruct, @PreDestroy 로 POJO 초기화/폐기 메서드 지정하기
- 자바 설정 클래스 외부에 (@Component 를 붙여) POJO 클래스를 정의할 경우에는 @PostsConstruct, @PreDestroy 를 해당 클래스에 직접 붙여 초기화/폐기 메서드를 지정한다.

```java
@Component
public class Cashier {

    @Value("checkout")
    private String fileName;

    @Value("c:/Windows/Temp/cashier")
    private String path;

    private BufferedWriter writer;

    public void setFileName(String fileName) {
        this.fileName = fileName;
    }

    public void setPath(String path) {
        this.path = path;
    }

    @PostConstruct
    public void openFile() throws IOException {
        File targetDir = new File(path);
        if (!targetDir.exists()) {
            targetDir.mkdir();
        }
        File checkoutFile = new File(path, fileName + ".txt");
        if (!checkoutFile.exists()) {
            checkoutFile.createNewFile();
        }
        writer = new BufferedWriter(new OutputStreamWriter(
                new FileOutputStream(checkoutFile, true)));
    }

    public void checkout(ShoppingCart cart) throws IOException {
        writer.write(new Date() + "\t" + cart.getItems() + "\r\n");
        writer.flush();
    }

    @PreDestroy
    public void closeFile() throws IOException {
        writer.close();
    }
}
```  

- openFile() 메서드에 @PostConstruct 가 선언되어 있기 때문에 빈 생성 이후에 메서드를 실행한다.
- closeFile() 메서드에 @PreDestroy 가 선언되어 있기 때문에 빈 폐기 이전에 메서드를 실행한다.

## @Lazy 로 느긋하게 POJO 초기화하기
- 기본적으로 스프링은 모든 POJO 를 Eager Initialization (조급한 초기화) 을 한다.
- 애플리케이션 구동과 동시에 POJO 를 초기화 한다.
- 느긋한 초기화는 필요에 따라 초기화 과정을 미루고, 나중에 초기화 하는 것이다.
- 느긋한 초기화를 하면 구동 시점에 리소스를 집중 소모하지 않아도 되므로 전체 시스템 리소스를 절약할 수 있다.
- 무거운 작업(네트워크 접속, 파일 처리 등)을 처리하는 POJO 같은 경우에는 느긋한 초기화가 효율적이다.
- 빈에 @Lazy 를 붙이며녀 느긋한 초기화가 적용된다.

```java
@Component
@Scope("prototype")
@Lazy
public class ShoppingCart {

    private List<Product> items = new ArrayList<>();

    public void addItem(Product item) {
        items.add(item);
    }

    public List<Product> getItems() {
        return items;
    }
}
```  

- @Lazy 선언을 통해 애플리케이션이 요구하거나 다른 POJO 가 참조하기 전까진 초기화 되지 않는다.

## @DependOn 으로 초기화 순서 정하기
- POJO  가 늘어나면 그에 따라 POJO 초기화 횟수도 증가한다.
- 여러개의 자바 설정 클래스에 분산 선언된 많은 POJO 가 서로를 참조하게 되면 Race Condition 이 일어나기 쉽다.
- B, F 빈의 로직이 C 빈에서 필요할때, 아직 스프링이 B, F 을 초기화하지 않았는데 C 빈이 먼저 초기화 되면 에러가 발생한다.
- @DependOn Annotation 은 빈 초기화 순서를 보장하며 어떤 POJO 가 다른 POJO 보다 먼저 초기화되도록 강제하고 에러가 발생하더라도 쉬운 메시지를 돌려준다.

```java
@Configuration
public class SequenceConfiguration {

    @Bean
    @DependsOn("datePrefixGenerator")
    public SequenceGenerator sequenceGenerator() {
        SequenceGenerator sequence = new SequenceGenerator();
        sequence.setInitial(100000);
        sequence.setSuffix("A");
        return sequence;
    }
}
```  

- @DependOn("dataPrefixGenerator") 를 붙였기 때문에 dataPrefixGenerator 빈은 sequenceGenerator 빈 보다 반드시 먼저 생성된다.
- @DependOn 속성값으로는 { } 로 둘러싼 CSV 리스트 형태로 의존하는 빈을 여러개 지정 가능하다.
	- @DependOn({"dataPrefixGenerator, numberPrefixGenerator, randomPrefixGenerator"})
	
---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
