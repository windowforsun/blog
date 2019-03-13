--- 
layout: single
classes: wide
title: "[Spring 실습] POJO 를 생성할 때 검증/수정 하기"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: '@PostConstruct 와 @Required 를 사용해 POJO 검증/수정을 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - Practice
    - A-Required
    - BeanPostProcessor
    - IoC
    - spring-core
---  

# 목표
- 모든 빈 인스턴스, 특정 타입의 인스턴스를 생성할 때 해당 빈 프로퍼티를 어떤 기준에 따라 검증/수정 한다.

# 방법
- Bean Post-Processor(빈 후처리기) 를 이용하면 초기화 콜백 메서드(@Bean 의 initMethod 혹은 @PostConstruct) 전후에 원하는 오직을 빈에 적용할 수 있다.
- Bean Post-Processor 의 특징은 IoC 컨테이네 내부의 모든 빈인스턴스를 대상으로 한다는 점이다.
- Bean Post-Processor 는 빈 프로퍼티가 올바른지 체크하거나 어떤 기준에 따라 빈 프로퍼티를 변경 또는 전체 빈 인스턴스를 상대로 어떤 작업을 수행하는 용도로 쓰인다.
- @Required 는 스프링에 내장된 RequiredAnnotationBeanPostProcessor 가 지원하는 Annotation 이다.
- RequiredAnnotationBeanPostProcessor 후처리기는 @Required 를 붙인 모든 빈 프로퍼티가 설정되었는지 확인한다.

# 예제
- Bean Post-Processor 를 이용하면 기존 POJO 코드를 건드리지 않고 빈 검증을 수행할 수 있다.

## 모든 빈 인스턴스를 처리하는 Post-Processor 생성하기
- 빈 후처리기는 BeanPostProcessor 인터페이스를 구현한 객체이다.
- 이 인터페이스를 구현한 빈은 스프링은 자신이 관리하는 모든 빈 인스턴스에 postProcessorBeforeInitialization(), postProcessorAfterInitialization() 두 메서드를 적용한다.
- 빈 상태를 조사, 수정, 확인 하는 등 어떤 로직를 위 메서드에 넣어 처리 할 수 있다.

```java
@Component
public class AuditCheckBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        System.out.println("In AuditCheckBeanPostProcessor.postProcessBeforeInitialization, processing bean type: " + bean.getClass());
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        return bean;
    }
}
```  

- postProcessorBeforeInitialization(), postProcessorAfterInitialization() 머세드는 세부 구현사항이 없더라도 원본 빈 인스턴스를 리턴해야 한다.
- 클래스 레벨에 @Component 를 붙이면 애플리케이션 컨텍스트에 빈 후처리기로 등록된다.
- 애플리케이션 컨텍스트는 BeanPostProcessor 구현 빈을 감지해 컨테이너안에 있는 다른 빈 인스턴스에 일괄 적용한다.

## 주어진 빈 인스턴스만 처리하는 후처리기 생성하기
- IoC 컨테이너는 자신이 생성한 빈 인스턴스를 모두 하나씩 빈 후처리기에 넘긴다.
- 만약 특정 타입의 빈만 후처리기를 적용하려면 인스턴스 타입을 체크하는 필터를 이용해야 한다.
- 예를 들어 Product 형 빈인스턴스만 빈 후처리기에 적용한 코드는 아래외 같다.

```java
@Component
public class ProductCheckBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof Product) {
            String productName = ((Product) bean).getName();
            System.out.println("In ProductCheckBeanPostProcessor.postProcessBeforeInitialization, processing Product: " + productName);
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
            throws BeansException {
        if (bean instanceof Product) {
            String productName = ((Product) bean).getName();
            System.out.println("In ProductCheckBeanPostProcessor.postProcessAfterInitialization, processing Product: " + productName);
        }
        return bean;
    }
}
```  

- 앞서 언급한 postProcessorBeforeInitialization(), postProcessorAfterInitialization() 메서드는 반드시 해당 인스턴스를 반환해야 한다는 말을 바꿔말하면 원본 빈 인스턴스를 다른 인스턴스로 변경가능하다는 뜻이다.

## @Required 로 프로퍼티 검사하기
- 특정 빈 프로퍼티가 설정되었는지 체크하고 싶은 경우에는 커스텀 후처리기를 작성하고 해당 프로퍼티에 @Required 를 붙인다.
- 스프링 빈 후처리기 RequiredAnnotationBeanPostProcessor 는 @Required 를 붙인 프로퍼티값이 설정 되었는지 확인한다.
- 프로퍼티값의 설정 여부만 체크할 뿐, 값이 null 인지, 다른 값인지는 확인하지 않는다.
- 예를 들어 prefixGenerator, suffix 는 시퀀스 생성기의 필수 프로퍼티이므로 아래 코드 처럼 @Required 를 붙인다.

```java
public class SequenceGenerator {

    private PrefixGenerator prefixGenerator;
    private String suffix;
    
    // ...
    
    @Required
    public void setPrefixGenerator(PrefixGenerator prefixGenerator) {
        this.prefixGenerator = prefixGenerator;
    }

    @Required
    public void setSuffix(String suffix) {
        this.suffix = suffix;
    }

	// ...
}
```  

- @Required 를 붙인 프로퍼티는 스프링이 감지해 값의 존재 여부를 검사하고 프로퍼티값이 없으면 BeanInitializationException 예외를 던진다.


---
## Reference
[스프링5 레시피](https://book.naver.com/bookdb/book_detail.nhn?bid=13911953)  
