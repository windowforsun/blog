--- 
layout: single
classes: wide
title: "[Git] Remote 에 올라간 파일 삭제하기"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'Git Remote Repository 에 잘못 올라간 파일을 삭제해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
---  

## 원격 저장소

![그림 1]({{site.baseurl}}/img/git/practice-removeremotedata-1.png)

- 위 그림과 같이 현재 `gitignore` 에 등록해 로컬에서만 사용해려 했던 파일들이 원격 저장소로 올라간 상황이다.

## Remote 파일 삭제
- `git rm <name>` 을 통해 원격 저장소와 로컬 저장소에 있는 파일 및 디렉토리를 삭제 할수도 있다.

	```
	$ git rm -r local*
	rm 'local-dir/etc-1'
	rm 'local-dir/etc-2'
	rm 'local-file'
	```  
	
- `git rm --cached <name>` 을 통해 원격 저장소에 있는 파일만 삭제 할 수도 있다.

	```
	$ git rm --cache -r local-dir/
    rm 'local-dir/etc-1'
    rm 'local-dir/etc-2'

	$ git rm --cache local-file
	rm 'local-file'
	```  	

## .gitignore 등록
- 원격 저장소에 올리지 않을 파일들을 `gitignore` 에 등록해준다.

	```
	local-dir/
    local-file
	```  
	
- `git add .gitignore` 을 통해 `.gitignore` 파일의 상태를 staged 로 변경해준다.

	```
	$ git add .gitignore
	```  
	
## commit & push
- `git commit -m "<message>"` 을 통해 변경된 사항을 커밋 한다.

	```
	$ git commit -m "remove local files and modify gitignore"
    [master 6ff8a6a] remove local files and modify gitignore
     4 files changed, 2 insertions(+)
     delete mode 100644 local-dir/etc-1
     delete mode 100644 local-dir/etc-2
     delete mode 100644 local-file
	```  
	
- `git push origin master` 을 통해 커밋을 push 해서 원격 저장소에 적용한다.

	```
	$ git push origin master
    Enumerating objects: 5, done.
    Counting objects: 100% (5/5), done.
    Delta compression using up to 8 threads
    Compressing objects: 100% (2/2), done.
    Writing objects: 100% (3/3), 348 bytes | 174.00 KiB/s, done.
    Total 3 (delta 0), reused 0 (delta 0)
    To https://github.com/windowforsun/git-test
       a3c49ed..6ff8a6a  master -> master
	```  
	
- 원격 저장소를 확인해 보면 로컬 파일들이 삭제된 것을 확인 할 수 있다.
	
	![그림 2]({{site.baseurl}}/img/git/practice-removeremotedata-2.png)


## Reference
---
[[Git] Github에 잘못 올라간 파일 삭제하기](https://gmlwjd9405.github.io/2018/05/17/git-delete-incorrect-files.html)
