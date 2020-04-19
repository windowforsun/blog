--- 
layout: single
classes: wide
title: "[Java 개념] JVM Garbage Collection"
header:
  overlay_image: /img/java-bg.jpg
excerpt: '자동으로 Heap 메모리에서 참조되지 않는 객체를 식별해 메모리를 해제해 주는 GC 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - JVM
    - Garbage Collection
toc: true
use_math: true
---  

## Garbage Collection 이란
- `GC` 라고 축약해서 부르기도 한다.
- `GC` 는 이미 할당된 메모리에서 더 이상 사용되지 않는 메모리를 해제하는 동작을 의미한다.
- 여기서 사용되지 않는 메모리는 `Stack Area` 에서 참조하지 않는 메모리 영역을 의미한다.
- Java 에서 메모리를 해제하기 위해서는 변수에 `null` 을 설정하거나, `System.gc()` 를 호출 할 수도 있다.
	- `System.gc()` 메소드를 호출하는 것은 매우 주의해야 하는데, 이는 구동 중인 시스템에 성능적으로 큰 영향을 끼칠 수 있기 때문이다.
- `GC` 작업은 `stop-the-world` 라는 동작을 수행하게 되는데, 이는 `GC` 를 실행하기 위해 `JVM` 이 `GC` 를 담당하는 쓰레드외 모든 쓰레드의 작업을 멈추는 것을 의미한다.(`STW`)
	- `GC` 작업 이후에 중지됐던 쓰레드는 다시 실행된다.
- 애플리케이션이 구동된다는 것은 호스트 머신에서 필요한 자원을 할당받아 사용한다는 의미이고, Java 는 다른 언어(C, C++) 과 달리 메모리 관리를 개발자가 직접하지 않고 `GC` 가 이를 담당해 주기 때문에 이러한 작업이 필요하다.
- `GC` 튜닝이란 피할 수 없는 `stop-the-world` 시간을 줄이는 것을 의미한다.
- `GC` 는 아래 2가지 전제조건을 통해 만들어 졌다.
	- 대부분의 객체는 금방 접근 불가능 상태(unreachable)가 된다.
	- 오래된 객체에서 젊은 객체로의 참조는 아주 적게 존재한다.
- 본 포스트에서는 Java 8 을 기준으로 `GC` 를 설명한다.

## Garbage Collection 의 과정
- `JVM` 의 `GC` 의 과정에 대한 기본적인 개념을 설명한다.

### Marking 

![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-1-0-1.PNG)

- 할당된 메모리 중 사용되지 않고 있는지 체크하는 단계이다.
- 모든 메모리 공간을 검사한다면 `GC` 수행마다 큰 부하가 발생할 수 있기 때문에, `JVM` 은 세대(`Generation`)로 나눠 `GC` 작업을 수행한다.(`Generation GC`)

### Normal Deletion

![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-1-0-2.PNG)

- `Marking` 단계에서 찾은 비참조 객체를 삭제하는 단계이다.
- 삭제 후 생긴 빈공간은 `Memory Allocator` 가 관리하고, 이후 새로운 메모리 할당시에 빈 공간을 찾아준다.

### Deletion with Compacting

![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-1-0-3.PNG)

- `Normal Deletion` 단계까지 수행되면 앞서 설명한 것처럼 메모리의 할당이 순차적이지 않게 되는데, 효율을 위해 `Compacting` 작업을 수행하는 단계이다.
- 사용중인 메모리를 한곳으로 모으고 압축해 이후 보다 쉽게 메모리 할당이 가능하도록 한다.

## Generation GC

![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-1-0-4.png)

- `JVM` 은 `GC` 의 효율성을 위해 메모리를 세대별로 나눠 `GC` 를 수행한다고 했다.
- 이는 대부분 객체의 수명이 짧다는 통계를 전제로 한다.


## Garbage Collection 의 구조
Hotspot JVM 의 Heap Memory 는 아래와 같이 크게 `Young Generation`, `Old Generation`, `Permanent Generation` 으로 구성된다.  

![그림 1]({{site.baseurl}}/img/java/concept_jvm_garbagecollection_1.png)

### Young Generation
- `Young Generation` 은 `Eden`, `Survivor0`, `Survivor1` 영역으로 나뉜다.
- 새롭게 생성된 객체가 위치하는 곳이다.
- 대부분의 객체가 금방 접근 불가능 상태가 되기 때문에 대부분의 객체는 `Young Generation` 영역에 생성되었다가 사라진다.
- `Young Generation` 영역에서 발생하는 `GC` 를 `Minor  GC` 라고 한다.


### Old Generation
- `Minor GC` 이후에도 생존한 객체는 `Old Generation` 으로 이동된다. 
- `Young Genration` 보다 큰 크기를 가지고 있고, `GC` 도 적게 발생한다.
- `Old Generation` 에서 발생하는 `GC` 를 `Major GC` 혹은 `Full GC` 라고 한다.

### Native Memory
- Java 8 이전 에는(Java 7까지) `Permanent Generation` 이라는 공간이 있고, 이또한 `GC` 영역이였지만 아래 이슈들로 인해 사라지게 되었다.
	- `Static Object` 의 사용에 따른 메모리 사용 증가
	- `Class`, `Method Meta Data` 의 증가
- Java 8 이전과 이후의 변화를 아래 표를 통해 정리한다.

. | Java 7 | Java 8
---|---|---
Class Meta Data | Permanent | Metaspace
Method Meta Data | Permanent | Metaspace
Static Object | Permanent | Heap
Constant Object(String) | Permanent | Heap

- `Natvie Memory` 공간은 OS 레벨에서 관리하는 영역이므로 `Metaspace` 영역의 크기에 대해 개발자가 크게 의식할 필요가 없어졌다.


## Garbage Collection 의 동작

![그림 1]({{site.baseurl}}/img/java/concept_jvm_garbagecollection_2.png)

![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-1-0.PNG)

### Young Generation
- 처음 객체가 생성되면 `Eden` 영역에 생성되고 `Eden` 영역이 가득 차게 되면, `Minor GC` 가 발생한다.
		
	![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-1.PNG)
	
- `Minor GC` 에서 살아남은 객체는 `Survivor` 영역중 한곳으로 이동하면서, `Survivor` 영역에는 `Eden` 에서 살아남은 객체가 쌓인다.

	![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-2.PNG)

- `Minor GC` 과정을 반복할 때마다 살아남은 객체는 `Age` 값이 증가하게 된다.

	![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-3.PNG)
	
- `Survivor` 영역이 가득 차게되면 그중 살아남은 객체를 다른 `Survivor` 영역으로 이동하고, 가득 찬 `Survivor` 영역은 빈 상태가 된다.
	- `Minor GC` 가 발생 할때 마다, `S0`, `S1` 영역을 번갈아 가면서 사용한다.
	- JVM 옵션을 통해 `Eden` 영역과 `Survivor` 영역의 비율을 조정할 수 있다. (`-XX:SurvivorRate=n`)

	![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-4.PNG)
	
- 이러한 과정을 반복하다 계속해서 살아 남은 객체는 `Old Generation` 영역으로 이동하게 된다.
- 빠른 메모리 할당을 위해 `bump-the-pointer` 와 `TLABs`(Thread-Local Allocation Buffers) 라는 기술을 사용한다.
	- `bump-the-pinter` : `Eden` 영역에 할당된 마지막 객체를 추적한다. 
	마지막 객체는 영역 맨 위에 존재한다. 
	그리고 다음 생성되는 객체는 크기가 `Eden` 영역에 넣을 수 있는지 검사한다. 
	정당한 크기라면 영역에 추가하고 추가된 객체가 맨위에 위치하게 된다. 
	새로운 객체를 생성할 때 마지막에 추가된 객체만 검사하기 때문에 매우 빠르게 동작이 가능하다. 
	- `TLABs` : 멀티 스레드 환경에서 `Thread-Safe` 를 보장하기 위해 각 쓰레드는 `Eden` 영역의 작은 부분을 가지게 된다. 
	각 쓰레드에는 자신의 `TLABs` 영역에만 접근 할 수 있기 때문에, `bump-the-pointer` 기술을 사용할 때 `Lock` 을 수행하지 않고 메모리 할당이 가능하다.

### Old Generation
- `Minor GC` 가 반복되고, 살아남은 객체들의 `Age` 가 증가해서 임계값을 넘어서면 `Old Generation` 영역으로 이동된다.
	- `Age` 의 임계값은 JVM 옵션에서 설정가능하다. (`-XX:MaxTenuringThreashold=n`)
	
	![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-5.PNG)
	
	![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-6.PNG)
	
- 계속해서 `Old Generation` 영역에 쌓인 객체가 가득 차게되면 `Major GC` 가 발생해서 영역을 정리하게 된다.
- `Old Generation` 에서의 `GC` 는 `mark-sweep-compact` 라는 알고리즘을 사용한다.
	- `Mark` : 사용중인 객체와 그렇지 않은 객체를 식별한다.
	- `Sweep` : 사용중인 객체만 남기고 삭제한다.
	- `Compact` : 삭제 동작으로 발생한 빈공간을 채우기위해 살아남은 객체가 연속되게 구성한다.	

## GC 방식의 종류
- `Minor GC` : `Young Generation` 에서 발생하는 `GC` 를 의미하고, 
- `Major GC` : 
- `Full GC` :
- `Serial GC`
- `Parallel GC`
- `Parallel Old GC` (`Parallel Compacting GC`)
- `Concurrent Mark & Sweep GC` (`CMS`)
- `G1 GC` (`Garbage First GC`)
	
### Serial GC
- 싱글 스레드를 사용해서 `GC` 작업을 수행한다.
- 싱글 프로세서일 경우를 위한 `GC` 이므로, 멀티 프로세서 환경에서는 성능 저하가 발생한다.
- 싱글 스레드이므로 스레드와 스레드 사이의 통신 오버헤드가 발생하지 않는 다는 장점이 있다.
- `-XX:+UseSerialGC` 옵션을 통해 설정할 수 있다.

### Parallel GC
- `Serial GC` 와 기본적인 동작과정은 같다.
- 멀티 스레드를 사용해서 `GC` 작업을 수행한다는 점이 `Serial GC` 와의 가장 큰 차이점이다.
- 멀티 스레드를 사용하기 때문에 처리속도가 빠르고, 충분한 메모리와 코어의 수가 있을 떄 유리하다.
- `Parallel GC` 는 `Throughput GC` 라고도 불린다.
- `-XX:+UseParallelGC` 옵션을 통해 설정할 수 있다.
- `-XX:ParallelGCThreads=n` 을 통해 `GC` 처리에서 사용할 스레드 수를 설정할 수 있다.
- `Parallel GC` 는 `Young Generation` 에서는 멀티 스레드로 `GC` 가 동작하고, `Old Generation` 에서는 싱글 스레드로 `GC` 가 동작한다.
	
### Parallel Old GC
- `Parallel GC` 와 비교해서 `Old Generation` 의 `GC` 알고리즘이 `mark-summary-compaction` 의 알고리즘을 사용한다는 점이 다르다.
- `Summary` 란 `GC` 를 수행한 영역에 대해서 별도로 살아있는 객체를 식별하는 과정을 의미한다.
- `-XX:+UseParallelOldGC` 옵션을 통해 설정할 수 있다.
- `Parallel Old GC` 는 `Young Generation`, `Old Generation` 에서 모두 멀티 스레드로 `GC` 가 동작한다.
- `GC` 에서 `Compaction`(압축) 은 `Old Generation` 에서만 수행하고 이 과정또한 멀티 스레드로 동작한다.
	- `Young Generation` 의 `GC` 는 `Copy Collector` 의 역할로 압축이 필요없다.

### CMS GC
- `Old Generation` 영역을 대상으로 이루어지는 `GC` 이고, `Young Generation` 은 `Parallel GC` 로 동작한다.
- 애플리케이션 스레드와 병렬로 동작해서 `stop-the-world` 시간을 줄이는 방식이고, `Compaction` 도 수행하지 않는다.
- 실제로 `stop-the-world` 의 시간이 짧기 때문에 `Low Latency GC` 라고도 불린다.
- `Compaction` 을 수행하지 않기 때문에 메모리 단편화가 발생하면서 할당이 불가능해 질 수 있는데, 이때 `Compaction` 작업을 수행하게 되면서 더 긴 시간의 `stop-the-world` 가 발생할 수 있다.
- `Java 9` 부터는 `Deprecated` 되었다.
- `-XX:+UseConMarkSweepGC` 옵션을 통해 설정할 수 있다.
- `CMS GC` 의 동작과정은 아래와 같다.
	- `Initial Mark` : 클래스 로더에서 가장 가까운 객체 중 살아있는 객체만 찾기 때문에 `STW` 가 매우 짧다.
	- `Concurrent Mark` : 병렬로 동작하면서 `Initial Mark` 단계에서 선별된 객체부터 `Reference Tree` 를 순회 하며 참조하고 있는 객체를 확인해서 `GC` 대상을 선별한다.
	- `Remark` : `Concurrent Mark` 단계를 검증하기 위해, `GC` 대상으로 추가된 객체를 선별한다. `STW` 시간을 줄이기 위해서 멀티 스레드로 작업을 수행한다.
	- `Concurrent Sweep` : 선별된 `GC` 대상 객체를 메모리에서 제거한다.
	
### G1 GC

![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-7.PNG)
	
- `Java 7` 부터 추가된 서버형 `GC` 로 대량의 힙과 멀티 프로세서 환경에서 사용되도록 만들어진 `GC` 이다.
- `CMS GC` 의 대체하기 위해 만들어 졌다.
- 메모리를 바둑판과 같은 논리적 단위(`Region`)로 나눠 각 영역에 객체를 할당하고 `GC` 를 실행한다. 한 영역이 꽉 차면 다른 영역에 객체를 할당하고 `GC` 를 수행한다.
- `Region` 이라는 논리적인 단위를 통해 메모리를 관리하기 때문에 `Region` 단위의 `Compaction` 단계를 진행을 통해 메모리 단편화를 해결하고, 짧은 `STW` 와 빠른 `GC` 수행이 가능하다.
- `Initial Mark` 단계를 멀티 스레드로 수행하고, 어느 `Region` 에서 많은 메모리 해제가 일어나는지 파악해서 먼저 수집하기 때문에 `Garbage First GC` 라는 이름이 지어졌다.
- 살아남은 객체를 하나의 `Region` 으로 옮기는 과정에서 `Compaction` 을 멀티 스레드로 수행해서 시간을 단축한다.
- `-XX:+UseG1GC` 옵션을 통해 설정 할 수 있다.
- `G1 GC` 의 기본 구조와 `Young Generation` 의 `GC` 동작과정은 아래와 같다.
	1. 메모리 공간을 고정된 크기의 `Region` 단위로 나눈다.

		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-8.PNG)
		
		- `Region` 의 크기는 `JVM` 이 실행될 때 결정되는데, 1~32Mb 의 크기의 2000 개로 구성한다.
		
	1. 나눠진 `Region` 의 각각의 공간은 `Eden`, `Survivor`, `Old` 로 설정되어 사용되고 기존 `GC` 흐름과 같이 살아남은 객체들은 알맞은 영역으로 이동되면서 관리된다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-9.PNG)
		
		- `Region` 은 애플리케이션 스레드를 중지하지 않고 병렬로 수집되도록 설계 되었다.
	1. `Heap` 에 할당된 모습을 보면 아래 그림과 같이 초록색은 `Young Generation`, 파란색은 `Old Generation` 으로 연속되지 않은 상태인것을 확인 할 수 있다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-10.PNG)
	
		- `Regoin` 이 연속적이지 않게 할당되고 사용되는 것은 `GC` 이후 `Region` 의 크기를 재할당 할때 쉽게 계산하기 위함이다.
	1. `Minor GC`(`Young GC`) 가 발생한 `Young Generation Region`(`Eden`) 에 있던 객체들은 `Survivor Regieon` 으로 이동하거나, `Age` 가 `Threshold` 에 도달한 객체는 `Old Generation Region` 으로 이동하게 된다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-11.PNG)
		
		- 이 과정에서 `STW` 가 발생하게 되는데, `Eden` 과 `Survivor` 의 크기를 통해 다음 `GC` 이후 할당될 `Region` 의 크기와 `STW` 의 시간을 계산할 수 있다.
		- 멀티 스레드를 사용해서 `STW` 의 시간을 최소화 한다.
		
	1. `Minor GC` 발생 후 아래와 같이 `Young Generation Region` 에서 살아 남은 객체는 `Old Generation Region` 혹은 `Survivor Regoin` 으로 이동 및 `Compaction` 된것을 확인 할 수 있다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-12.PNG)
		
- `G1 GC` 에서 `Old Generation` 의 `GC` 동작과정은 아래와 같다.
	1. `Initial Mark` : `Young Generation GC` 에서 살아남은 객체를 `Mark` 단계와 같이 수행되며 `Survivor Region` 중 `Old Generation Region` 의 객체를 참조하고 있는 `Regoin` 을 식별하고 `STW` 가 발생한다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-13.PNG)
	
	1. `Root Region Scanning` : `Initial Mark` 단계에서 식별된 `Region` 중 `Old Generation Region` 의 객체를 참조하고 있는 객체를 스캔한다. 이 과정은 애플리케이션 스레드와 함께 수행된다.
		- `Young GC` 가 발생하기 전에 이 작업은 완료된다.
	1. `Concurrent Marking` : `Heap` 전체를 대상으로 살아 있는 객체를 찾는 과정으로 애플리케이션 스레드와 함께 동작하고, `Young GC` 를 통해 인터럽트 될 수 있다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-14.PNG)

	1. `Remark` : `Heap` 전체에서 살아있는 객체에 대한 과정의 마무리 단계로 `STW` 가 발생한다. 
	`Concurrent Marking` 단계에서 찾은 비어있는 `Regoin` 을 삭제하는 등의 전체적으로 `Regoin` 을 정리하는 작업을 수행한다. 
	`snapshot-at-the-beginning`(`SATB`) 라는 알고리즘을 사용하는데, 이는 `CMS` 에서 사용하는 콜렉터보다 훨씬 빠르다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-15.PNG)
	
	1. `Copying/Cleanup` : 살아있는 객체를 새로운 `Regoin` 으로 복사하게(`Reclaim`) 되는데, `G1 GC` 는 가장 생존력이 낮은 `Regoin` 으로 복사를 수행한다. 
	해당 `Regoin` 을 선택함으로써 이후 빠른 수집이 가능하기 때문이다. 
	선택된 `Regoin` 에 대해서 이후 `Young GC` 와 동시에 수집을 수행한다. 
	이는 `Young Generation` 과 `Old Generation` 의 수집이 동시에 진행된다는 것을 의미한다.
	
		![그림 1]({{site.baseurl}}/img/java/concept-jvm-garbagecollection-16.PNG)
	
	1. `After Copying/Cleanup` : `G1 GC` 가 선택한 `Regoin` 으로 살아남은 객체를 `Collected and Compacted` 를 수행한다.
	
	
---
## Reference
[Java Garbage Collection](https://d2.naver.com/helloworld/1329)  
[JDK 8 Garbage Collection Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)  
[What is Java Garbage Collection? How It Works, Best Practices, Tutorials, and More](https://stackify.com/what-is-java-garbage-collection/)  
[Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)  
[Getting Started with the G1 Garbage Collector](https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)  
[Java - Garbage Collection(GC,가비지 컬렉션) 란?](https://coding-start.tistory.com/206)  