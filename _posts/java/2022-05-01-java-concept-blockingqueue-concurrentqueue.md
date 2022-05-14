--- 
layout: single
classes: wide
title: "[Java 개념] ConcurrentLinkedQueue, LinkedBlockingQueue"
header:
  overlay_image: /img/java-bg.jpg 
excerpt: 'Java 에서 동시성 상황에서 사용할 수 있는 대기열 큐인 ConcurrentLinkedQueue 와 LinkedBlockingQueue 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - ConcurrentLinkedQueue
  - LinkedBlockingQueue
toc: true 
use_math: true
---  

## LinkedBlockingQueue vs ConcurrentLinedQueue
`LinkedBlockingQueue` 와 `ConcurrentLinkedQueue` 는
`Java` 에서 동시성이 존재하는 상황에서 사용할 수 있는 대표적인 대기열이다. 
두가지 모두 동시성에서 `Thread-Safe` 를 보장한다는 점, 연결 리스트라는 점은 동일하지만 각자
고유한 특성을 가지고 있다. 
이번 포스트에서는 위 두가지의 특징과 차이점에 대해서 알아보고자 한다.  

### LinkedBlockingQueue
[LinkedBlockingQueue](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/LinkedBlockingDeque.html)
([BlockingQueue](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/BlockingQueue.html)) 
은 이름에서도 알수 있듯이 `Blocking` 방식으로 대기열을 조작할 수 있는 `Queue` 구현체이다. 
가장 큰 특징 중 하나는 최대 크기를 제한 할 수 있다는 점이다. 
대기열 크기를 제한 할 수 있기 때문에 순간 증가해 시스템 전체에 영향을 줄 수 있는 부분을 제한 할 수 있다.  

`BlockingQueue` 의 구현체들은 아래와 같은 특징을 가진 연산자를 사용 할 수 있다.  

-|예외 발생|특정 결과 리턴|Block|Time out
---|---|---|---|---
Insert|add(e)|offer(e)|put(e)|offer(e, time, unit)
Remove|remove()|poll()|take()|poll(time, unit)
Examine|element()|peek()|-|-

각 연산자에 대한 테스트 는 아래와 같다.  

```java
public class LinkedBlockingQueueTest {
	@Test
	public void add_max_size_throwException() {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(3);
		blockingQueue.add(1);
		blockingQueue.add(2);
		blockingQueue.add(3);

		assertThrows(IllegalStateException.class, () -> blockingQueue.add(4), "Queue ful");
	}

	@Test
	public void remove_empty_throwException() {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>();

		assertThrows(NoSuchElementException.class, () -> blockingQueue.remove());
	}

	@Test
	public void element_empty_throwException() {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>();

		assertThrows(NoSuchElementException.class, () -> blockingQueue.element());
	}

	@Test
	public void offer_max_size_ignore() {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(3);
		blockingQueue.add(1);
		blockingQueue.add(2);
		blockingQueue.add(3);

		assertThat(blockingQueue.offer(4), is(false));
		assertThat(blockingQueue, hasItems(1, 2, 3));
	}

	@Test
	public void poll_empty_returnNull() {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>();

		assertThat(blockingQueue.poll(), nullValue());
		assertThat(blockingQueue, empty());
	}

	@Test
	public void peek_empty_returnNull() {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>();

		assertThat(blockingQueue.peek(), nullValue());
		assertThat(blockingQueue, empty());
	}

	@Test
	public void offer_max_size_waiting() throws InterruptedException {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(3);
		blockingQueue.add(1);
		blockingQueue.add(2);
		blockingQueue.add(3);

		assertThat(blockingQueue.offer(4, 2, TimeUnit.SECONDS), is(false));
		assertThat(blockingQueue, hasItems(1, 2, 3));
	}

	@Test
	public void offer_max_size_waiting_and_insert() throws InterruptedException {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(3);
		blockingQueue.add(1);
		blockingQueue.add(2);
		blockingQueue.add(3);
		Thread thread = new Thread(() -> {
			try {
				Thread.sleep(1500);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			blockingQueue.poll();
		});
		thread.start();

		assertThat(blockingQueue.offer(4, 2, TimeUnit.SECONDS), is(true));
		assertThat(blockingQueue, hasItems(2, 2, 4));
	}

	@Test
	public void poll_wait_returnNull() throws InterruptedException {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>();

		assertThat(blockingQueue.poll(2, TimeUnit.SECONDS), nullValue());
		assertThat(blockingQueue, empty());
	}

	@Test
	public void poll_wait_return_and_remove() throws InterruptedException {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>();
		Thread thread = new Thread(() -> {
			try {
				Thread.sleep(1500);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			blockingQueue.offer(1);
		});
		thread.start();

		assertThat(blockingQueue.poll(2, TimeUnit.SECONDS), is(1));
		assertThat(blockingQueue, empty());
	}

	@Test
	@Timeout(5)
	public void put_max_size_wait_forever() throws InterruptedException {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(3);
		blockingQueue.put(1);
		blockingQueue.put(2);
		blockingQueue.put(3);
		blockingQueue.put(4);
	}

	@Test
	@Timeout(5)
	public void take_empty_wait_forever() throws InterruptedException {
		BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(3);
		blockingQueue.take();
	}
}
```  

`BlockingQueue` 는 `Blocking` 방식을 사용하기 때문에
`put`(대기열에 공간이 있을 떄까지), `take`(대기열에 값이 존재할 떄까지) 처럼 연산자가 가능할 떄까지 대기하는 
동작이 가능하다.  

위와 같은 특징으로 `BlockingQueue` 는 `BlockingQueue` 연산을 수행하는 전용 스레드를 사용하는 경우에 적합해 보인다. 
`put`, `take` 를 사용하게 되면 해당 스레드에서는 지속적으로 `Blocking` 이 발생하기 때문에 
만약 해당 스레드가 다른 역할로도 사용된다면 전체적인 성능 저하를 발생 시킬 수 있다.  

`BlockingQueue` 는 `two-lock-queue` 알고리즘을 사용한다. 
이는 두가지 서로 다른 `putLock` 과 `takeLock` 이 존재하고, `put/offer` 은 `putLock` 을 `take/poll` 은 `takeLock` 을 사용한다.  


그리고 이러한 `Blocking` 기능으로 인해 `BlockingQueue` 를 `생산자-소비자` 관점으로 봤을 때, 
`생산자-소비자` 간의 경합이 존재한다. (`Insert-Remove` 동작간 경합 발생)
즉 이는 다수의 `생산자-소비자` 가 존재할 때는 더 큰 성능 저하를 불어 일으킬 수 있다.  

> 생산자끼리, 소비자끼리의 경합도 존재한다. 

`BlockingQueue` 는 `단일 생산자-다수 소비자` 보다는 `다수 생산자-단일 소비자` 인 경우 효율이 좋다.  


### ConcurrentLinkedQueue
[ConcurrentLinkedQueue](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/ConcurrentLinkedQueue.html)
은 `BlockingQueue` 와는 달리 `Non-Blocking` 방식으로 대기열을 조작할 수 있는 `Queue` 구현체이다. 
`ConcurrentLinkedQueue` 는 `Non-Blocking lock-free` 의 방식을 사용하고, 
[Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms](https://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf)
알고리즘을 기반으로 구현되었다고 한다. 
그리고 `BlockingQueue` 와 같이 대기열의 전체 크기 제한을 제공하지 않고, 
전체 대기열 크기에 대한 연산을 일반적인 `Queue` 구현체 처럼 상수 시간에 제공하지 못하고 `O(n)` 이 소요된다.  

`ConcurrentLinkedQueue` 는 `BlockingQueue` 에 비해서는 단순한 연산자만 제공한다. 

```java
public class ConcurrentLinkedQueueTest {
    @Test
    public void offer_add() {
        ConcurrentLinkedQueue<Integer> concurrentLinkedQueue = new ConcurrentLinkedQueue<>();

        assertThat(concurrentLinkedQueue.offer(1), is(true));
        assertThat(concurrentLinkedQueue.add(2), is(true));
    }

    @Test
    public void poll_empty_returnNull() {
        ConcurrentLinkedQueue<Integer> concurrentLinkedQueue = new ConcurrentLinkedQueue<>();

        assertThat(concurrentLinkedQueue.poll(), nullValue());
    }

    @Test
    public void remove_empty_throwException() {
        ConcurrentLinkedQueue<Integer> concurrentLinkedQueue = new ConcurrentLinkedQueue<>();

        assertThrows(NoSuchElementException.class, () -> concurrentLinkedQueue.remove());
    }
}
```  

`add`, `offer` 의 경우 대기열의 크기 제한은 없지만 애플리케이션 메모리에 더 이상 공간이 없을 경우 `java.lang.OutOfMemoryError ` 가 발생하게 된다.  

`ConcurrentLinkedQueue` 는 `Blocking` 방식이 아니므로 대기열 연산을 수행할 때 `Lock` 을 사용하지 않는 `lock-free` 방식으로 사용한다. 
그러므로 `BlockingQueue` 와 같이 `put`, `take` 처럼 대기열의 상태에 따라 연산이 수행되지 않고 대기하는 경우가 없다. 
만약 대기열 크기 제한이 없기 때문에 `offer` 은 항상 바로 수행되고, 대기열이 비어 있는 경우에는 `null` 값을 즉시 리턴한다.  

위와 같은 특징으로 `BlockingQueue` 와 비교한다면 동시성이 높은 상황에서 더 큰 효율을 보여 줄 수 있다. 
`생산자-소비자` 측면에서는 `생산자-소비자` 간의 경합은 발생하지 않는다. (`Insert-Remove` 동작간 경합 발생 X)
하지만 생산자들간의 경합은 발생하기 때문에 `다수 생산자-단일 소비자` 보다는 `단일 생산자-다수 소비자` 가 더울 효율이 좋다.  

또한 `BlockingQueue` 와 비교한다면 대기열 전용 스레드가 아니더라도 `ConcurrentLinkedQueue` 를 사용한다면 
큰 지연 없이 대기열 작업을 다른 작업과 동일한 스레드에서 수행 할 수 있다.  

### 비교

특징|LinkedBlockingQueue|ConcurrentLinkedQueue
---|---|---
Blocking|`BlockingQueue` 인터페이스 구현|`Non-Blcoking` 이므로 `BlockingQueue` 구현 X
Queue Size|대기열 최대 크기 지정 가능|무제한 대기열 최대 크기 지정 불가
Locking|사용|사용 X
Algorithm|`two-lock-queue` 알고리즘 사용|Michael & Scott algorithm for non-blocking, lock-free queues 사용
Implementation|`two-lock-queue` 알고리즘 사용으로 `putLock`, `takeLock` 사용|`CAS(Compare-And-Swap)` 사용
Blocking Behavior|대기열이 비었을 때나, 대기열이 꽉찼을 때 발생|대기열이 비었을 때는 `null` 리턴하고 다른 `Blocking` 발생 X

### 성능 테스트
성능 테스트에서는 동시성 수준에 따른 `Insert`, `Remove` 성능을 확인 해보고자 한다. 

실제 성능 테스트전에 `BlockingQueue` 또는 `ConcurrentLinkedQueue` 를 사용이 필요한 경우를 가정해 본다. 
만약 단일 스레드를 사용하는 경우(소비자만 존재, 생산자만 존재, 생산자-소비자가 모두 단일 스레드에서 동작)는 
위 2개의 `Queue` 구현체를 사용할 필요 없이 일반적인 `Queue` 를 사용하는 것이 더 합리적일 것이다. 
물론 성능도 단일 스레드에서 수행되는 `Insert`, `Remove` 가 더욱 빠를 것이다. (공유 자원에 따른 경합이 존재하지 않기 때문)  

위와 같더라도 동시성을 늘리면서까지 `BlockingQueue`, `ConcurrentLinkedQueue` 를 사용하는 이유는 
대기열의 데이터를 기반으로 다른 동작들에 대한 소요시간이 더 크게 작용하기 때문일 것이다. (`poll` -> `다른 동작(DB 연산, 외부 요청 등)`)

즉 해당 성능 테스트에서는 동시성에 늘어남에 따라 성능 저하가 발생하는 부분은 자명하다는 전재를 두고, 
얼마나 혹은 어떠한 상황에서 더 큰 지연이 발생하는지에 대해서 알아보는게 목적이다. 
그러므로 절대적인 수치보다는 동시성에 따른 상대적인 수치로 봐야 한다.  

테스트의 결과는 동일하게 주어진 데이터 개수를 모두 연산하기까지 소요된 시간으로 값이 낮을 수록 성능이 좋다고 볼 수 있고, 
50번의 반복 테스트 결과의 평균치 이다.  

#### poll(소비자) 테스트
아래는 `poll` 연산을 동시성을 늘리가며 테스트한 결과이다. 

스레드 수|LinkedBlockingQueue|ConcurrentLinkedQueue
---|---|---
1|199ms|115ms
2|463ms|212ms
4|375ms|293ms
8|369ms|427ms
16|337ms|413ms
32|350ms|397ms
64|349ms|417ms
128|357ms|505ms

`poll` 연산의 경우 `ConcurrentLinkedQueue` 가 동시성이 높에 짐에 따라 비교적 더 큰 성능 저하가 발생하는 것을 확인 할 수 있다.    



#### offer(생산자) 테스트
아래는 `offer` 연산을 동시성을 늘리가며 테스트한 결과이다.

스레드 수|LinkedBlockingQueue|ConcurrentLinkedQueue
---|---|---
1|2356ms|2353ms
2|2530ms|2505ms
4|2693ms|2613ms
8|2751ms|2651ms
16|2843ms|2724ms
32|3061ms|2842ms
64|3277ms|2912ms
128|3510ms|2946ms

`offer` 연산의 경우 전반적으로 동시성에 따른 성능이 `ConcurrentLinkedQueue` 가 더 좋은 것을 확인 할 수 있다.  


### offer(생산자)-poll(소비자) 테스트
아래는 `offer`, `poll` 가 동시에 독립된 스레드에서 수행 될때 동시 성을 늘려가며 테스트 한 결과이다. 

첫 번째로 `offer(생산자)` 가 1개일 때 `poll(소비자)` 가 다수 일때의 테스트 결과이다. 

offer 스레드 수|poll 스레드 수|LinkedBlockingQueue(ms)|ConcurrentLinkedQueue(ms)
---|---|---|---
1|1|2743|2571
1|2|2748|2592
1|4|3554|3545
1|8|10852|8881
1|16|611697|13095
1|32|1073715|26299
1|64|3402783|39705
1|128|4189562|74713

`단일 생산자-다수 소비자` 에서는 동시성이 늘어 날수록 `LinkedBlockingQueue` 가 큰 성능 저하를 보이는 것을 확인 할 수 있다. 

두 번째로 `offer(생산자)` 가 다수개 일때 `poll(소비자)` 가 1개 일때의 테스트 결과이다.

offer 스레드 수|poll 스레드 수|LinkedBlockingQueue(ms)|ConcurrentLinkedQueue(ms)
---|---|---|---
1|1|2743|2571
2|1|2760|2483
4|1|3038|2748
8|1|3148|2887
16|1|3271|2868
32|1|3530|2922
64|1|3562|3010
128|1|3519|3078


`다수 생산자-단일 소비자` 는 전반적으로 `단일 생산자-다수 소비자` 보다 더 욱 빠른 성능을 보여주지만, 
동시성이 늘어 날 수록 `LinkedBlockingQueue` 의 성능 저하도 크게 늘어나고 있음을 알 수 있다. 

마지막으로  `offer(생산자)` 가 다수개 일때 `poll(소비자)` 도 다수개 일때의 테스트 결과이다.

offer 스레드 수|poll 스레드 수|LinkedBlockingQueue(ms)|ConcurrentLinkedQueue(ms)
---|---|---|---
2|2|2665|2510
2|4|2876|3595
2|8|6891|8610
2|16|156223|10038
2|32|1202980|13286
2|64|3482263|22036
2|128|12414848|40981
4|2|2778|2877
4|4|2869|4075
4|8|3845|7316
4|16|46264|8523
4|32|729522|10886
4|64|1440221|16024
4|128|8922409|30971
8|2|2949|2984
8|4|3251|3849
8|8|3626|7252
8|16|8773|7995
8|32|45267|9323
8|64|349204|14029
8|128|3327985|21342
16|2|3173|3063
16|4|3359|3923
16|8|3642|7253
16|16|7796|7851
16|32|15249|8824
16|64|250712|11888
16|128|639010|22243
32|2|3376|3226
32|4|3349|4043
32|8|3342|7376
32|16|3844|7711
32|32|7025|8783
32|64|23617|12020
32|128|207434|19566
64|2|3478|3311
64|4|3531|4033
64|8|3673|7126
64|16|3653|7637
64|32|6055|8481
64|64|16296|10727
64|128|44641|18316
128|2|3468|3309
128|4|3468|4178
128|8|3635|6935
128|16|3894|7371
128|32|7112|8106
128|64|8996|9772
128|128|24787|15536

동시성에 낮을 때는 `LinkedBlockingQueue` 가 약간 더 우수한 경우도 있지만, 
동시성이 높은 경우만 본다면 거의 대부분 큰 차이로 `ConcurrentLinkedQueue` 가 우수한 것을 확인 할 수 있다.  

전반적인 결과를 미루어 본다면, `ConcurrentLinkedQueue` 의 경우 `생산자-소비자` 간의 경합은 우선 존재하지 않는다. 
그러므로 동시성이 늘어나더라도 성능이 `LinkedBlockingQueue` 처럼 큰 폭으로 저하되는 현상은 발견되지 않았다. 
하지만 `ConcurrentLinkedQueue` 또한 생산자들 끼리는 경합이 존재하지 때문에 다수의 생산자를 사용하는 경우에는 성능 저하에 대해서 고려가 필요하다.  

`LinkedBlockingQueue` 는 `생산자-소비자` 간의 경합이 존재하기 떄문에 전반적으로 동시성이 늘어 날때 마다 큰 폭의 성능 저하를 확인 할 수 있었다.  
하지만 `다수 생산자-단일 소비자` 의 경우에는 비교적 `ConcurrentLinkedQueue` 와 큰 차이 없는 성능을 보여줬기 때문에, 
해당 모델에서는 사용을 검토해도 좋을 것같다.  


---
## Reference
[LinkedBlockingQueue vs ConcurrentLinkedQueue](https://www.baeldung.com/java-queue-linkedblocking-concurrentlinked)  
[LinkedBlockingQueue vs ConcurrentLinkedQueue in Java](https://www.javacodestuffs.com/2020/07/linkedblockingqueue-vs.html)  
[LinkedBlockingQueue](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/LinkedBlockingQueue.html)  
[BlockingQueue](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/BlockingQueue.html)  
[ConcurrentLinkedQueue](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/ConcurrentLinkedQueue.html#size())  

