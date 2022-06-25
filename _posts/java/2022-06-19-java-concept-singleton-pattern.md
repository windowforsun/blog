--- 
layout: single
classes: wide
title: "[Java 개념] 싱글턴(Singleton)"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'JVM 내 단일 인스턴스를 보장하는 싱글턴 구현 방법과 주의 사항에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - Effective Java
  - Singleton Pattern
toc: true 
use_math: true
---  

## 싱글턴(Singleton)
`Singleton` 은 인스턴스를 오직 하나만 생성할 수 있는 클래스를 의미한다. 
하나의 애플리케이션을 관리하는 매니저 역할 클래스나, 시스템 컴포넌트 등이 예시가 될 수 있다.  

`Java` 에서 `Singleton` 을 구현할 수 있는 방법은 아래와 같은 방법들이 있다. 

1. `public static final` 필드 사용
2. `생성자 예외` 사용
3. `정적 팩토리 매소드` 사용
4. `readResolve` 메소드 사용
5. `Enum` 사용
6. `Lazy-Initialization` 사용
7. `Lazy` + `synchronized` 키워드 사용
8. `Lazy` + `Checking Lock` 사용
9. `Lazy` + `DCL(Double checking lock)` 사용

그리고 위 방법들을 사용해서 `Singleton` 을 구현해서 사용할 때 발생할 수 있는 모든 문제점이자 고민이 필요한 부분들을 나열하면 아래와 같다.    

1. `reflection` 사용시 단일 인스턴스 보장
2. `직렬화/역직렬화` 수행시 단일 인스턴스 보장
3. 불필요한 초기 리소스 최소화
4. `Multi-Thread` 환경에서 단일 인스턴스 보장
5. `Multi-Thread` 환경에서 단일 인스턴스 보장을 위한 성능 저하

나열한 `Singleton` 구현 방법과 문제점들에 대해서 예시를 보며 더 자세히 알아본다. 

### `public static final` 필드 사용

```java
public class Singleton implements Serializable {
    public static final Singleton INSTANCE = new Singleton();

    private Singleton() {
    }
    
    // some methods
}
```  

가장 간단한 `Singleton` 구현으로 `public static final` 키워드를 사용해서 `Singleton` 클래스의 인스턴스를 외부로 노출하고, 
생성자는 `private` 으로 설정해 외부에서 인스턴스 생성을 막는 방법이다.  

```java
public class PublicStaticFinal_SingletonTest {
    @Test
    public void singleton_sameInstance() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;

        assertThat(singleton1, sameInstance(singleton2));
    }

    @Test
    public void reflection_notSameInstance() throws Exception {
        Singleton singleton1 = Singleton.INSTANCE;
        Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton singleton2 = constructor.newInstance();

        assertThat(singleton1, not(sameInstance(singleton2)));
    }

    @Test
    public void serialize_notSameInstance() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;

        assertThat(singleton1, sameInstance(singleton2));


        singleton2 = (Singleton) Util.deserialize(Util.serialize(singleton2));

        assertThat(singleton2, notNullValue());
        assertThat(singleton1, not(sameInstance(singleton2)));
    }
}
```  

실제 인스턴스는 클래스 로드 시점에 한번 초기화 되고 생성자는 외부에서 호출 불가하기 때문에 시스템에서 단일 인스턴스 임은 보장이 된다. 
하지만 `reflection` 을 통해 `private` 생성자를 강제로 호출하게 될 경우 단일 인스턴스는 보장할 수 없게 된다는 문제점이 있다.
그리고 `Serialize` 를 수행할 때는 역직렬화를 할때 새로운 인스턴스가 만들어지는 상황이다.  

`INSTANCE` 는 클래스 로드 시점에 딱 한번만 초기화 되는 구조이므로 `Multi-Thread` 환경에서도 단일 인스턴스를 보장 할 수는 있다. 
하지만 해당 클래스가 실제로 사용되지 않더라도 인스턴스는 생성되기 때문에 불필요한 리소스를 초기에 사용 중인 상황이다.

구분|보장 여부
---|---
`reflection` 사용시 단일 인스턴스 보장|X
`직렬화/역직렬화` 수행시 단일 인스턴스 보장|X
불필요한 초기 리소스 최소화|X
`Multi-Thread` 환경에서 단일 인스턴스 보장|O
`Multi-Thread` 환경에서 단일 인스턴스 보장을 위한 성능 저하 최소화|O


### `생성자 예외` 사용
`Singleton` 구현시 `reflection` 에 의한 강제적인 인스턴스 생성 동작을 방어하는 방법은 생성자에 예외처리 코드를 추가하는 것이다.  

```java
public class Singleton implements Serializable {
    public static final Singleton INSTANCE = new Singleton();

    private Singleton() {
        if(INSTANCE != null) {
            throw new RuntimeException("instance already exists");
        }
    }

    // some methods
}
```  

생성자를 강제로 호출 했을 때 인스턴스 존재여부를 검사해 예외를 던지도록 해서 `reflection` 수행시에도 단일 인스턴스를 보장한다.  

```java
public class SingletonTest {
    @Test
    public void singleton_sameInstance() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;

        assertThat(singleton1, sameInstance(singleton2));
    }

    @Test
    public void reflection_thowsException() throws NoSuchMethodException {
        Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);

        Assertions.assertThrows(InvocationTargetException.class, () -> constructor.newInstance());
    }

    @Test
    public void serialize() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;

        assertThat(singleton1, sameInstance(singleton2));

        singleton2 = (Singleton) Util.deserialize(Util.serialize(singleton2));

        assertThat(singleton2, notNullValue());
        assertThat(singleton1, not(sameInstance(singleton2)));
    }
}
```  

생성자에서 `INSTANCE` 전역 변수에 인스턴스 생성 여부를 검사해서 존재하는 경우 예외를 던져 하나 이상의 인스턴스 생성을 막기 때문에, 
`reflection` 으로부터 안전하다.  


구분|보장 여부
---|---
`reflection` 사용시 단일 인스턴스 보장|O
`직렬화/역직렬화` 수행시 단일 인스턴스 보장|X
불필요한 초기 리소스 최소화|X
`Multi-Thread` 환경에서 단일 인스턴스 보장|O
`Multi-Thread` 환경에서 단일 인스턴스 보장을 위한 성능 저하 최소화|O


### `readResolve` 메소드 사용
`readResolve` 메소드를 사용해서 직렬화/역직렬화 일때도 동일한 인스턴스임을 보장 할 수 있다. 

```java
public class Singleton implements Serializable {
    public static final Singleton INSTANCE = new Singleton();

    private Object readResolve() {
        return INSTANCE;
    }

    // some methods
}
```  

```java
public class SingletonTest {
    @Test
    public void singleton_sameInstance() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;

        assertThat(singleton1, sameInstance(singleton2));
    }

    @Test
    public void reflection_notSameInstance() throws Exception {
        Singleton singleton1 = Singleton.INSTANCE;
        Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton singleton2 = constructor.newInstance();

        assertThat(singleton1, not(sameInstance(singleton2)));
    }

    @Test
    public void serialize_sameInstance() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;
        assertThat(singleton1, sameInstance(singleton2));

        singleton2 = (Singleton) Util.deserialize(Util.serialize(singleton2));
        assertThat(singleton1, sameInstance(singleton2));
    }
}
```  

`readResolve()` 메소드를 선언하고 현재 생성된 인스턴스를 리턴하도록 한다. 
그러면 `ObjectInputstream` 이 `Stream` 에서 `Object` 를 읽을 때 호출하게 되면서 `직렬화/역직렬화` 과정에서도 동일한 인스턴스임을 보장 할 수 있다. 

구분|보장 여부
---|---
`reflection` 사용시 단일 인스턴스 보장|O
`직렬화/역직렬화` 수행시 단일 인스턴스 보장|O
불필요한 초기 리소스 최소화|X
`Multi-Thread` 환경에서 단일 인스턴스 보장|O
`Multi-Thread` 환경에서 단일 인스턴스 보장을 위한 성능 저하 최소화|O

### `정적 팩토리 메소드` 사용
`Singleton` 구현의 다른 방법은 `정적 팩토리 메소드` 를 사용해 인스턴스를 제공하는 것이다.

```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();

    public static Singleton getInstance() {
        return INSTANCE;
    }

    // some methods
}
```  

`INSTANCE` 전역 변수는 `private` 으로 선언하고 `getInstance()` 전역 메소드를 사용해서 인스턴스를 외부로 공개한다.

```java
public class SingletonTest {

    @Test
    public void singleton_sameInstance() {
        Singleton singleton1 = Singleton.getInstance();
        Singleton singleton2 = Singleton.getInstance();

        assertThat(singleton1, sameInstance(singleton2));
    }

    @Test
    public void reflection_notSameInstance() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        Singleton singleton1 = Singleton.getInstance();
        Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        Singleton singleton2 = constructor.newInstance();

        assertThat(singleton1, not(sameInstance(singleton2)));
    }

    @Test
    public void serialize_notSameInstance() {
        Singleton singleton1 = Singleton.getInstance();
        Singleton singleton2 = Singleton.getInstance();
        assertThat(singleton1, sameInstance(singleton2));

        singleton2 = (Singleton) Util.deserialize(Util.serialize(singleton2));
        assertThat(singleton1, not(sameInstance(singleton2)));
    }
}
```  

`정적 팩토리 메소드` 를 사용한 구현 방법은 기존에 알려진 어떤 이슈를 개선하는 방법은 아니다. 
`reflection`, `직렬화/역직렬화` 에 대한 이슈를 방어하고 싶다면 개선 코드를 동일하게 추가해 줘야 한다.  

`정적 팩토리 메소드` 를 사용하게 되면 `API` 변경 없이 `Singleton` 에서 `Singleton` 이 아니도록 변경하기 용의하다는 점이다. 
`getInstance()` 메소드에서 매번 새로운 인스턴스를 리턴하도록 하면 된다. 
그리고 제네릭을 활용해서 다양한 타입의 `Singleton` 인스턴스를 제공할 수도 있다. 
마지막으로는 `Singleton::getInstance` 와 같은 메소드 참조 공급자로 사용할 수도 있다는 점이 있다.  


구분|보장 여부
---|---
`reflection` 사용시 단일 인스턴스 보장|X
`직렬화/역직렬화` 수행시 단일 인스턴스 보장|X
불필요한 초기 리소스 최소화|X
`Multi-Thread` 환경에서 단일 인스턴스 보장|O
`Multi-Thread` 환경에서 단일 인스턴스 보장을 위한 성능 저하 최소화|O


### `Enum` 사용
앞선 방법에서 `reflection` 을 방지 하기 위해 생성자에 예외처리를 추가했고, 
`직렬화/역직렬화` 에 대한 이슈를 위해 `readResolve()` 메소드를 추가 했다. 
하지만 `Enum` 을 사용한 `Singleton` 을 구현하게 되면 이런 추가적인 코드 추가 없이 해당 이슈들을 모두 해결 가능하다.  

```java
public enum Singleton {
    INSTANCE;

    // some methods
}
```  

`public` 필드 방식과 매우 유사하지만 더 간결하고 간편하게 안전한 `Singleton` 구현이 가능하다.  

```java
public class SingletonTest {
    @Test
    public void singleton_sameInstance() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;

        assertThat(singleton1, sameInstance(singleton2));
    }

    @Test
    public void reflection_reflectionSafe() {
        Assertions.assertThrows(NoSuchMethodException.class, () -> Singleton.class.getDeclaredConstructor());
    }

    @Test
    public void serialize_sameInstance() {
        Singleton singleton1 = Singleton.INSTANCE;
        Singleton singleton2 = Singleton.INSTANCE;

        assertThat(singleton1, sameInstance(singleton2));

        singleton2 = (Singleton) Util.deserialize(Util.serialize(Singleton.INSTANCE));
        assertThat(singleton1, sameInstance(singleton2));
    }
}
```  

`Enum` 이라는 문법적 특성을 이용하는 방법이이 때문에, 어떤 코드의 추가도 없이
`reflection` 을 통한 새로운 인스턴스 생성, `직렬화/역직렬화` 시 새로운 인스턴스 생성과 같은 이슈들을 손쉽게 해결 할 수 있다.  

하지만 `Enum` 기반으로 `Singleton` 은 클래스를 상속할 수 없다는 단점이 있다. 
물론 `Enum` 이 인터페이스를 구현하도록 하는 구조는 가능하다.  


구분|보장 여부
---|---
`reflection` 사용시 단일 인스턴스 보장|O
`직렬화/역직렬화` 수행시 단일 인스턴스 보장|O
불필요한 초기 리소스 최소화|X
`Multi-Thread` 환경에서 단일 인스턴스 보장|O
`Multi-Thread` 환경에서 단일 인스턴스 보장을 위한 성능 저하 최소화|O


> 참고 [Singleton]({{site.baseurl}}{% link _posts/designpattern/2020-01-11-designpattern-concept-singleton.md %})

[//]: # ()
[//]: # (### `Lazy-Initialization` 사용)

[//]: # (앞선 방법들은 모두 클래스 로드 시점에 `Singleton` 의 인스턴스가 초기화되는 방식이다. )

[//]: # (이는 인스턴스가 단 하나만 존재한다는 점은 확실하게 보장하지만, )

[//]: # (초기 시스템 구동시에 아직 사용하지 인스턴스 생성을 위해 리소스를 낭비한다는 단점이 있다.  )

[//]: # ()
[//]: # (이런 초기 리소스를 절감하는 방법이 `Lazy-Initialization` 이다. )

[//]: # (`정적 팩토리 메소드` 방법을 기반으로 `getInstance&#40;&#41;` 메소드가 호출 될때 `null` 인 경우 인스턴스를 생성하는 것이다.  )

[//]: # ()
[//]: # (```java)

[//]: # (public class Singleton {)

[//]: # (    private static Singleton INSTANCE = null;)

[//]: # ()
[//]: # (    public static Singleton getInstance&#40;&#41; {)

[//]: # (        if&#40;INSTANCE == null&#41; {)

[//]: # (            INSTANCE = new Singleton&#40;&#41;;)

[//]: # (        })

[//]: # ()
[//]: # (        return INSTANCE;)

[//]: # (    })

[//]: # (})

[//]: # (```  )

[//]: # ()
[//]: # (```java)

[//]: # (public class SingletonTest {)

[//]: # (    @Test)

[//]: # (    public void singleton_sameInstance&#40;&#41; {)

[//]: # (        Singleton singleton1 = Singleton.getInstance&#40;&#41;;)

[//]: # (        Singleton singleton2 = Singleton.getInstance&#40;&#41;;)

[//]: # ()
[//]: # (        assertThat&#40;singleton1, sameInstance&#40;singleton2&#41;&#41;;)

[//]: # (    })

[//]: # ()
[//]: # (    @Test)

[//]: # (    public void reflection_notSameInstance&#40;&#41; throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {)

[//]: # (        Singleton singleton1 = Singleton.getInstance&#40;&#41;;)

[//]: # (        Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor&#40;&#41;;)

[//]: # (        constructor.setAccessible&#40;true&#41;;)

[//]: # (        Singleton singleton2 = constructor.newInstance&#40;&#41;;)

[//]: # ()
[//]: # (        assertThat&#40;singleton1, not&#40;sameInstance&#40;singleton2&#41;&#41;&#41;;)

[//]: # (    })

[//]: # ()
[//]: # (    @Test)

[//]: # (    public void serialize&#40;&#41; {)

[//]: # (        Singleton singleton1 = Singleton.getInstance&#40;&#41;;)

[//]: # (        Singleton singleton2 = Singleton.getInstance&#40;&#41;;)

[//]: # ()
[//]: # (        assertThat&#40;singleton1, sameInstance&#40;singleton2&#41;&#41;;)

[//]: # ()
[//]: # (        singleton2 = &#40;Singleton&#41; Util.deserialize&#40;Util.serialize&#40;singleton2&#41;&#41;;)

[//]: # ()
[//]: # (        assertThat&#40;singleton1, not&#40;sameInstance&#40;singleton2&#41;&#41;&#41;;)

[//]: # (    })

[//]: # (})

[//]: # (```  )

[//]: # ()
[//]: # (`reflection`, `직렬화/역직렬화` 이슈 해결을 위해서는 해결을 위한 코드가 추가로 필요하다는 점은 동일하다. )

[//]: # (다만 `getInstance&#40;&#41;` 가 호출 되는 인스턴스가 정말로 필요한 시점에 생성이 이뤄진다는 점으로 초기 리소스 절감에 도움이 된다는 장점이 있다.  )

[//]: # ()
[//]: # ()
[//]: # (구분|보장 여부)

[//]: # (---|---)

[//]: # (`reflection` 사용시 단일 인스턴스 보장|X)

[//]: # (`직렬화/역직렬화` 수행시 단일 인스턴스 보장|X)

[//]: # (불필요한 초기 리소스 최소화|O)

[//]: # (`Multi-Thread` 환경에서 단일 인스턴스 보장|X)

[//]: # (`Multi-Thread` 환경에서 단일 인스턴스 보장을 위한 성능 저하 최소화|X)

[//]: # ()
[//]: # ()
[//]: # (### `Lazy` + `synchronized` 키워드 사용)

[//]: # (`Lazy-Initialization` 방법의 가장 큰 단점은 `Multi-Thread` 환경에서 `getInstance&#40;&#41;` 메소드를 동시에 호출 한다면 단일 인스턴스 임을 보장 할 수 없다는 점이다. )

[//]: # (위 문제를 가장 간단하게 해결 하는 방법은 `getInstance&#40;&#41;` 메소드에 `synchronized` 키워드를 사용하는 것이다.  )

[//]: # ()
[//]: # (```java)

[//]: # (public class Singleton {)

[//]: # (    private static Singleton INSTANCE = null;)

[//]: # ()
[//]: # (    public static synchronized Singleton getInstance&#40;&#41; {)

[//]: # (        if&#40;INSTANCE == null&#41; {)

[//]: # (            INSTANCE = new Singleton&#40;&#41;;)

[//]: # (        })

[//]: # ()
[//]: # (        return INSTANCE;)

[//]: # (    })

[//]: # (})

[//]: # (```  )



---
## Reference
[Effective Java](https://book.naver.com/bookdb/book_detail.nhn?bid=14097515)  


