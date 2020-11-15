--- 
layout: single
classes: wide
title: "[Java 개념] Fork/Join Framework"
header:
  overlay_image: /img/java-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Concept
    - Java
    - ForkJoinPool
    - Fork/Join Framework
toc: true
use_math: true
---  

## Fork/Join Framework
`Fork/Join Framework` 는 멀티 프로세서를 활용할 수 있도록 지원하는 `ExecutorService` 인터페이스의 구현체로 구성된다. 
하나의 큰 문제를 분할 가능한 작은 문제로 반복해서 분할해서, 작은 문제를 해결하고 그 결과를 합쳐 큰 문제를 해결 및 결과를 도출하는 방법을 사용한다. 
이는 분할정복 알고리즘과 비슷한 성격을 띈다. 
그리고 시스템에서 사용가능한 모든 프로세서의 자원을 최대한으로 활용해서 문제 해결에 대한 성능을 향상시키는 것을 목표로 하고 있다.  

`For/Join` 의 과정을 나열하면 아래와 같다. 
1. 큰 문제를 작은 단위로 분할 한다. 
1. 부모 스레드로 부터 처리로직을 복사해서 새로운 스레드에 분할된 문제를 수행(`Fork`) 시킨다. 
1. 더 이상 `Fork` 가 일어나지 않고, 분할된 모든 문제가 완료될 때까지 위 과정을 반복한다. 
1. 분할된 모든 문제가 완료되면, 분할된 문제의 결과를 `Join` 해서 취합한다. 
1. 사용한 모든 스레드에 대해 위 과정을 반복하면서 큰 문제의 결과를 도출해 낸다. 

위 과정을 그림으로 도식화 하면 아래와 같다. 

![그림 1]({{site.baseurl}}/img/java/concept_parallelstream_forkjoin_1.png)  


`ExecutorService` 를 사용한 다른 구현체와 비슷하게 스레드에 처리할 작업을 할당한다. 
차이점이 있다면 `Fork/Join` 은 [work-stealing](https://en.wikipedia.org/wiki/Work_stealing)
알고리즘을 사용해서 `ExecutorService` 를 구현했기 때문에 노는 스레드 없이 모든 스레드가 함께 처리를 계속해서 문제를 빠르게 해결한다. 

![그림 1]({{site.baseurl}}/img/java/concept_parallelstream_forkjoin_2.png)  

1. 앞서 설명한 것처럼 큰 문제를 작은 문제(작업)로 분할 한다. 
1. 분할된 작업은 `ForkJoinPool` 에서 관리하는 `inbound queue` 에 `submit` 한다. 
1. 할당된 스레드들에서 `inbound queue` 에 있는 작업을 `take` 한다. 
1. 스레드는 `take` 한 작업을 각 스레드에서 관리하는 `deque` 에 `push` 한다. 
1. 각 스레드는 자신의 `deque` 에 있는 작업을 `pop` 하며 분할된 작업을 계속해서 처리한다. 
1. 만약 스레드가 자신의 `deque` 의 작업도 모두 처리하고 `inbound queue` 에도 작업이 없다면, 
다른 스레드의 `deque` 에서 작업을 `steal` 해서 처리한다. 

위와 같이 `Fork/Join` 은 하나의 큰 작업이 모두 완료되기 전까지 할당된 스레드들이 `Idle` 한 상태로 대기하지 않고, 
분할된 모든 작업을 빠른 시간내에 처리하는 방법을 사용한다.  

`Fork/Join Framework` 를 구성하는 주요 클래스는 아래와 같다. 
- `ForkJoinPool` : `Fork/Join` 방식으로 분할 및 정복을 수행하는 `Fork/Join Framework` 의 메인 클래스이면서 `ExecutorService` 의 구현체이다. 
- `RecursiveTask<V>` : 결과가 존재하는 작업으로, 실제 작업은 해당 클래스를 상속해서 `compute` 메소드에 처리과정을 구현하고 결과를 리턴한다. 
- `RecursiveAction` : 결과가 존재하지 않는 작업으로 작업은 해당 클래스를 상속해서  `compute` 메소드에  처리과정을 구현한다. 
- `ForkJoinTask<V>` : `RecursiveTask`, `RecursiveAction` 의 부모 클래스로 `fork`, `join` 메소드가 정의돼 있고, `Future` 의 구현체이다. 


### RecursiveAction
`RecursiveAction` 은 결과(리턴)이 존재하지 않는 작업을 `Fork/Join` 방식의 처리를 구현하는 클래스이다. 
`RecursiveAction` 클래스를 상속 받고 추상 메소드인 `compute()` 에 처리할 작업을 구현하는 방식으로 사용 가능하다. 
아래 테스트 코드는 간단하게 출력만 수행하는 구현체의 테스트 코드이다. 

```java
public class RecursiveActionTest {
    public static int coreCount = Runtime.getRuntime().availableProcessors();

    @Test
    public void printSimpleRecursiveAction() {
        ForkJoinPool forkJoinPool = new ForkJoinPool(coreCount);
        PrintSimpleRecursiveAction action = new PrintSimpleRecursiveAction(32);
        forkJoinPool.invoke(action);
    }

    public class PrintSimpleRecursiveAction extends RecursiveAction {
        private long workLoad;

        public PrintSimpleRecursiveAction(long workLoad) {
            this.workLoad = workLoad;
        }

        @Override
        protected void compute() {
            if (this.workLoad > 4) {
                System.out.println("spliting workload : " + this.workLoad);

                List<PrintSimpleRecursiveAction> subTasks = new ArrayList<>();
                subTasks.addAll(this.createSubTasks());

                for (RecursiveAction subTask : subTasks) {
                    subTask.fork();
                }
            } else {
                System.out.println("Doing workload myself: " + this.workLoad);
            }
        }

        private List<PrintSimpleRecursiveAction> createSubTasks() {
            List<PrintSimpleRecursiveAction> subTasks = new ArrayList<>();

            long subTaskWorkLoad = this.workLoad / 2;
            PrintSimpleRecursiveAction subTask1 = new PrintSimpleRecursiveAction(subTaskWorkLoad);
            PrintSimpleRecursiveAction subTask2 = new PrintSimpleRecursiveAction(subTaskWorkLoad);

            subTasks.add(subTask1);
            subTasks.add(subTask2);

            return subTasks;
        }
    }

    @Test
    public void listPrintSimpleRecursiveAction() {
        ForkJoinPool forkJoinPool = new ForkJoinPool(coreCount);
        List<Integer> list = IntStream.iterate(1, (i) -> i + 1)
                .limit(32)
                .boxed()
                .collect(Collectors.toList());
        ListPrintSimpleRecursiveAction action = new ListPrintSimpleRecursiveAction(list);
        forkJoinPool.invoke(action);
    }

    public class ListPrintSimpleRecursiveAction extends RecursiveAction {
        private List<Integer> source;

        public ListPrintSimpleRecursiveAction(List<Integer> source) {
            this.source = source;
        }

        @Override
        protected void compute() {
            if (this.source.size() > 4) {
                List<ListPrintSimpleRecursiveAction> subTasks = this.createSubTasks();

                for (RecursiveAction subTask : subTasks) {
                    subTask.fork();
                }
            } else {
                System.out.println(Arrays.toString(source.toArray()));
            }
        }

        private List<ListPrintSimpleRecursiveAction> createSubTasks() {
            List<ListPrintSimpleRecursiveAction> subTasks = new ArrayList<>();

            int midIndex = this.source.size() / 2;
            subTasks.add(new ListPrintSimpleRecursiveAction(this.source.subList(0, midIndex)));
            subTasks.add(new ListPrintSimpleRecursiveAction(this.source.subList(midIndex, this.source.size())));

            return subTasks;
        }
    }
}
```  

`FrokJoinPool` 클래스를 사용해서 스레드 풀을 생성하는데, 
그 개수는 현재 시스템의 코어수로 지정한다.  

그리고 `PrintSimpleRecursiveAction` 클래스는 생성자의 인자로 `workLoad` 수를 받아, 
4보다 큰 경우 이를 `PrintSimpleRecursiveAction` 객체를 만들어 2개의 작업으로 분리하고 `fork()` 메소드를 호출한다. 
그리고 `workLoad` 의 수가 4보다 작은 경우 지정된 실제 작업인 `System.out.println()` 메소드를 호출해 현재 자신의 `workLoad` 를 출력한다. 
테스트 코드와 같이 초기 `workLoad` 의 수를 32로 지정할 경우 출력문은 실제 순서는 다를 수 있겠지만 아래와 같다. 

```
spliting workload : 32
spliting workload : 16
spliting workload : 16
spliting workload : 8
spliting workload : 8
spliting workload : 8
spliting workload : 8
Doing workload myself: 4
Doing workload myself: 4
Doing workload myself: 4
Doing workload myself: 4
Doing workload myself: 4
Doing workload myself: 4
Doing workload myself: 4
Doing workload myself: 4
```  

다음으로 `ListPrintSimpleRecursiveAction` 클래스는 생성자의 인자로 `List` 를 받아 이를 지정된 크기로 분리해서 출력하는 동작을 수행한다. 
마찬가지로 `List` 의 크기가 4보다 큰 경우 배열을 2개로 분리해서 `ListPrintSimpleRecursiveAction` 객채를 만들고 `fork()` 메소드를 호출한다. 
4보다 작은 경우는 실제 목표가 되는 작업인 배열 출력을 수행한다. 
테스트 코드와 같이 `List` 의 크키가 32이고, `List` 가 `1 ~ 32` 로 구성된다면 출력은 아래와 같이 
출력 순서는 섞여있지만 1부터 32까지의 수가 모두 출력된 것을 확인 할 수 있다.  

```
[25, 26, 27, 28]
[1, 2, 3, 4]
[9, 10, 11, 12]
[17, 18, 19, 20]
[13, 14, 15, 16]
[29, 30, 31, 32]
[21, 22, 23, 24]
[5, 6, 7, 8]
```  


### RecursiveTask
`RecursiveTask<V>` 은 결과(리턴)이 존재하지 작업을 `Fork/Join` 방식의 처리를 구현하는 클래스이다. 
`RecursiveTask<V>` 클래스를 상속 받고 추상 메소드인 `V compute()` 에 처리할 작업을 구현하는 방식으로 사용 가능하다. 
추가적으로 하위로 분할된 작업의 결과를 병합하는 관련 코드도 필요하다. 

```java
public class RecursiveTaskTest {
    public static int coreCount = Runtime.getRuntime().availableProcessors();

    @Test
    public void listSumRecursiveTask() {
        // given
        ForkJoinPool forkJoinPool = new ForkJoinPool(coreCount);
        int listCount = 1024 * 1024 * 10;
        List<Integer> source = IntStream.generate(() -> 1)
                .limit(listCount)
                .boxed()
                .collect(Collectors.toList());
        ListSumRecursiveTask task = new ListSumRecursiveTask(source);

        // when
        long actual = forkJoinPool.invoke(task);

        // then
        assertEquals(listCount, actual);
    }

    public class ListSumRecursiveTask extends RecursiveTask<Long> {
        private List<Integer> source;

        public ListSumRecursiveTask(List<Integer> source) {
            this.source = source;
        }

        @Override
        protected Long compute() {
            long result = 0;

            if(this.source.size() > 128) {
                List<ListSumRecursiveTask> subTasks = this.createSubTasks();

                for(RecursiveTask<Long> subTask : subTasks) {
                    subTask.fork();
                }

                for(RecursiveTask<Long> subTask : subTasks) {
                    result += subTask.join();
                }
            } else {
                for(int num : this.source) {
                    result += num;
                }
            }

            return result;
        }

        private List<ListSumRecursiveTask> createSubTasks() {
            List<ListSumRecursiveTask> subTasks = new ArrayList<>();

            int midIndex = this.source.size() / 2;
            subTasks.add(new ListSumRecursiveTask(this.source.subList(0, midIndex)));
            subTasks.add(new ListSumRecursiveTask(this.source.subList(midIndex, this.source.size())));

            return subTasks;
        }
    }
}
```  

`LsitSumRecursiveTask` 클래스는 `List` 를 인자로 받아 원소의 합을 구하는 작업을 수행한다. 
배열을 128 개씩 나눠 그 합을 구하고, 128 개씩 나뉜 `List` 는 `for` 문을 통해 원소의 합을 구한다. 
그리고 128 개의 합을 다시 합치는 과정으로 최종적인 초기 `List` 의 전체 원소의 합을 구하게 된다. 
지정된 크기보다 큰 경우 해당 작업을 작은 두개의 작업을 나누고 `fork()` 메소드를 수행하는 것까지는 `RecursiveAction` 과 동일하다. 
차이점으로는 `fork()` 메소드 호출 이후 `join()` 메소드를 호출해서, 
분리된 작업에서 결과를 얻어와 자신에게 할당된 결과를 만들고 이를 리턴한다는 점이다.  


### 성능
멀티 코어 환경에서는 분명히 모든 코어의 자원을 최대한으로 사용하는 것이 성능적으로 빠를 것이다. 
하지만 모든 처리를 `For/Join` 으로 처리하면 모두 성능적인 향상을 가져오는 것은 아니다. 
이를 설명하기 위해 아래 작업을 아래 2가지로 나눈다. 
1. 단위 작업의 처리 비용이 크지 않은 경우
1. 단위 작업의 처리 비용이 큰 경우 

 




















































---
## Reference
[ForkJoinPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html)  
[Fork and Join: Java Can Excel at Painless Parallel Programming Too!](https://www.oracle.com/technical-resources/articles/java/fork-join.html)  
[The fork/join framework in Java 7](http://www.h-online.com/developer/features/The-fork-join-framework-in-Java-7-1762357.html)  
