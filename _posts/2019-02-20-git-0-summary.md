--- 
layout: single
classes: wide
title: "깃(Git) 정리"
header:
  overlay_image: /img/git-bg.jpg
subtitle: 'git을 간략하게 정리'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - Summary
---  

'git을 간략하게 정리'

# Git 명령어
## Git 설정
- 사용자명/이메일 설정
	- 전역 설정
	
		```
		git config --global user.name "<User Name>"
		git config --global user.name "window_for_sun"
		git config --global user.email <User Email>
		git config --global user.email window_for_sun@domain.com
		```  
		
		- --global 옵션을 사용하지 않으면 해당 저장소만 유효한 설정
- 출력 색상 변경

	```
	git config --global color.ui auto
	```  
	
- 명령어에 Alias(단축키) 설정

	```
	git config --global alias.<Alias Name> <Command Name>
	git config --global alas.ci commit
	```	  
	
- 불필요한 파일을 제외하기
	
	```
	echo <File Name> >> .gitignore
	echo *.class >> .gitignore
	```  
	
- 빈 폴더 관리 대상에 포함시키기

	```
	cd <Dir Name>
	cd /dir/name
	touch .gitkeep
	```  
	
- 프록시 서버를 경유하여 http 접속

- 사용자 인증이 필요한 프록시 서버를 경유하여 http 접속

- 설정 정보 조회
	
	```
	git config --global --list
	```  
	
- 저장소 복제하기

	```
	git clone <Repository URL>
	git clone https://github.com/window_for_sun/test.git
	git clone <Repository URL> <RepositoryName>
	git clone https://github.com/window_for_sun/test.git testRepository
	```  
	
- 원격 저장소 추가

	```
	git remote add <Remote Name> <Remote URL>
	git remote add origin https://github.com/window_for_sun/test.git
	```
	
## 기본 명령어
	
- git 저장소 생성(디렉토리 이동 후)
	
	```
	git init
	```  
	
- 새로운 파일, 폴더를 인덱스(Stage)에 등록하기

	```
	git add <File Pattern>
	git add *.java
	git add test.java
	```  
	
	- "." 를 지정하면 하위 폴더 내의 모든 파일을 인덱스에 등록할 수 있다.
	- -p 옵션을 붙이면, 파일 변경 부분의 일부만 등록할 수 있다.
	- -i 옵션을 붙이면 인덱스에 등록한 파일을 대화식으로 선택 할 수 있다.
	
- 인덱스(Stage)에 추가된 파일을 커밋하기

	```
	git commit
	```  
	
	- -a 옵션을 붙이면 변경된 파일(신규 파일은 제외)을 검출하여 인덱스에 추가하고, 커밋하는 동작 수행
	- -m 옵션을 붙이면 커밋 메시지를 지정하여 커밋
	
- 변경된 파일 목록 확인
	
	```
	git status
	```  
	
	- -s 옵션을 붙이면 설명문을 표시하지 않음
	- -b 옵션을 붙이면 설명문은 표사하지 않으면서 브랜치명은 표시

- 파일의 변경 내용 확인

	```
	git diff
	```  
	
	- 옵션을 지정하지 않으면 작업 트리와 인덱스의 변경 사항 표시
	- -cached 옵션을 붙이면 인덱스와 HEAD의 변경 사항 표시
	- HEAD나 커밋을 지정하면 작업 트리와 지정된 HEAD와의 변경 사항 표시
	
- 커밋 로그 확인
	
	```
	git log
	```  
	
	- 특정 파일의 커밋 로그를 참조하려면 파일명을 지정
	
- 커밋의 상세내용 확인

	```
	git show <Commnit>
	
	```
---
 
## Reference
[자주 사용하는 기초 Git 명령어 정리하기](https://medium.com/@pks2974/%EC%9E%90%EC%A3%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94-%EA%B8%B0%EC%B4%88-git-%EB%AA%85%EB%A0%B9%EC%96%B4-%EC%A0%95%EB%A6%AC%ED%95%98%EA%B8%B0-533b3689db81)
[[Git] Git 명령어 정리](https://medium.com/@joongwon/git-git-%EB%AA%85%EB%A0%B9%EC%96%B4-%EC%A0%95%EB%A6%AC-c25b421ecdbd)
[누구나 쉽게 이해할 수 있는 Git입문](https://backlog.com/git-tutorial/kr/)






