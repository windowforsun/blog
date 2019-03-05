--- 
layout: single
classes: wide
title: "[Spring4 입문] Spring DI - Annotation"
header:
  overlay_image: /img/spring-bg.jpg
subtitle: 'Spring DI Annotation 설정법은 무엇이고, 어떻게 사용하는지'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Web
    - Spring
    - DI
    - Annotation
    - Spring4 입문
---  

'Spring DI Annotation 설정법은 무엇이고, 어떻게 사용하는지'

## 간단하게 보기 @Autowired, @Component

```java
public interface ProductService {
	void addProduct(Product product);
	Product findByProductName(String name);
}
```  

```java
@Component		// @1
public class ProductServiceImpl implements ProductService {
	@Autowired		// @2
	private ProductDao productDao;
	
	@Override
	public void addProduct(Product product) {
		this.productDao.addProduct(product);
	}
	
	@Override
	public Product findByProductName(String name) {
		return this.productDao.findByProductName(name); 
	}
}
```  

```java
public interface ProductDao {
	void addProduct(Product product);
	Product findByProductName(String name);
}
```  

```java
@Component		// @3
public class ProductDaoImpl implements ProductDao {
	// 간단한 DAO 구현을 위해 실제 데이터베이스 연동은 하지 않고 Map을 사용한다.
	private Map<String, Product> storage = new HashMap<String, Product>();		// @4
	
	@Override
	public void addProduct(Product product) {
		this.storage.put(product.getName(), product);
	}
	
	@Override
	public Product findByProductName(String name) {
		return this.storage.get(name);
	}
}
```  

- 인스턴스 변수 앞에 @Autowired를 붙이면 DI 컨테이너가 인스턴스 변수의 타입에 대입할 수 있는 클래스를 @Component가 붙은 클래스 중에서 찾아내 인젝션 해준다.
- 접근 제어자가 private 이라도 별도의 setter없이 인젝션이 가능하다.
- DI 컨테이너는 기본적으로 타입에 따라 수행하는 byType을 통한 인젝션을 사용한다.
	- @Component가 선언된 클래스가 여러 개 있더라도, 타입이 다를경우 @Autowired가 붙은 인스턴스 변수에 인젝션이 되지 않는다.
- Annotation이 선언된 클래스파일 외에도 XML로 기술된 Bean 정의파일 혹은 JavaConfig 파일이 필요하다.
- 우선 DI 컨테이너를 사용하기 위한 XML로 기술된 Bean 정의파일은 아래와 같다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
  xsi:schemaLocation="
     http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans.xsd
     http://www.springframework.org/schema/context
     http://www.springframework.org/schema/context/spring-context.xsd">
  <context:annotation-config />
  <context:component-scan base-package="sample.di.business" />		// @5
  <context:component-scan base-package="sample.di.dataaccess" />		// @6
</beans>
```  

