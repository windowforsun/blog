--- 
layout: single
classes: wide
title: "[Git] 원격 저장소 여러개 사용하기"
header:
  overlay_image: /img/git-bg.jpg
excerpt: '로컬 저장소 하나에 여러 원격 저장소를 연결해 사용하자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - Multiple Repository
---  

## 기존 저장소 확인
- `git remote -v` 를 통해 현재 로컬 저상소의 연결된 원격 저장소 목록을 확인한다.

	```
	$ git remote -v
	origin  https://github.com/windowforsun/multiple-repo.git (fetch)
	origin  https://github.com/windowforsun/multiple-repo.git (push)
	```  
	
## 다른 원격 저장소 추가
- `https://bitbucket.org/window_for_sun/multiple-repo.git` 주소를 가진 비어있는 Bitbucket 저장소를 새로 추가하려고 한다.
- `git remote add <repository-name> <repository-url> 을 통해 새로운 저장소를 추가한다.

	```
	$ git remote add bitbucket-repo https://bitbucket.org/window_for_sun/multiple-repo.git
	```  

- 다시 `git remote -v` 로 연결된 원격 저장소를 확인한다.

	```
	$ git remote -v
	bitbucket-repo  https://bitbucket.org/window_for_sun/multiple-repo.git (fetch)
	bitbucket-repo  https://bitbucket.org/window_for_sun/multiple-repo.git (push)
	origin  https://github.com/windowforsun/multiple-repo.git (fetch)
	origin  https://github.com/windowforsun/multiple-repo.git (push)
	```  

- `git push <repository-name> <branch-name>` 를 통해 비어있는 Bitbucket 저장소에 현재 GitHub 저장소의 내용을 올린다.

	```
	$ git push bitbucket-repo master
	Enumerating objects: 3, done.
	Counting objects: 100% (3/3), done.
	Writing objects: 100% (3/3), 224 bytes | 37.00 KiB/s, done.
	Total 3 (delta 0), reused 0 (delta 0)
	To https://bitbucket.org/window_for_sun/multiple-repo.git
	 * [new branch]      master -> master
	```  
	
## Push 하기
- `second` 라는 파일을 새로 만든다.

	```
	$ touch second
	```  
	
- `git add` 와 `git commit` 을 통해 커밋한다.
	
	```
	$ git add --all

	$ git commit -m "second commit"
	[master d062ac2] second commit
	 1 file changed, 0 insertions(+), 0 deletions(-)
	 create mode 100644 second
	```  
	
- 2개의 원격 저장소에 각각 푸시 해준다.

	```
	$ git push origin master
	Enumerating objects: 3, done.
	Counting objects: 100% (3/3), done.
	Delta compression using up to 8 threads
	Compressing objects: 100% (2/2), done.
	Writing objects: 100% (2/2), 258 bytes | 36.00 KiB/s, done.
	Total 2 (delta 0), reused 0 (delta 0)
	To https://github.com/windowforsun/multiple-repo.git
	   c46ffae..d062ac2  master -> master
	
	$ git push bitbucket-repo master
	Enumerating objects: 3, done.
	Counting objects: 100% (3/3), done.
	Delta compression using up to 8 threads
	Compressing objects: 100% (2/2), done.
	Writing objects: 100% (2/2), 258 bytes | 64.00 KiB/s, done.
	Total 2 (delta 0), reused 0 (delta 0)
	To https://bitbucket.org/window_for_sun/multiple-repo.git
	   c46ffae..d062ac2  master -> master
	```  
	
## 한번에 여러개 저장소에 Push 하기
- `git config -e` 을 통해 기존 Git 설정 정보를 확인한다.

	```
	$ git config -e
	
	[core]
	        repositoryformatversion = 0
	        filemode = false
	        bare = false
	        logallrefupdates = true
	        symlinks = false
	        ignorecase = true
	[remote "origin"]
	        url = https://github.com/windowforsun/multiple-repo.git
	        fetch = +refs/heads/*:refs/remotes/origin/*
	[branch "master"]
	        remote = origin
	        merge = refs/heads/master
	[remote "bitbucket-repo"]
	        url = https://bitbucket.org/window_for_sun/multiple-repo.git
	        fetch = +refs/heads/*:refs/remotes/bitbucket-repo/*
	
	```  
	
- `git remote add set-url origin --push --add <user-name>@<repository-url>` 혹은 `git remote add set-url origin --push --add <repository-url>` 으로 한번에 푸시하려고 하는 2개 원격 저장소 주소를 추가한다.

	```
	$ git remote set-url origin --push --add https://github.com/windowforsun/multiple-repo.git

	$ git remote set-url origin --push --add https://bitbucket.org/window_for_sun/multiple-repo.git
	```  
	
- 다시 Git 설정 정보를 확인하면 아래와 같다.

	```
	[core]
	        repositoryformatversion = 0
	        filemode = false
	        bare = false
	        logallrefupdates = true
	        symlinks = false
	        ignorecase = true
	[remote "origin"]
	        url = https://github.com/windowforsun/multiple-repo.git
	        fetch = +refs/heads/*:refs/remotes/origin/*
	        pushurl = https://bitbucket.org/window_for_sun/multiple-repo.git
	        pushurl = https://github.com/windowforsun/multiple-repo.git
	[branch "master"]
	        remote = origin
	        merge = refs/heads/master
	[remote "bitbucket-repo"]
	        url = https://bitbucket.org/window_for_sun/multiple-repo.git
	        fetch = +refs/heads/*:refs/remotes/bitbucket-repo/*
	```  
	
- `git remote -v` 로 원격 저장소 정보를 확인하면 아래와 같다.

	```
	$ git remote -v
	bitbucket-repo  https://bitbucket.org/window_for_sun/multiple-repo.git (fetch)
	bitbucket-repo  https://bitbucket.org/window_for_sun/multiple-repo.git (push)
	origin  https://github.com/windowforsun/multiple-repo.git (fetch)
	origin  https://bitbucket.org/window_for_sun/multiple-repo.git (push)
	origin  https://github.com/windowforsun/multiple-repo.git (push)
	```  
	
- `third` 파일을 추가하고, `git add`, `git commit` 을 통해 커밋한다.

	```
	$ touch third

	$ git add --all
	
	$ git commit -m "third commit"
	[master 0e0b77c] third commit
	 1 file changed, 0 insertions(+), 0 deletions(-)
	 create mode 100644 third
	```  
	
- `git push origin master` 를 통해 원격 저장소에 푸시하면 등록된 2개의 원격 저장소에 푸시 된다.

	```
	$ git push origin master
	Enumerating objects: 3, done.
	Counting objects: 100% (3/3), done.
	Delta compression using up to 8 threads
	Compressing objects: 100% (2/2), done.
	Writing objects: 100% (2/2), 264 bytes | 52.00 KiB/s, done.
	Total 2 (delta 0), reused 0 (delta 0)
	To https://bitbucket.org/window_for_sun/multiple-repo.git
	   b9ce345..373d6f5  master -> master
	Enumerating objects: 3, done.
	Counting objects: 100% (3/3), done.
	Delta compression using up to 8 threads
	Compressing objects: 100% (2/2), done.
	Writing objects: 100% (2/2), 264 bytes | 37.00 KiB/s, done.
	Total 2 (delta 0), reused 0 (delta 0)
	To https://github.com/windowforsun/multiple-repo.git
	   b9ce345..373d6f5  master -> master
	```  
	
---
 
## Reference
[pull/push from multiple remote locations](https://stackoverflow.com/questions/849308/pull-push-from-multiple-remote-locations/3195446#3195446)



