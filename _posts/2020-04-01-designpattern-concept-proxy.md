--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Proxy Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '투과성의 성질을 이용해서 실제 주체를 대신해서 역할을 수행하고 필요시에 역할을 위임하는 Proxy 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Proxy
use_math : true
---  

## Proxy 패턴이란
- `Proxy` 는 대리인이라는 의미를 가진 것과 같이, 특정 주체를 대신해서 대리인이 가능함 범위의 역할을 수행하는 패턴을 의미한다.

### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_proxy_1.png)

- `Subject` : 주제의 역할로 `Proxy` 와 `RealSubject` 의 공통적인 동작이 정의된 인터페이스 혹은 추상클래스이다. 사용자는 `Subject` 를 통해 `Proxy` 와 `RealSubject` 의 구분없이 사용할 수 있다.
- `Proxy` : 대리인의 역할로 `Subject` 의 구현체이면서 주어진 역할에 대해서 `RealSubject` 대신 수행하고, 그외의 역할은 `RealSubject` 에게 맡긴다.
- `RealSubject` : 실주체의 역할로 `Subject` 의 구현체이면서, `Proxy` 에서 수행하지 못하는 부분을 직접 수행한다.


### Proxy 를 이용한 속도 향상
- `Proxy` 를 사용하면 무거운 인스턴스 생성에 대해서, 실제로 필요한 시점에 수행해 전체적인 수행속도를 향상 시킬수 있다.

### 대리인과 주체의 약할
- `Proxy` 패턴은 실제 주체와 대리인을 구분해 구성하는데, 구분하는 주된 목적은 역할 분담을 통해 부품화를 수행하기 위함이다.
- 실제 주체는 말그대로 어떤 하나의 객체를 표현하고 구현하는데 목적이 있다.
- 대리인은 실제 주체 대신 역할을 수행하는데 목적이 있다.
- 목적이 다른 만큼 개발과정에서 다양한 세부 구현이 변경될 수 있기 때문에, 이를 분리해 개발을 진행 할 경우 보다 유연하게 확장및 변경이 가능하다.

### 대리인과 주체의 관계
- `Proxy` 패턴에서 대리인과 실제 주체는 위임의 관계를 가지고 있다.
- 대리인은 실제 주체의 역할을 수행하기 위해서는 실제 주체 인스턴스에서 적절한 메소드 호출을 통해 역할을 위임한다.

### Proxy 패턴의 투과성
- `Proxy` 패턴에서 사용자와 실제 주체의 사이에 대리인이 위치하고 있지만, 불편함 없이 사용하고 있는 이러한 상태를 투과적이라고 한다.
- 사용자는 실제 주체를 직접 사용하지 않고 대리인을 통해 사용하지만, 특정 제약없이 대리인을 통해 실제 주체의 동작을 모두 사용 할 수 있다.

### Proxy 패턴의 종류
- `Virtual Proxy` : 실제 주체의 인스턴스가 실제로 필요한 시점에 생성, 초기화를 수행한다.
- `Remote Proxy` : 실제 주체가 같은 호스트가 아닌 다른 호스트에 있더라도 네트워크를 통해, 마치 같은 호스트에 있는 것처럼(투과성) 메소드를 호출해 사요하는 것을 의미한다. (Java RMI)
- `Access Proxy` : 실제 주체의 기능에 대해 접근 제어를 수행하는 것을 의미한다.

### 기타
- `HTTP Proxy` 는 많은 역할을 가지고 있지만, 그 중 웹 페이지를 캐싱해 사용자에게 제공한다. 실제 사용자가 웹 페이지에 접근하는 것이 아니라 `HTTP Proxy` 가 대신 웹 페이지에 접근하고 그 결과를 사용자에게 전달해 준다.


## 데이터베이스 만들기
- 데이터베이스는 하나의 커다란 시스템으로 한번 구동하는데 오랜 시간이 걸릴 수 있다.
- 이를 `Proxy` 패턴 중 `Virtual Proxy` 를 사용해서 속도 향상을 가져올 수 있도록 구성해 본다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_proxy_2.png)

### DB

```java
public interface DB {
    void setName(String name);
    String getName();
    void set(String key, String value);
    String get(String key);
}
```  

- `DB` 는 데이터베이스가 가져야 하는 동작을 나타내는 인터페이스이다.
- `Proxy` 패턴에서 `Subject` 역할을 수행한다.
- `DB` 를 통해 주체가 되는 객체와 `Proxy` 역할을 하는 객체를 동일 시해서 사용자에게 제공할 수 있다.
- `setName()`, `getName()` 은 데이터베이스의 이름을 설정하고 가져오는 메소드이다.
- `set()`, `get()` 메소드는 데이터베이스 저장소에서 데이터를 추가, 갱신 하거나 키를 기준으로 데이터를 가져오는 메소드이다.

### KeyValueDB

```java
public class KeyValueDB implements DB {
    private String name;
    private Map<String, String> storage;

    public KeyValueDB(String name) {
        this.name = name;
        this.init();
    }

    public void init() {
        try {
            Thread.sleep(3000);
            this.storage = new HashMap<>();
        } catch(Exception e){
            e.printStackTrace();
        }
    }

    @Override
    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public void set(String key, String value) {
        this.storage.put(key, value);
    }

    @Override
    public String get(String key) {
        return this.storage.get(key);
    }
}
```  

- `KeyValueDB` 는 `DB` 의 구현체로 실제 데이터베이스인 주체를 나타내는 클래스이다.
- `Proxy` 패턴에서 `RealSubject` 역할을 수행한다.
- `name` 필드는 데이터베이스의 이름, `storage` 필드는 `key-value` 구조로 데이터를 저장할 수 있는 저장소 이다.
- `init()` 메소드는 데이터베이스를 생성하기 위해 초기화하는 메소드로, 여러작업과 오랜시간이 걸린다는 것을 나타내기 위해 `Thread.sleep(3000)` 을 통해 3초 시간동안 수행되도록 했다.
- `setName()`, `getName()` 은 `name` 필드에 데이터베이스의 이름을 설정 및 가져오는 메소드이다.
- `set()`, `get()` 메소드는 `storage` 필드에 `key-value` 형식으로 데이터를 추가, 갱신하거나 가져오는 메소드이다.

### KeyValueDBProxy

```java
public class KeyValueDBProxy implements DB {
    private String name;
    private KeyValueDB instance;

    public KeyValueDBProxy(String name) {
        this.name = name;
    }

    private synchronized void realize() {
        if(this.instance == null) {
            this.instance = new KeyValueDB(this.name);
        }
    }

    @Override
    public synchronized void setName(String name) {
        if(this.instance != null) {
            this.instance.setName(name);
        }
        this.name = name;
    }

    @Override
    public String getName() {
        return this.name;
    }

    @Override
    public void set(String key, String value) {
        this.realize();
        this.instance.set(key, value);
    }

    @Override
    public String get(String key) {
        this.realize();
        return this.instance.get(key);
    }
}
```  

- `KeyValueDBProxy` 는 `DB` 의 구현체 이면서 `KeyValueDB` 의 대리인을 역할을 수행하는 클래스이다.
- `Proxy` 패턴에서 `Proxy` 역할을 수행한다.
- `name` 은 데이터베이스의 이름, `real` 은 실제 주체의 인스턴스를 나타내는 필드이다.
- `realize()` 를 통해 `Proxy` 에서 대리인 역할을 수행하는 실제 주체의 인스턴스를 생성한다. 
- `setName()`, `getName()` 은 `name` 필드에 데이베이스의 이름을 설정하고 가져오면서, `setName()` 의 경우 실제 주체인 `real` 인스턴스가 생성된 경우 주체의 메소드도 함께 호출해준다.
- `realize()`, `setName()` 은 다수의 사용자에 대해서 초기화 될 수 있기 때문에, `synchronized` 키워드를 사용해서 메소드 동작에 대한 동기화를 보장하도록 했다.
- `set()`, `get()` 는 대리인에서는 수행할 수 없는 역할이기 때문에, `realize()` 를 호출 하고 주체인 `KeyValueDB` 에게 동작에 대한 역할을 위임한다.

### Proxy 의 처리과정
- `KeyValueDBProxy` 가 주체 대신 수행할 수 있는 역할을 `setName()`, `getName()` 이다.
- `KeyValueDBProxy` 가 주체 대신 수행할 수 없어 실제 주체에게 위임하는 역할을 `set()`, `get()` 이 있다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_proxy_3.png)

### 테스트

```java
public class ProxyTest {
    @Test(timeout = 3100)
    public void SingleUser_KeyValueDBProxy_DoingOnce_During3seconds() {
        // given
        DB db = new KeyValueDBProxy("db");

        // when
        db.set("key1", "value1");

        // then
        assertThat(db.get("key1"), is("value1"));
    }

    @Test(timeout = 3100)
    public void SingleUser_KeyValueDBProxy_DoingMultiple_During3seconds() {
        // given
        DB db = new KeyValueDBProxy("db");

        // when
        db.set("key1", "value1");
        db.set("key2", "value2");
        db.set("key3", "value3");
        db.set("key4", "value4");
        db.set("key5", "value5");

        // then
        assertThat(db.get("key1"), is("value1"));
        assertThat(db.get("key2"), is("value2"));
        assertThat(db.get("key3"), is("value3"));
        assertThat(db.get("key4"), is("value4"));
        assertThat(db.get("key5"), is("value5"));
    }

    @Test(timeout = 3100)
    public void MultipleUser_KeyValueDBProxy_DoingEachOnce_During3seconds() throws Exception {
        // given
        DB db = new KeyValueDBProxy("db");
        int userCount = 10;
        List<Thread> userThreadList = new ArrayList<>(userCount);
        for(int i = 0; i < userCount; i++) {
            userThreadList.add(new Thread(new Runnable() {
                private int index;

                public Runnable init(int index) {
                    this.index = index;
                    return this;
                }

                @Override
                public void run() {
                    db.set("key" + index, "value" + index);
                }
            }.init(i)));
        }

        // when
        for(int i = 0; i < userCount; i++) {
            userThreadList.get(i).start();
        }
        for(int i = 0; i < userCount; i++) {
            userThreadList.get(i).join();
        }

        // then
        for(int i = 0; i < userCount; i++) {
            assertThat(db.get("key" + i), is("value" + i));
        }
    }

    @Test(timeout = 10)
    public void SingleUser_KeyValueDBProxy_GetSetName() {
        // given
        DB db = new KeyValueDBProxy("db");

        // when
        db.setName("db2");

        // then
        assertThat(db.getName(), is("db2"));
    }
}
```  

---
## Reference

	