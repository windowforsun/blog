--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 컨트롤러(Job, CronJob)"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스 클러스터에서 파드를 관리하는 컨트롤러 중 Job, CronJob 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Controller
  - Job
  - CronJob
toc: true
use_math: true
---  

## 잡
잡(`Job`) 은 실행 후 종료하는 방식의 작업을 수행할 때 사용한다. 
지정한 수의 파드를 정상적으로 실행하고 종료하는 것을 보장한다.  

만약 잡 컨트롤러가 파드를 실행할 때 실패하건, 장애 발생 등의 문제가 발생하게 되면, 
잡 컨트롤러는 이후에 다시 파드를 실행한다. 또한 하나의 잡에서 여러 파드를 구성하는 것도 가능하다. 

### 잡 템플릿
아래는 `Alpine` 리눅스 이미지를 사용해서 쉘을 통해 현재 시간을 출력하는 간단한 명령을 수행하는 잡을 정의한 템플릿이다. 

```yaml
# job-time.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: time
spec:
  template:
    spec:
      containers:
        - name: time
          image: alpine
          command: ["date"]
      restartPolicy: Never
  backoffLimit: 4
```  

- `.apiVersion` : 잡 템플릿은 `batch/v1` 이라는 배치 작업을 실행하는 `API` 를 사용한다. 
- `.spec.template.spec.containers[].command` : `Alpine` 리눅스에서 실행할 쉘 명령어를 작성한다. 
- `.spec.template.spec.restartPolicy` : 재시작에 대한 정책을 설정하는 부분으로 `Never` 로 설정해 한번 실행해 완료되면 종료 되도록 한다. 
이외에 `OnFailure` 은 다양한 이유로 비정상 종료되면 다시 시작시킨다. 
- `.spec.backoffLimit` : 잡을 실행할 때 실행이 실패하면 몇번 재시작 할지에 대한 설정이다. 
기본 값은 6이고, 재시작 횟수가 늘어날 수록 재시작 간격을 점차적으로 늘려 간다. 
이러한 것을 파드 비정상 실행 종료의 백오프(`backoff`) 라고 한다. 

구성한 잡 템플릿은 `kubectl apply -f job-shell.yaml` 명령으로 클러스터에 적용 할 수 있다. 
그리고 생성된 잡 상태 확인을 위해 `kubectl describe job <잡이름>` 을 수행한다. 

```bash
$ kubectl apply -f job-time.yaml
job.batch/time created
$ kubectl describe job time
.. 생략 ..

Parallelism:    1
Completions:    1
Start Time:     Thu, 09 Jul 2020 22:59:38 +0900
Completed At:   Thu, 09 Jul 2020 22:59:43 +0900
Duration:       5s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed

.. 생략 ..
```  

`kubectl describe` 명령으로 시작시간, 완료시간, 걸린시간, 파드 상태 정보를 확인 할 수 있다.  

잡 실행의 결과인 출력한 시간값은 `time` 잡에서 실행된 파드의 로그를 통해 확인 할 수 있다. 

```bash
$ kubectl get pods
NAME         READY   STATUS      RESTARTS   AGE
time-6hf4j   0/1     Completed   0          50s
$ kubectl logs time-6hf4j
Thu Jul  9 13:59:42 UTC 2020
```  

`kubectl get jobs` 로 잡을 확인하면 아래와 같다. 

```bash
kubectl get jobs
NAME   COMPLETIONS   DURATION   AGE
time   1/1           6s         6s
```  

`COMPLETIONS` 은 완료 횟수/총 작업 수를 의미한다. 
`DURATION` 은 잡을 작업완료 하는데 까지 소요된 시간이다.  

### 병렬 잡
병렬 잡의 설정은 템플릿에서 `.spec.parallelism` 필르 설정으로 가능하다. 
하나의 잡에서 몇개의 파드를 동시에 실행할지를 `잡 병렬성` 이라고 한다. 
기본 값은 1이고, 만약 0으로 설정하게 되면 잡에 수행되는 모든 파드는 종료된다.  

아래는 특정 이유로 인해 `.spec.parallelism` 에 설정한 값보다 적은 파드가 실행되는 경우이다. 
- 수행해야하는 잡의 수가 10이고, 그중 정상 완료된 잡이 9개이면 `.spec.parallelism` 값이 2라도 1개의 파드면 실행된다. 
- 워크 큐용 잡에서는 파드 하나가 정상적으로 완료되었을 때 새로운 파드가 실행되지 않는다. 현재 실행 중인 잡은 완료될 때까지 실행한다. 
- 잡 컨트롤러가 반응하지 않는 경우
- 자원, 권한 부족등으로 파드 실행이 불가한 경우
- 잡에서 실패한 횟수가 많아 더이상 해당 잡에 새로운 파드를 생성하지 못하는 상황
- 파드가 그레이풀하게 종료된 상황

### 잡 종류
잡 종류로는 단일 잡, 완료된 잡 개수가 있는 병렬 잡, 워크 큐가 있는 병렬 잡이 있다. 

먼저 단일 잡은 아래와 같은 특징이 있다. 
- 잡하나에 파드 하나만 실행ㅔ되고, 
파드가 실행되고 정상종료(파드가 `Succeeded`) 되면 잡 실행은 완료 된다. 
- 정상 종료되야 하는 파드 수 `.spec.completions` 와 병렬성의 수 `.spec.parallelism` 필드를 설정하지 않는다. 
두 필드의 기본값은 모두 1이므로 단일 잡인 상태이다.

다음으로 완료 개수가 있는 병렬 잡은 아래와 같은 특징이 있다. 
- `.spec.completions` 필드에 1이상의 값을 설정한다. 설정한 개수만큼 파드가 정상종료 되면 잡은 완료 된다. 
- `.spec.parallelism` 필드는 별도로 설정하지 않는다. 

마지막으로 워크 큐가 있는 벙렬 잡에는 아래와 같은 특징이 있다. 
- `.spec.parallelism` 필드를 1이상의 값으로 설정한다. 
- `.spec.completions` 값은 따로 설정하지 않으면, `.spec.parallelism` 과 동일하게 설정된다. 
- 잡에 구성되는 파드는 개별적으로 정상종료가 될 수 있다. 
- 실행 된 파드 중 하나라도 정상종료되면 새로운 파드는 실행되지 않는다. 
- 여러개의 파드가 실행됐을 때, 1개의 파드만 정상종료되면 잡은 정상종료 이다.
- 1개의 파드가 정상종료 되면, 다른 파드는 동작하지 않거나 결과를 내지 않고 종료된다. 

### 비정상 종료
비정상 종료를 대비하기 위해 `.spec.template.spec.restartPolicy` 에 재시작 정책을 설정할 수 있다. 
앞서 설명한 것과 같이 `Never` 는 재시작을 하지 않고 새로운 파드를 실행한다.
 그리고 `OnFailure` 은 비정상 종료 되면 동일한 파드를 재시작을 수행한다.  

어느 하나의 설정 값이 절대적으로 좋은 것은 없고, 상황과 여건에 따라 적당한 값을 설정해야 한다. 


### 잡 종료와 삭제
잡이 정상종료되면 파드 생성은 당연히하지 않고, 사용한 파드도 삭제하지 않는다. 
잡 또한 삭제되지 않고 그대로 남아 있다. 
이러한 특징으로 잡이 완료된 상태를 확인하거나 로그, 에러 등을 추후에 확인 및 분석 가능하다.  

잡 수행시간의 제한을 두고 싶은 경우, `.spec.activeDeadlineSeconds` 필드에 시간을 설정하는 방법으로 가능하다. 
설정한 시간이후에도 잡이 종료되지 않으면 `reason:DeadlineExceeded` 의 메시지와 함께 종료된다.  

잡 삭제는 `kubectl delete job <잡이름>` 명령으로 직접 해야 한다. 

### 잡패턴
잡을 구성하는 일반적인 패턴은 아래와 같다.
- 논리적인 작업마다 하나의 잡을 생성하는 것보다는, 
잡 하나에 여러 작업을 구성하는 방식이 더 좋다. 
잡을 생성하는 오버해드가 크고, 
잡 하나가 여러 작업을 수행하는 것이 효율이 더 좋다. 
- 논리적인 작업 개수만큼 파드를 생성하는 것보다는, 
하나의 파드에서 여러 작업을 수행하는 것이 다 좋다. 
파드를 생성하는 오버해드도 무시할 수 없기 때문이다. 
- 워크 큐에서는 `RabbitMQ`, 카프카 같은 메시징 시스템을 사용해야 하기 때문에 기존 애플리케이션, 컨테이너 수정이 필요하다. 


## 크론잡
크론잡(`CronJob`)은 정해진 주기, 정해진 시간에 맞춰 잡을 관리하고 생성하는 것을 의미한다. 
기존 `Linux` 에서 사용할 수 있었던 `cron` 명령어와 동일한 설정으로 사용 할 수 있다.  

크론잡은 설정한 스케쥴링에 맞춰 잡을 실행하는 방식으로 동작한다. 

### 크론잡 템플릿
아래는 잡 템플릿에서 수행한 현재시간 출력을 크론잡으로 동일하게 구성한 템플릿이다. 

```yaml
# cronjob-time.yaml

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: time
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: time
              image: busybox
              args:
                - /bin/sh
                - -c
                - date
          restartPolicy: OnFailure
```  

- `.spec.schedule` : `cron` 명령에서 사용하는 형식을 사용해서 스케쥴을 지정한다. 현재 템플릿은 1분 마다 스케쥴링을 수행한다. 
- `.spec.jobTemplate.spec.template.spec.containers[].image` : 스케쥴링을 수행할 이미지를 설정한다. 
- `.spec.jobTemplate.spec.template.spec.containers[].args[]` : 사용하는 이미지인 `busybox` 에서 수행할 쉘을 작성해서 구체적인 동작을 지정한다. 

`kubectl apply -f cronjob-time.yaml` 명령으로 클러스터에 크론잡을 등록하면, 
`kubectl get cronjobs` 명령으로 조회 할 수 있다. 

```bash
$ kubectl apply -f cronjob-time.yaml
cronjob.batch/time created
$ kubectl get cronjobs
NAME                 SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/time   */1 * * * *   False     0        <none>          14s
```  

조회한 필드중 `SUSPEND` 는 크론잡 스케쥴링이 정지 되었는지를 의미한다. 
현재 `False` 이므로 `cronjob.batch/time` 은 정지되지 않았음을 의미힌다. 
`ACTIVE` 필드는 현재 실행 중인 잡의 개수를 나타낸다. 
그리고 `LAST SCHEDULE` 은 마지막으로 잡을 실행 한 후 지난 시간을 의미한다. 

`kubectl get jobs` 로 크론잡이 실행한 잡을 조회하면 아래와 같다. 

```bash
$ kubectl get jobs
NAME              COMPLETIONS   DURATION   AGE
time-1594311240   1/1           6s         2m40s
time-1594311300   1/1           4s         100s
time-1594311360   1/1           6s         49s
```  

크론잡에서 자신의 이름과 뒤에 숫자를 붙여 실행한 잡을 확인 할 수 있다. 


### 추가 설정
크론잡에서 `.spec.startingDeadlineSeconds` 와 `.spec.concurrencyPolicy` 필드를 사용하면 더욱 세밀한 설정을 할 수 있다.  

먼저 `.spec.startingDeadlineSeconds` 필드는 스케쥴링으로 지정한 시간에 잡이 실행되지 못했을 때, 
설정한 값이 지난 시간에는 잡을 실행하지 않도록 하는 필드이다. 
해당 필드를 설정하지 않을 경우 스케쥴링시간이 한참 지나서 잡이 수행되는 상황이 발생할 수 있다.  

`.spec.concurrencyPolicy` 필드는 크론잡에서 동시성을 설정하는 필드이다. 
기본값은 `Allow` 로 여러 잡을 동시에 수행할 수 있다. 
`Forbid` 로 설정할 경우 동시에 여러 잡을 수행할 수 없게 된다. 
현재 수행 중인 잡이 있는 상태에서 새로운 잡을 실행해야하는 시간이 되면, 
잡을 수행하지 않고 다음 스케쥴링 때로 미룬다. 
`Replace` 로 설정하면 이전에 실행 중인 잡을 새로운 잡으로 대체한다. 

추가로 `.spec.successFulJobHistoryLimit` 와 `.spec.failedJobHistoryLimit` 는 잡의 정상종료나 비정상종료의 히스토리를 관리할 개수를 설정할 수 있는 필드이다. 
`.spec.successFullJobHistoryLimit` 의 기본값은 3이고, `.spec.failedJobHistoryLimit` 의 기본값은 1 이다. 
값을 0으로 설정하게 되면 내역을 저장하지 않게 된다. 

```yaml
# cronjob-time-concurrency.yaml

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: time-concurrency
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 90
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: time
              image: busybox
              args:
                - /bin/sh
                - -c
                - date; sleep 99999
          restartPolicy: OnFailure
```  

- `.spec.startingDeadlineSeconds` : 데드라인시간을 테스트를 위해 90초로 설정했다. 
- `.spec.concurrencyPolciy` : 테스트를 위해 동시성을 제한한다. 
- `.spec.jobTemplate.spec.template.spec.containers[].args[]` : 테스트를 위해 추가로 `sleep` 을 오랬동안 건다.

`kubectl apply -f` 명령을 통해 새로구성한 템플릿을 클러스터에 적용한다. 
현재 `Forbid` 로 설정 돼있기 때문에 1분 마다 수행하는 잡이지만, 
잡 수행시에 99999초 동안 대기 하기 때문에 새로운 잡은 생성되지 않는다. 

```bash
$ kubectl get cronjob,job,pod
NAME                             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/time-concurrency   */1 * * * *   False     1        103s            116s

NAME                                    COMPLETIONS   DURATION   AGE
job.batch/time-concurrency-1594312260   0/1           103s       103s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/time-concurrency-1594312260-sw262   1/1     Running   0          103s
```  

현재 상태에서 `kubectl edit cronjob` 을 통해 `.spec.concurrencyPolicy` 를 `Allow` 로 수정한다. 
그리고 다시 조회하면, 새로운 잡과 파드가 생성된 것을 확인 할 수 있다. 

```bash
$ kubectl edit cronjob time-concurrency

spec:
  concurrencyPolicy: Allow

cronjob.batch/time-concurrency edited

$ kubectl get cronjob,job,pod
NAME                             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/time-concurrency   */1 * * * *   False     2        34s             8m47s

NAME                                    COMPLETIONS   DURATION   AGE
job.batch/time-concurrency-1594312260   0/1           8m34s      8m34s
job.batch/time-concurrency-1594312740   0/1           33s        33s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/time-concurrency-1594312260-sw262   1/1     Running   0          8m34s
pod/time-concurrency-1594312740-cvsrt   1/1     Running   0          33s
```  

새로운 파드는 생성되지만, 종료되는 되지 않고 계속 실행 중인 상태이다.  

이번에는 `Replace` 로 바꾸고 다시 조회를 해서 상태를 확인한다. 

```bash
$ kubectl edit cronjob time-concurrency

spec:
  concurrencyPolicy: Replace

cronjob.batch/time-concurrency edited

$ kubectl get cronjob,job,pod
NAME                             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/time-concurrency   */1 * * * *   False     1        3s              11m

NAME                                    COMPLETIONS   DURATION   AGE
job.batch/time-concurrency-1594312920   0/1           1s         1s

NAME                                    READY   STATUS              RESTARTS   AGE
pod/time-concurrency-1594312260-sw262   1/1     Terminating         0          11m
pod/time-concurrency-1594312740-cvsrt   1/1     Terminating         0          3m2s
pod/time-concurrency-1594312800-d8jmc   1/1     Terminating         0          2m2s
pod/time-concurrency-1594312860-x5s8q   1/1     Terminating         0          62s
pod/time-concurrency-1594312920-nzlx6   0/1     ContainerCreating   0          1s
$ kubectl get cronjob,job,pod
NAME                             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/time-concurrency   */1 * * * *   False     1        38s             11m

NAME                                    COMPLETIONS   DURATION   AGE
job.batch/time-concurrency-1594312920   0/1           36s        36s

NAME                                    READY   STATUS    RESTARTS   AGE
pod/time-concurrency-1594312920-nzlx6   1/1     Running   0          36s
```  

기존에 실행 중이던 잡을 모두 종료 시키고, 새로운 잡을 실행하는 것을 확인 할 수 있다.  

크론잡의 스케쥴링을 중간에 잠시 정지시키고 싶을 때는,
`kubectl edit cronjob` 명령으로 설정을 열고 `.spec.suspend` 필드를 `true` 로 수정해주면 된다. (기본값 `false`)


---
## Reference
