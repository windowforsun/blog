--- 
layout: single
classes: wide
title: "[Linux 실습] find 명령어"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'find 명령어어로 파일을 찾는 다양한 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
  - CentOS 7
  - find
---  

## 환경
- CentOS 7

## 기본 사용
- `find <PATH> -name "<FILE>"` 을 통해 파일이름에 해당 하는 파일들을 찾을 수 있다.
	- `FILE` 에는 찾고자 하는 파일이름도 가능하지만, 정규식을 이용한 파일이름 패턴도(`my-*`) 가능하다
- 이름을 통한 명령어 활용은 아래와 같다.
	- `find` : 현재 디렉토리에 있는 파일 및 디렉토리 리스트 표시
	- `find <PATH>` : 대상 디렉토리에 있는 파일 및 디렉토리 리스트 표시
	- `find . -name <FILE>` : 현재 디렉토리에 아래 모든 파일 및 하위 디렉토리에서 파일 검색
	- `find / -name <FILE>` : 전체 시스템(root 디렉토리)에서 파일 검색
	- `find . -name <FILE>*` : 현재 디렉토리에서 파일 이름이 특정 문자열로 시작하는 파일 검색
	- `find . -name *<FILE>` : 현재 디렉토리에서 파일 이름이 특정 문자열로 끝나는 파일 검색
	- `find . -name *<FILE>*` : 현재 디렉토리에서 파일 이름이 특정 문자열을 포함하는 파일 검색
	- `find . -empty` : 빈 디렉토리 또는 크기가 0인 파일 검색
	- `find . -name *.EXE -delete` : 패턴으로 파일을 검색한 후, 모두 삭제
	- `find . -name <FILE> -print0` : 검색된 파일 리스트를 줄 바꿈 없이 이어서 출력
	- `find . -name <FILE> -type f` : 파일만 검색
	- `find . -name <FILE> -type d` : 디렉토리만 검색
	- `find . -size +<MIN>c -and -size -<MAX>c` : 파일 크기를 사용해서 검색
	- `find . -name <FILE> | xargs ls -l` : 검색된 파일에 대한 상세 리스트 출력(ls -l)
	- `find . -name <FILE> > <SAVE_FILE>` : 검색된 결과를 파일로 저장
	- `find . -name <FILE> 2> /dev/null` : 검색 중 에러 메시지 출력하지 않기
	- `find . -maxdept 1 -name <FILE>` : 하위 디렉토리 검색하지 않기
	- `find . -name <FILE> -exec cp {} <PATH> \;` : 검색된 파일 PATH 로 복사(find, cp 연계)

## 최근 파일 찾기

>- atime(접근 시간) : 파일에 마지막으로 접근한 날짜와 시간을 의미한다.
애플리케이션이나 서비스가 시스템 호출을 사용해 파일을 읽을 때마다 파일 접근 시간이 갱신된다.  
>- mtime(수정 시간) : 파일이 마지막으로 수정된 날짜와 시간을 의미한다.
파일 내용이 변경될 때 파일 수정 시간이 갱신된다.  
>- ctime(변경 시간) : 파일이 마지막으로 변경된 날짜와 시간을 의미한다.
파일 속성의 변경(inode)과  파일 데이터의 변경 양쪽 모두에 관련된다.(권한 등)  

- `find` 명령어에서 아래 옵션을 사용하면 최근에 수정되거나 생성된 파일을 찾을 수 있다.
	- `-amin` : 파일에 접근한 시각을 분단위로 검색
	- `-atime` : 파일에 접근한 시각을 일단위로 검색
	- `-cmin`: 파일을 생성한 시각을 분단위로 검색
	- `-ctime` : 파일을 생성한 시각을 일단위로 검색
	- `-mmin` : 파일을 수정한 시각을 분단위로 검색
	- `-mtime` : 파일을 수정한 시각을 일단위로 검색
- 위 옵션들은 `-옵션이름` 뒤에 `-` 또는 `+` 가 붙고, 숫자 값이 들어간다.
	- `find . -mtime -3` : 최근 3일(72시간)이내에 수정된 파일
	- `find . -mmin -120` : 최근 120분(2시간)이내에 수정된 파일
	- `find . -mtime +3` : 최근 3일(72시간)이상 동안 수정되지 않음
	- `find . -mmin +120` : 최근 120분(2시간)이상 동안 수정되지 않음
- `-` 는 현재 시각부터 ~ 입력한 값까지 과거 시간 사이의 파일을 뜻한다.
- `+` 는 입력한 값 과거시간 ~ 더 과거 시간 사이의 파일을 뜻한다.

### 다른 명령어와 연계하기
- `-type f` 옵션은 파일만 찾도록 하는 옵션이다.
- `tar` 명령어와 연계 : 최근 3일이내에 수정된 파일들 모두 압축하기

	```
	$ find . -type f -mtime -3 | xargs tar cvf recently_3.tar
	```  

- `rm` 명령어와 연계 : 최근 3일 이상 동안 수정되지 않은 파일 모두 삭제하기

	```
	$ find . -type f -mtime +3 | xargs rm
	```  


---
## Reference
[]()  