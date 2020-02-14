--- 
layout: single
classes: wide
title: "[Git] 특정 파일, 특정 디렉토리만 Clone 하기"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'Sparse checkout 을 사용해서 특정 파일, 디렉토리만 Clone 해보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - Sparse checkout
    - Clone
---  

## Sparse checkout
- Git 은 기본적으로 저장소단위로 clone, push, commit 등 동작을 수행한다.
- SVN 에서는 디렉토리 단위로 수행을 할 수 있어서, 특정 디렉토리 clone 등 수행이 가능하다.
- Git 에서도 1.17 버전부터 [sparse checkout](https://git-scm.com/docs/git-read-tree) 이라는 기능이 추가되었는데 저장소의 특징 파일이나, 디렉토리만을 clone 할 수 있다.

## 테스트
- `git-test` 라는 저장소는 아래와 같은 구조로 돼있다.

	```bash
	.
	|-- a
	|   |-- a-file-1
	|   `-- aa
	|       |-- aa-clone-file-1
	|       |-- aa-clone-file-2
	|       `-- clone-dir
	|           `-- clone-file-11
	|-- b
	|   |-- b-clone-file-1
	|   |-- b-file-2
	|   `-- bb
	|       `-- bb-file-1
	|-- c
	|   `-- c-file-1
	`-- file-1
	```  
	
	- `a/aa` 디렉토리와 `b/b-clone-file-1` 파일이 클론 받을 특정 파일과 디렉토리이다.
- 호스트에서 클론받을 디렉토리를 먼저 만들고, `git init` 명령어로 초기화 해준다.

	```bash
	$ mkdir test-sparse-checkout
		
	$ cd test-sparse-checkout/
	
	$ git init
	Initialized empty Git repository in /test-sparse-checkout/.git/
	```  
	
- `git config` 명령어로 `sparse chekcout` 을 활성화 시켜준다.

	```bash
	$ git config core.sparseCheckout true
	```  
	
- `git remote add -f origin <REMOTE_REPO_URL>` 명령어로 원격 git 저장소를 추가해 준다.

	```bash
	$ git remote add -f origin https://github.com/windowforsun/git-test.git
	Updating origin
	remote: Enumerating objects: 22, done.
	remote: Counting objects: 100% (22/22), done.
	remote: Compressing objects: 100% (18/18), done.
	remote: Total 26 (delta 3), reused 19 (delta 0), pack-reused 4
	Unpacking objects: 100% (26/26), done.
	From https://github.com/windowforsun/git-test
	 * [new branch]      master     -> origin/master
	```  
	
- `.git/info/sparse-checkout` 파일에 클론할 경로를 명시해 준다.

	```bash
	$ echo "a/aa" >> .git/info/sparse-checkout

	$ echo "b/b-clone-file-1" >> .git/info/sparse-checkout
	```  
	
- `git pull` 명령어로 pull 받는다.

	```bash
	$ git pull origin master
	From https://github.com/windowforsun/git-test
	 * branch            master     -> FETCH_HEAD
	```  
	
- pull 받아진 현재 디렉토리 구성을 확인하면 아래와 같다.
	
	```bash
	$ tree
	.
	|-- a
	|   `-- aa
	|       |-- aa-clone-file-1
	|       |-- aa-clone-file-2
	|       `-- clone-dir
	|           `-- clone-file-11
	`-- b
	    `-- b-clone-file-1
	
	4 directories, 4 files
	```  
	
	- `a/aa` 경로의 하위 디렉토리와 파일들, `b/b-cline-file-1` 파일만 로컬 저장소로 가져온 것을 확인 할 수 있다.

>저장소의 크기가 너무커서(30GB 이상) 필요한 특정 디렉토리만 pull 을 받고 싶어서 조사를 했을 때 찾은 기능이 `sparse checkout` 이다. 
>하지만 테스트 해본 결과 `git remote add` 명령어를 수행하며 저장소에 있는 정보를 다 `.git` 에 저장하기 때문에 명시한 파일 혹은 디렉토리만 보이고, pull, commit, push 를 수행할 수 있지만
>실제 용량은 동일하게7GB 이상이다. 
<!-->다른 자료에서 봤던 것과 같이, 명시한 디렉토리나 파일만 보이도록 하는 기능인것 같다.-->


---
 
## Reference



