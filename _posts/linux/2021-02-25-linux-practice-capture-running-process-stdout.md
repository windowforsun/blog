--- 
layout: single
classes: wide
title: "[Linux 실습] 실행 중인 프로세스 STDOUT 추적/출력 하기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: '실행 중인 프로세스에서 발생하는 STDOUT 을 실제로 확인하는 방법에 대해서 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Python
  - Kubernetes
  - STDOUT
toc: true
use_math: true  
---  

## 실행 중인 프로세스 STDOUT 추적하기
`Docker`, `Kubernetes` 를 기반으로 런타임 환경을 구성하다보면 `STDOUT`, `STDERR` 의 사용빈도와 필요성이 느껴지곤 한다. 
최근 컨테이너에서 실행 중인 프로세스의 `STDOUT` 출력 주기가 의도한 바와 맞지 않아, 
해당 프로세스에서 실제로 출력하는 `STDOUT` 을 추적하는 방법에 대해 알아보았다.  

먼저 방법은 간단하다. 
먼저 `ps aux` 명령으로 실행 중인 프로세스 중 `STDOUT` 을 추적할 프로세스 `PID` 를 찾는다. 
그리고 `cat /proc/<PID>/fd/1` 명령을 수행하면, 실제로 해당 프로세스에서 출력되는 `STDOUT` 을 눈으로 확인해 볼 수 있다.  

추가로 `ls -l /proc/<PID>/fd` 를 수행하면 아래와 같은 리스트가 출력되는데 의미는 아래와 같다. 

```bash
/proc/<PID>/fd/0 # STDIN
/proc/<PID>/fd/1 # STDOUT
/proc/<PID>/fd/2 # STDERR
```  

## 테스트
`Kubernetes` 를 활용해서 간단한 테스트를 진행해본다. 
언어는 `Python` 을 사용해서 테스트를 진행한다. 

>`Python` 의 스크립트를 실행하면 `STDOUT` 출력은 버퍼링 동작에 의해 출력된다. 
>즉 `STDOUT` 출력이 실시간으로 출력되는 것이 아니라 설정된 일정 버퍼를 모두 채운 후 한번에 출력되는 동작으로 수행된다. 

먼저 파드를 구성하는 `pod.yaml` 내용은 아래와 같다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: python-pod
  labels:
    app: python-pod
spec:
  containers:
    - name: buffering
      image: python:alpine
      command:
        - python
        - /test/buffering.py
      volumeMounts:
        - name: script
          mountPath: /test
          readOnly: true
    - name: nobuffering
      image: python:alpine
      command:
        - python
        - /test/nobuffering.py
      volumeMounts:
        - name: script
          mountPath: /test
          readOnly: true
  volumes:
    - name: script
      configMap:
        defaultMode: 0777
        name: test-script
```  

- `buffering` 컨테이너에서는 버퍼링을 사용하는 스크립트를 실행한다. 
- `nobuffering` 컨테이너에서는 버퍼링을 사용하지 않는 스크립트를 실행한다. 

`Pod` 의 `command` 에서 사용하는 `Python` 스크립트를 내용이 있는 `ConfigMap` 템플릿은 아래와 같다. 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-script
  namespace: default
data:
  buffering.py: |
    import time
    import sys

    sys.stdout = open(1, "w", buffering = 2)

    while True:
        time.sleep(0.01)
        timestamp = time.time()
        print(timestamp)
  nobuffering.py: |
    import time
    import sys

    while True:
        time.sleep(0.01)
        timestamp = time.time()
        print(timestamp)
        sys.stdout.flush()
```  

- 두 스크립트는 모두 `0.01` 초 슬립 후 현재 `Timestamp` 를 출력한다. 
- `buffering.py` 는 임의로 줄인 버퍼링 공간을 사용한다. 
- `nobuffering.py` 는 버퍼를 사용하지 않고 바로 `flush` 를 수행한다. 
    - 추가로 `Python` 에서 버퍼를 사용하지 않는 방법으로는 명령어를 수행할 때 옵션 `-u` 다음과 같이주면 된다. `python -u <script>.py` 

두 템플릿을 모두 `Kubernetes` 에 적용하고 정상 동작여부를 확인한다.      

```bash
$ kubectl apply -f configmap.yaml
configmap/test-script created
$ kubectl apply -f pod.yaml
pod/python-pod created
$ kubectl get configmap,pod
NAME                          DATA   AGE
configmap/test-script         3      11s

NAME                                 READY   STATUS    RESTARTS   AGE
pod/python-pod                       2/2     Running   0          6s
```  

`buffering` 컨테이너에 접속해서 실행 중인 프로세스의 `STDOUT` 을 추적하면 아래와 같다. 

```bash
$ kubectl exec -it python-pod -c buffering -- /bin/sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:01 python /test/buffering.py
    7 root      0:00 /bin/sh
   13 root      0:00 ps aux
/ # cat /proc/1/fd/1

.. 일정 시간 대기 ..

1614201371.1130676
1614201371.1232424
1614201371.1333559
1614201371.1435351
1614201371.1537154
1614201371.1638806
1614201371.174033
1614201371.1842384
1614201371.1944118
1614201371.2045991
1614201371.2148013
1614201371.2249327

.. 반복 ..
```  

`buffering` 컨테이너는 버퍼를 사용하기 때문에 일정시간 대기후 한번에 버퍼에 있는 `STDOUT` 내용을 출력하는 동작을 반복한다.  

`nobuffering` 컨테이너에서도 `STDOUT` 를 동일하게 추적하면 아래와 같다. 

```bash
$ kubectl exec -it python-pod -c nobuffering -- /bin/sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 python /test/nobuffering.py
    8 root      0:00 /bin/sh
   15 root      0:00 ps aux
/ # cat /proc/1/fd/1
1614201529.6443202
1614201529.744668
1614201529.8449268
1614201529.945202
1614201530.0454514
1614201530.1457365
1614201530.246049
1614201530.3463488
1614201530.4466588
1614201530.5469842
1614201530.6475396
1614201530.748022
1614201530.8483

.. 반복 ..
```  

`nobuffering` 은 버퍼를 사용하지 않기 때문에 일정시간 대기 없이 `STDOUT` 이 수행되면 즉각적으로 출력된다.  

---
## Reference
[See the STDOUT redirect of a running process](https://unix.stackexchange.com/questions/15693/see-the-stdout-redirect-of-a-running-process)  
[How to capture stdout of a running process redirected to /dev/null](https://unix.stackexchange.com/questions/81774/how-to-capture-stdout-of-a-running-process-redirected-to-dev-null)  
	