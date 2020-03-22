--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Flyweight Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '인스턴스의 공유하는 방식으로 메모리를 절약하는 Flyweight 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Flyweight
use_math : true
---  

## Flyweight 패턴이란
- `Flyweight` 는 권투에서 가장 체중이 가벼운 체급을 의미하는 것처럼, 객체를 가볍게 구성할 수 있도록 해주는 패턴이다.
- 여기서 객체를 가볍게라는 의미는 객체를 생성하기 위해 소비되는 메모리의 사용용량을 의미한다.
- `Flyweight` 는 같은 종류의 객체가 많은 곳에서 사용될 때 유용하게 사용될 수 있다.
- Java 에서는 `new` 를 통해 객체의 인스턴스를 생성하게 되면 필요한 만큼 메모리가 할당된다.
- 객체를 가볍게 만들어 메모리 사용량을 줄이는 방법은 객체의 인스턴스를 가능한 공유시켜 불필요한 `new` 연산을 최소화 시키는 것이다.
 
### 패턴의 구성요소

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_flyweight_1.png)

- `Flyweight` : 메모리 절약을 위해 인스턴스를 공유하고 싶은 객체를 나태내는 클래스이다.
- `FlyweightFactory` : `Flyweight` 의 실제 인스턴스를 생성하고, 이미 생성된 인스턴스의 공유를 관리하는 클래스이다.
- `Client` : `FlyweightFactory` 를 사용해서 `Flyweight` 인스턴스를 얻고, `Flyweight` 를 통해 동작을 수행하는 클래스이다.

### 인스턴스가 공유 될때 주의할 점
- `Flyweight` 패턴은 인스턴스의 공유를 통해 메모리의 사용량을 최적화한다.
- 인스턴스가 공유된다는 부분은 일반적으로 객체를 `new` 를 통해 인스턴스를 생성해 사용할 때와 비교해서 주의해야할 점이 있다.
- 공유되는 인스턴스를 변경하게 되면, 그 인스턴스를 사용하는 모든 부분에 영향이 가게 된다.
- 이러한 주의점으로 `Flyweight` 패턴을 설계할 때는 구현사항에 알맞게 `공유`가 되야하는 부분만 `Flyweight` 로 취급해야 한다.
	- 공유 시켜야할 부분과 공유 시키면 안되는 구분해야 한다.

### Intrinsic 과 Extrinsic
- `인스턴스가 공유 될때 주의할 점` 에서 설명했던 공유 시켜야 할 부분과 공유 시키지 말야아 하는 부분을 `intrinsic` 과 `extrinsic` 라고 한다.
- `intrinsic` 한 정보는 본질적인이라는 의미로, 인스턴스의 정보 중 변하지 않는 정보, 상태에 의존하지 않는 정보를 의미한다.
	- 장소나 상황에 의존하지 않기 때문에 공유할 수 있다.
- `extrinsic` 한 정보는 비본질적인이라는 의미로, 인스턴스의 정보 중 상황에 따라서 변하는 정보, 상태에 의존하는 정보를 의미한다.
	- 장소나 상황에 의존하기 때문에 공유할 수 없다.

### 공유 인스턴스와 Garbage Collection
- `Flyweight` 패턴에서 인스턴스를 공유시키기 위해 특정 저장소에 인스턴스를 모아 저장하게 된다.
- 이때 Java 의 메모리를 관리하는 `garbase collection` 의 대상에 공유되는 인스턴스는 포함되지 않는다는 점을 유의해야 한다.
- `garbase collection` 은 `new` 를 통해 인스턴스를 생성할 때 메모리를 확보하고, 메모리가 부족하게 되면 메모리 공간(Heap 영역)을 조사해 사용되지 않는 인스턴스의 메모리를 해제해서 메모리 공간을 확보한다.
- `grabase collection` 는 인스턴스가 외부에서 참고하고 있으면 사용중이라고 판별하고 그렇지 않다면 사용하고 있지 않다고 판별해서 메모리를 해제시킨다.
- 공유되는 인스턴스는 공유를 위해 외부의 참조가 있기 때문에 `garbase collection` 의 대상이 되지 않는다.

### 기타
- 인스턴스를 공유하는 `new` 의 동작 횟수를 줄일 수 있기 때문에, 메모리 뿐만 아니라 `new` 를 수행할때 소요되는 시간도 줄일 수 있다.
- 특정 상황에서는 인스턴스를 공유하는 것이 독이 될 수도 있다.
	- 불필요하게 많은 인스턴스를 공유하는 경우
	- 사용가능한 메모리의 대부분이 공유 인스턴스로 사용되는 경우

## 버스 탑승 인원 카운트
- 버스는 많은 사람들이 이용하는 대중교통이고, 여러 정류장에서 같은 버스가 다니기도 하고 다른 버스가 다니기도 한다.
	- 버스와 버스 정류장의 관계는 1:n 의 관계이다.
- 버스 정류장에서는 각 버스에 탑승하는 인원을 버스의 번호를 기준으로 카운트한다.
- 각 버스 정류장에서 같은 번호를 가진 버스의 카운트는 공유 된다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_flyweight_2.png)

### Bus

```java
public class Bus {
    private int number;
    private int count;

    public Bus(int number) {
        this.number = number;

        try {
            Thread.sleep(1000);
            // 버스를 만드는데 많은 시간과 비용이 들어간다고 가정한다.
        } catch(Exception e) {
            e.printStackTrace();
        }
    }

    public synchronized int increaseCount() {
        return ++this.count;
    }

    public int getNumber() {
        return number;
    }

    public int getCount() {
        return count;
    }
}
```  

- `Bus` 는 공유되는 객체인 버스를 나타내는 클래스이다.
- `Flyweight` 패턴에서 `Flyweight` 역할을 수행한다.
- 버스는 버스의 고유한 번호인 `number` 필드를 가진다.
- 버스에 탑승한 승객의 횟수를 `count` 필드를 통해 관리한다.
- `Bus` 를 생성하기 위해서는 많은 비용이 필요하다는 것을 표현하기 위해, 생성자에 `Thread.sleep()` 메소드로 인스턴스 생성에 오랜 시간이 걸리도록 했다.
- 탑승 승객의 카운트 값을 증가시키는 `increaseCount()` 메소드는 `Bus` 의 인스턴스가 `BusStation` 에 의해 공유되는 객체이기 때문에 `count` 값 증가는 `synchronized` 를 통해 동기화 처리를 했다.
	- 하나의 `Bus` 인스턴스는 여러개의 `BusStation` 에 의해 공유될 수 있다.


### BusFactory 

```java
public class BusFactory {
    private static BusFactory instance = new BusFactory();
    private final Map<Integer, Bus> pool;

    private BusFactory() {
        this.pool = new HashMap<>();
    }

    public static BusFactory getInstance() {
        return instance;
    }

    public synchronized Bus get(int number) {
        Bus bus = this.pool.get(number);

        if(bus == null) {
            bus = new Bus(number);
            this.pool.put(number, bus);
        }

        return bus;
    }

    public void clearPool() {
        this.pool.clear();
    }
}
```  

- `BusFactory` 는 공유되는 객체인 `Bus` 의 인스턴스를 생성하고 관리하는 싱글톤 클래스이다.
- `Flyweight` 패턴에서 `FlyweightFactory` 역할을 수행한다.
- `Map` 형태로 `pool` 필드에 생성된 `Bus` 인스턴스를 번호와 인스턴스 구조로 저장해 관리한다.
- `Bus` 의 인스턴스를 생성하는 `get()` 메소드 `BusStation` 에 의해 동시에 같은 `Bus` 의 인스턴스 생성을 요청할 수 있기 때문에 `synchronized` 를 통해 동기화 처리를 했다.
- `get()` 메소드를 통해 반환된 `Bus` 인스턴스는 공유 객체이다.
- `clearPool()` 는 원활한 테스트를 위해 강제로 `pool` 필드를 초기화하는 메소드이다.

### BusStation

```java
public class BusStation {
    private String name;
    private Map<Integer, Bus> busMap;

    public BusStation(String name, int[] busNumberArray) {
        this.name = name;
        this.busMap = new HashMap<>();
        this.addBus(busNumberArray);
    }

    private void addBus(int[] busNumberArray) {
        for(int number : busNumberArray) {
            this.busMap.put(number, BusFactory.getInstance().get(number));
        }
    }

    public Bus takeBus(int number) {
        Bus bus = this.busMap.get(number);

        if(bus != null) {
            bus.increaseCount();
        }

        return bus;
    }

    public List<Integer> busNumberList() {
        return new ArrayList<>(this.busMap.keySet());
    }
}
```  

- `BusStation` 은 버스 정류장을 나타내면서 `BusFactory` 를 통해 생성된 여러 `Bus` 인스턴스를 가지는 클래스이다.
- `Flyweight` 패턴에서 `Client` 역할을 수행한다.
- `name` 은 하나의 버스 정류장이 가지는 이름을 의미하는 필드이다.
- `busMap` 을 통해 버스 정류장에서 탑승할 수 있는 버스를 버스 번호, 인스턴스 구조로 저장해 관리한다.
- 생성자에서는 `busNumberArray` 인 정류장에서 탑승 가능한 버스 번호를 배열로 전달해 `addBus()` 메소드를 사용해 `Bus` 인스턴스를 설정한다.
- `addBus()` 에서 배열로 받은 버스 번호 배열을 바탕으로 `BusFactory` 의 `get()` 메소드를 사용해서 실제로 `busMap` 에 `Bus` 인스턴스를 추가한다.
	- `BusFactory` 를 통해 설정된 `BusStation` 에 저장한 `Bus` 인스턴스 들은 모두 공유 객체이다.
- `takeBus()` 는 버스 번호를 인자값으로 받아 `Bus` 의 `increaseCount()` 메소드를 통해 버스의 탑승 처리를 수행한다.
- `busNumberList()` 는 버스 정류장에서 탑승할 수 있는 버스의 번호를 `List` 형태로 반환한다.

### Flyweight 의 공유 인스턴스
- 아래와 같은 구조로 4개의 `BusStation` 과 3개의 `Bus` 가 있을 때 인턴스의 모습은 다음과 같다.
	- `BusStationA` : `Bus1`, `Bus2`
	- `BusStationB` : `Bus2`, `Bus3`
	- `BusStationC` : `Bus1`, `Bus2`, `Bus3`
	- `BusStationD` : `Bus1`, `Bus3`

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_flyweight_3.png)

### 테스트

```java
public class FlyweightTest {
    @Before
    public void setUp() {
        BusFactory.getInstance().clearPool();
    }

    @Test
    public void BusStation_TakeBus_SingleThread() {
        // given
        BusStation aStation = new BusStation("a", new int[]{1, 2});
        BusStation bStation = new BusStation("b", new int[]{2, 3});

        // then
        aStation.takeBus(1);
        aStation.takeBus(2);
        bStation.takeBus(2);
        bStation.takeBus(3);

        // when
        Bus bus1 = BusFactory.getInstance().get(1);
        assertThat(bus1.getCount(), is(1));
        Bus bus2 = BusFactory.getInstance().get(2);
        assertThat(bus2.getCount(), is(2));
        Bus bus3 = BusFactory.getInstance().get(3);
        assertThat(bus3.getCount(), is(1));
    }

    @Test
    public void BusStation_TakeBus_MultiThread() {
        // given
        BusStation[] stationArray = new BusStation[]{
                new BusStation("a", new int[]{1, 2}),
                new BusStation("b", new int[]{2, 3}),
                new BusStation("c", new int[]{1, 2, 3}),
                new BusStation("d", new int[]{1, 3})
        };
        // Thread 의 수는 설정된 정류장의 수로 설정한다.
        int threadCount = stationArray.length;
        // 정류장 하나당 반복하는 횟수
        int loopCount = 1000;
        List<Thread> threadList = new ArrayList<>(threadCount);
        for (int i = 0; i < threadCount; i++) {
            threadList.add(new Thread(new Runnable() {
                // Thread 에서 사용하는 정류장의 인스턴스
                private BusStation busStation;

                // 정보 초기화
                public Runnable init(BusStation busStation) {
                    this.busStation = busStation;
                    return this;
                }

                @Override
                public void run() {
                    // 정류장에서 사용할 수 있는 Bus 의 번호를 가져옴
                    List<Integer> busNumberList = this.busStation.busNumberList();
                    // loopCount 수만큼 모든 버스 한번씩 탑승
                    for (int i = 0; i < loopCount; i++) {
                        for (int number : busNumberList) {
                            this.busStation.takeBus(number);
                        }
                    }
                }
            }.init(stationArray[i])));
        }

        // then
        for (int i = 0; i < threadCount; i++) {
            // Thread 시작
            threadList.get(i).start();
        }
        for (int i = 0; i < threadCount; i++) {
            try {
                threadList.get(i).join();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        // then
        Bus bus1 = BusFactory.getInstance().get(1);
        assertThat(bus1.getCount(), is(3000));
        Bus bus2 = BusFactory.getInstance().get(2);
        assertThat(bus2.getCount(), is(3000));
        Bus bus3 = BusFactory.getInstance().get(3);
        assertThat(bus3.getCount(), is(3000));
    }
}
```


---
## Reference

	