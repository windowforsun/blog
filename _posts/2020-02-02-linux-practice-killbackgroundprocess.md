--- 
layout: single
classes: wide
title: "[Linux 실습] 백그라운드 프로세스 종료시키기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Background 에서 실행중인 Process 를 강제로 종료 시켜보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
---  

## 환경
- CentOS 7

## Kill
- 프로세스를 강제로 종료시키는 방법은 `Kill` 명령어를 통해 가능하다.
	
	```bash
	$ kill -9 <pid>
	```  
	
- 프로세스를 강제로 종료시키기 위해서는 PID 를 알아야 하는데 `ps auxf` 명령어를 통해 가능하다.

	```bash
	# 전체 프로세스 트리 출력
	$ ps auxf
	
	# 검색 키워드와 관련된 프로세스 트리 출력
	$ ps auxf | grep <검색키워드>
	```  
	
- 현재 Bash 에서 실행한 백그라운드 작업이라면 `jobs` 명령를 통해 좀 더 쉽게 종료 시킬 수 있다.

	```bash
	$ sleep 100 &
	[1] 12109
	$ jobs
	[1]+  Running                 sleep 100 &
	$ kill %1
	$ jobs
	[1]+  Terminated              sleep 100
	$ jobs
	
	```  
	
## 테스트
- 백그라운드에서 여러 쓰래드로 명령어를 수행하는 간단한 스크립트를 아래와 같이 작성한다.

	```bash
	#!/bin/bash
	# background-thread.sh
	
	thread_count=3
	
	function printDateTime() {
        thread_index=$1

        while [ 1]
        do
        	# 날짜 시간
            datetime=$(date "+%Y-%m-%d %T.%6N")
            
            # Thread 인덱스와 날짜 시간을 출력한다.
            echo "thread_index: ${thread_index}, time : ${datetime}"
            
            # 1초 슬립
            sleep 1
        done
	}
	
	# thread_count 수만 큼 백그라운드로 Thread 생성 및 실행
	for ((i=0; i < $thread_count; i++)) do
        # execute background
        printDateTime $i &
	done
	```  
	
- `. background-thread.sh` 명령어로 실행 시키면 아래와 같이, 현재 Bash 에서 3개의 Thread 가 날짜 시간을 출력한다.
	
	```bash
	$ . background-thread.sh
	thread_index: 0, time : 2020-02-02 21:30:59.104670
	thread_index: 2, time : 2020-02-02 21:30:59.105817
	thread_index: 1, time : 2020-02-02 21:31:00.107640
	thread_index: 2, time : 2020-02-02 21:31:00.108621
	thread_index: 0, time : 2020-02-02 21:31:00.109698
	thread_index: 2, time : 2020-02-02 21:31:01.112937
	thread_index: 1, time : 2020-02-02 21:31:01.113078
	
	.. 반복 ..
	```  
	
- 현재 Bash 에서도 가능하지만, 계속 출력되기 때문에 다른 Bash 에서 아래 작업을 수행한다.
- `ps auxf` 명령어의 결과에서 스크립트의 명령어인 `sleep 1` 이 있는 부분을 찾는다.

	```bash
	$ ps auxf
	root     10482  0.0  0.0 116680  3400 pts/1    S+   16:21   0:00              \_ bash
	root     18592  0.0  0.0 116680  2184 pts/1    S    16:30   0:00                  \_ bash
	root      2022  0.0  0.0 107952   616 pts/1    S    16:42   0:00                  |   \_ sleep 1
	root     18593  0.0  0.0 116680  2184 pts/1    S    16:30   0:00                  \_ bash
	root      2019  0.0  0.0 107952   616 pts/1    S    16:42   0:00                  |   \_ sleep 1
	root     18594  0.0  0.0 116680  2184 pts/1    S    16:30   0:00                  \_ bash
	root      2018  0.0  0.0 107952   612 pts/1    S    16:42   0:00                      \_ sleep 1
	```  
	
- 검색된 18592, 18593, 18594 을 종료시켜 준다.

	```bash
	$ kill -9 18592
	$ kill -9 18593
	$ kill -9 18594
	```  
	
- 스크립트를 실행했던 Bash 에 아래와 같이 출력되면서 스크립트에서 실행했던 쓰래드는 모두 종료되었다.

	```bash
	[1]   Killed                  printDateTime $i
	[2]-  Killed                  printDateTime $i
	[3]+  Killed                  printDateTime $i
	```  
	
---
## Reference
[background process 종료하는 법](https://brown.ezphp.net/entry/background-process-%EC%A2%85%EB%A3%8C%ED%95%98%EB%8A%94-%EB%B2%95)