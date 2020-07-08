--- 
layout: single
classes: wide
title: "[Linux 실습] RAM Disk(램디스크)"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: '보조 메모리와 메인 메모리의 속도차이를 극복할 수 있는 RAM Disk 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - RAM Disk
toc: true
use_math: true  
---  


## RAM Disk
`SSD` 의 등장으로 기존 `HDD` 와 비교했을 때 메인 메모리와 보조 메모리의 속도차이는 크게 극복 됐다. 
하지만 여전히 두 메모리간의 속도차이는 무시하지 못하는 수준이다. 
속도의 측면에서 이를 극복 할 수 있는 방법이 바로 `RAM Disk` 이다.  

이렇게 `RAM Disk` 는 메인 메모리를 보조 메모리처럼 사용하는 것을 의미한다. 

## 주의사항
`RAM` 은 휘발성 메모리이기 때문에 `RAM Disk` 로 설정한 경로는 전원이 꺼지게 되면 모든 데이터는 삭제 된다. 
하지만 무엇보다 성능이 더욱 중요한 경우 `RAM Disk` 를 활용해서 성능적인 이슈를 극복할 수 있다. 

## RAM Disk 의 종류
리눅스 기반 `RAM Disk` 를 구성하는 종류로는 아래와 같은 2가지가 있다. 
1. `ramfs`
1. `tmpfs`

### ramfs
비교적 오래된 파일 시스템으로 동적할당으로 크기를 할당한다. 
이러한 특징으로 시스템에서 사용가능한 메모리의 크기를 넘지 않도록 주의해야 한다. 
`ramfs` 에 `1G` 크기로 마운트 된 경로에 `1G` 가 넘는 데이터를 저장하는 동작을 허용하고, 이로 인해 시스템 전체에 문제가 발생할 수 있다. 


### tmpfs
`ramfs` 와 달리 동적으로 크기를 할당하지 않는다. 
`tmpfs` 는 설정한 크기가 넘는 데이터를 저장하지 못한다. 
이러한 특징으로 `tmpfs` 에 마운트 된 경로에 쓰기 작업을 수행할 때 설정한 크기를 넘지 않도록 주의해야 한다. 
설정한 크기를 넘게되면 `No space left on device` 라는 메시지를 보낸다. 


## RAM Disk 만들기
`RAM Disk` 를 만들기 위해서는 먼저 현재 메모리 상태 확인이 필요하다. 
메모리 상태 확인은 `free -h` 명령을 사용한다. 

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           24Gi       2.3Gi       5.7Gi       635Mi        16Gi        21Gi
Swap:         7.0Gi          0B       7.0Gi
```  

명령 결과를 보면 현재 사용가능한 메모리는 `5G` 정도 된다. 
설정을 위해서는 가용 메모리 보다 적은 `RAM Disk` 크기를 할당해야 한다. 
테스트를 위해 `1G` 크기로 할당하도록 한다.  

우선 `RAM Disk` 로 구성할 경로를 만들어 준다. 

```bash
$ mkdir -p /test/ramdisk
```  

`RAM Disk` 구성하는 명령어는 `mount -t tmpfs -o size=<크기> tmpfs <경로>` 을 통해 가능하다. 

```bash
$ mount -t tmpfs -o size=1G tmpfs /test/ramdisk
```  

`RAM Disk` 가 잘 생성 되었는지 `df -h` 명령을 통해 확인해 본다. 

```bash
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdd        251G  1.7G  237G   1% /
tmpfs            13G     0   13G   0% /mnt/wsl
tmpfs            13G     0   13G   0% /sys/fs/cgroup
C:\             459G  407G   52G  89% /mnt/c
D:\              18G   16G  2.3G  88% /mnt/d
E:\             1.9T  800G  1.1T  43% /mnt/e

.. 생략 .. 

# 새로 생성한 RAM Disk
tmpfs           1.0G     0  1.0G   0% /test/ramdisk
```  

### 부팅시 자동 마운트
`RAM Disk` 의 특성상 재부팅시 마운트 경로의 데이터는 물론이고, 추가한 `RAM Disk` 설정도 초기화 된다. 
데이터는 미리 백업이 필요하고, 재부팅 마다 `RAM Disk` 설정을 계속 반복할 수는 없기 때문에 자동 설정 하는 방법이 필요하다.  

자동 설정은 `/etc/fstab` 경로에 아래의 내용을 추가해서 가능하다. 

```bash
tmpfs <경로> tmpfs default,size=1G 0 0
```  

기존에 명령어를 수행했던 것과 같은 설정으로 작성하면 아래와 같다. 

```bash
tmpfs /test/ramdisk tmpfs default,size=1G 0 0
```  

## RAM Disk 설정 해제
마운트 된 `RAM Disk` 를 언마운트는 `unmount <경로>` 명령을 통해 가능하다. 

```bash
$ umount /test/ramdisk
$ df -h | grep /test/ramdisk

```  

## 성능 비교
일반적인 `Disk` 경로와 `RAM Disk` 로 마운트 한 경로의 성능 비교를 위해 `sysbench` 명령어와 `dd` 명령어를 사용한다. 

### dd
`dd` 명령어에 대해 간단하게 설명하면, 
블록 단위로 파일을 복사하거나 파일 변환을 할 수 있는 명령어이다. 
해당 명령을 활용하면 디스크의 성능 측정도 가능하다.  

사용할 명령어의 예시는 아래와 같다. 

```bash
$ dd if=/dev/zero of=<테스트경로> bs=<데이터크기> count=<데이터개수>
```  

`RAM Disk` 에 마운트 된 경로에서 1M 데이터 1000회 테스트를 수행한 결과는 아래와 같다. 

```bash
$ dd if=/dev/zero of=/test/ramdisk/bench bs=1M count=1000; rm -f /test/ramd
isk/bench
1000+0 records in
1000+0 records out
1048576000 bytes (1.0 GB, 1000 MiB) copied, 0.45228 s, 2.3 GB/s
```  

일반 `Disk` 경로에서 동일한 테스트를 수행하면 아래와 같다. 

```bash
$ dd if=/dev/zero of=/test/notramdisk/bench bs=1M count=1000; rm -f /test/n
otramdisk/bench
1000+0 records in
1000+0 records out
1048576000 bytes (1.0 GB, 1000 MiB) copied, 0.828263 s, 1.3 GB/s
```  

`dd` 명령어를 사용한 테스트 결과로는 `RAM Disk` 가 일반 `Disk` 보다 2배 정도의 성능을 보이는 것을 확인 할 수 있다. 


### sysbench
`sysbench` 는 리눅스 성능 측정관련 툴로 다양한 종류의 성능 측정을 제공한다. 
만약 설치가 돼있지 않다면 `yum install sysbench` 혹은 `apt install systench` 를 통해 설치 가능하다.  

`sysbench` 에는 `fileio` 관련 여러 테스트 모드가 존재하는데,
그 중 순차 쓰기(`seqwr`) 와 순차 읽기(`seqrd`) 를 사용해서 테스트를 한다. 
테스트 과정은 1G 데이터를 순차 쓰기로 쓰기 테스트를 수행하고, 
테스트 과정에서 만들어진 데이터를 순차 읽기로 수행해 읽기 테스트를 수행한다. 

먼저 `RAM Disk` 에 마운트 된 경로에 대한 테스트이다. 
앞에서 설명한  것과 같이 쓰기 테스트는 순차 쓰기를 이용한 `sysbench fileio --file-total-size=<데이터크기> --file-test-mode=seqwr run` 명령을 통해 가능하지만 좀더 편의성을 위해 아래 명령어를 사용한다. 
`sysbench fileio --file-total-size=1G prepare`

```bash
$ pwd
/test/ramdisk
$ sysbench fileio --file-total-size=1G prepare
sysbench 1.0.18 (using system LuaJIT 2.1.0-beta3)

128 files, 8192Kb each, 1024Mb total
Creating files for the test...
Extra file open flags: (none)
Creating file test_file.0
Creating file test_file.1
Creating file test_file.2

.. 생략

Creating file test_file.125
Creating file test_file.126
Creating file test_file.127
1073741824 bytes written in 0.41 seconds (2473.96 MiB/sec).
```  

다음으로는 `RAM Disk` 의 읽기 테스트를 위해, 
`sysbench fileio --file-total-size=<데이터크기> --file-test-mode=seqrd run` 를 수행한다. 

```bash
$ pwd
/test/ramdisk
$ sysbench fileio --file-total-size=1G --file-test-mode=seqrd run
sysbench 1.0.18 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Extra file open flags: (none)
128 files, 8MiB each
1GiB total file size
Block size 16KiB
Periodic FSYNC enabled, calling fsync() each 100 requests.
Calling fsync() at the end of test, Enabled.
Using synchronous I/O mode
Doing sequential read test
Initializing worker threads...

Threads started!


File operations:
    reads/s:                      356747.23
    writes/s:                     0.00
    fsyncs/s:                     0.00

Throughput:
    read, MiB/s:                  5574.18
    written, MiB/s:               0.00

General statistics:
    total time:                          10.0001s
    total number of events:              3568104

Latency (ms):
         min:                                    0.00
         avg:                                    0.00
         max:                                    1.19
         95th percentile:                        0.00
         sum:                                 9314.50

Threads fairness:
    events (avg/stddev):           3568104.0000/0.00
    execution time (avg/stddev):   9.3145/0.00
```  

이제 `Disk` 경로에서 동일한 `RAM Disk` 마운트 경로에서 수행했던 동일한 순서와 명령어를 수행한다.  

`Disk` 쓰기 테스트

```bash
$ pwd
/test/notramdisk
$ sysbench fileio --file-total-size=1G prepare
sysbench 1.0.18 (using system LuaJIT 2.1.0-beta3)

128 files, 8192Kb each, 1024Mb total
Creating files for the test...
Extra file open flags: (none)
Creating file test_file.0
Creating file test_file.1
Creating file test_file.2

.. 생략 ..

Creating file test_file.125
Creating file test_file.126
Creating file test_file.127
1073741824 bytes written in 1.86 seconds (551.32 MiB/sec).
```  

`Disk` 읽기 테스트

```bash
$ pwd
/test/notramdisk
$ sysbench fileio --file-total-size=1G --file-test-mode=seqrd run
sysbench 1.0.18 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Extra file open flags: (none)
128 files, 8MiB each
1GiB total file size
Block size 16KiB
Periodic FSYNC enabled, calling fsync() each 100 requests.
Calling fsync() at the end of test, Enabled.
Using synchronous I/O mode
Doing sequential read test
Initializing worker threads...

Threads started!


File operations:
    reads/s:                      356297.70
    writes/s:                     0.00
    fsyncs/s:                     0.00

Throughput:
    read, MiB/s:                  5567.15
    written, MiB/s:               0.00

General statistics:
    total time:                          10.0001s
    total number of events:              3563584

Latency (ms):
         min:                                    0.00
         avg:                                    0.00
         max:                                    1.36
         95th percentile:                        0.00
         sum:                                 9282.64

Threads fairness:
    events (avg/stddev):           3563584.0000/0.00
    execution time (avg/stddev):   9.2826/0.00
```  

테스트가 끝나고 생성된 파일은 `sysbench fileio --file-total-size=1G cleanup` 명령을 통해 정리할 수 있다.  

`sysbench` 를 이용한 테스트 결과를 확인하면, 
우선 쓰기에서는 `RAM Disk` 가 `Disk` 에 비해 4배 이상 빠른 것을 확인 할 수 있다. 
하지만 읽기에서는 큰 차이는 보이지 않았다. 

---
## Reference
[akopytov/sysbench](https://github.com/akopytov/sysbench)
	