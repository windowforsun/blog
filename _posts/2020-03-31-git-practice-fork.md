--- 
layout: single
classes: wide
title: "[Git] Fork 를 사용해 저장소 복제 및 관리"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'fork 를 통해 다른 저장소를 복제하고 변경사항까지 적용해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - Fork
toc: true
---  

## Fork 란
- 프로세스에서 다른 프로세스를 만들때 `fork()` 함수를 사용하는 것과 같이, 원본 Repository 를 복사해서 자식 Repository 를 만드는 것을 뜻한다.
- 원본 Repository 와 자식 Repository 는 서로 의존관계에 있어서, 원본 Repository 의 변경사항을 자식 Repository 에 반영할 수 있고, 그 반대도 가능하다.
- 주로 다른 사람의 Repository 에서 어떤 부분을 수정하거나 추가하고 싶을 때 `fork` 기능을 사용한다.
- 상대방의 Repository 를 `fork` 를 통해 그대로 복사한 후에, 수정하거나 추가하고 싶은 부분을 `Push` 하고 원본 Repository 쪽으로 `Pull request` 를 보내면 된다.
- 원본 Repository 의 관리자가 `Pull request` 를 허용하기 전까지 `Push` 한 변경사항은 자식 Repository 에만 적용돼 있다.

## Fork 하기
- Github 에서 `fork` 를 수행하는 더 자세한 설명은 [여기](https://help.github.com/en/github/getting-started-with-github/fork-a-repo)
에서 확인 가능하다.
- 아래 원본이 되는 Repository (windowforsun/test-base) 가 있다.

	![그림 1]({{site.baseurl}}/img/git/practice-fork-1.png)

- 원본 Repository 계정이 아닌 다른 계정으로 접속해서 상단의 `fork` 버튼을 눌러 접속된 계정으로 복사한다.

	![그림 1]({{site.baseurl}}/img/git/practice-fork-2.png)

	![그림 1]({{site.baseurl}}/img/git/practice-fork-3.png)

- 자식 Repository(MInDle/test-base) 를 로컬 Repository 로 `clone` 한다.

	```bash
	$ git clone https://github.com/MInDle/test-base.git
    Cloning into 'test-base'...
    remote: Enumerating objects: 6, done.
    remote: Counting objects: 100% (6/6), done.
    remote: Compressing objects: 100% (4/4), done.
    remote: Total 6 (delta 0), reused 6 (delta 0), pack-reused 0
    Unpacking objects: 100% (6/6), 550 bytes | 14.00 KiB/s, done.
	```  
	
## 원본 Repository 변경사항 적용하기
- 원본 Repository 에서 `base-file` 을 수정하고, `base-file-2` 을 추가했다.

	![그림 1]({{site.baseurl}}/img/git/practice-fork-4.png)

- 원본 Repository 의 변경 사항을 자식 Repository 에 적용하기 위해서는 자식 Repository 에 `remote` 를 추가해 줘야 한다.
	
	```bash
	.. 현재 자식 Repository 에 등록된 remote ..
	$ git remote -v
    origin  https://github.com/MInDle/test-base.git (fetch)
    origin  https://github.com/MInDle/test-base.git (push)
	```  
	
	```bash
	.. 원본 Repository, upstream 이름으로 추가 ..
	$ git remote add upstream https://github.com/windowforsun/test-base.git
	```  
	
	```bash
	$ git remote -v
    origin  https://github.com/MInDle/test-base.git (fetch)
    origin  https://github.com/MInDle/test-base.git (push)
    upstream        https://github.com/windowforsun/test-base.git (fetch)
    upstream        https://github.com/windowforsun/test-base.git (push)
	```  
	
- `git pull upstream` 을 통해 원본 Repository 에 변경사항을 `pull` 받는다.

	```bash
	$ git pull upstream
    remote: Enumerating objects: 6, done.
    remote: Counting objects: 100% (6/6), done.
    remote: Compressing objects: 100% (2/2), done.
    remote: Total 4 (delta 0), reused 4 (delta 0), pack-reused 0
    Unpacking objects: 100% (4/4), 301 bytes | 6.00 KiB/s, done.
    From https://github.com/windowforsun/test-base
     * [new branch]      master     -> upstream/master
    You asked to pull from the remote 'upstream', but did not specify
    a branch. Because this is not the default configured remote
    for your current branch, you must specify a branch on the command line.
	```  
	
- `git merge upstream/master` 으로 현재 자식 Repository 의 `master` 브랜치와 원본 Repository 의 `upstream/master` 브랜치를 병합해, 원본의 변경사항을 적용해 준다.
	
	```bash
	$ git merge upstream/master
    Updating daef48e..7e64656
    Fast-forward
     base-file   | 1 +
     base-file-2 | 1 +
     2 files changed, 2 insertions(+)
     create mode 100644 base-file-2
	```  
	
- 위 상황은 원본 저장소에만 변경사항이 있고, 자식 저장소에는 변경 사항이 없어 `merge` 명령어로 간단히게 완료 하였지만, 복잡한 상황에서는 기존 `branch` 에 다른 `branch` 의 변경사항을 적용하는 것과 같이 수행하면 된다.

## 자식 Repository 변경사항 적용 요청하기
- 자식 Repository 에서 `base-file` 을 수정하고, `child-file` 을 추가하고 `commit`, `push` 한다.

	```bash
	$ git commit -m "add child"
    [master 411d1af] add child
     2 files changed, 2 insertions(+)
     create mode 100644 child-file
	```  

	```bash
	$ git push origin master
    Enumerating objects: 6, done.
    Counting objects: 100% (6/6), done.
    Delta compression using up to 8 threads
    Compressing objects: 100% (2/2), done.
    Writing objects: 100% (4/4), 350 bytes | 350.00 KiB/s, done.
    Total 4 (delta 0), reused 0 (delta 0)
    To https://github.com/MInDle/test-base.git
       daef48e..411d1af  master -> master
	```  
	
- Github 저장소에 방문해 보면 정상적으로 변경사항이 `push` 된 것을 확인 할 수 있다.
  
	![그림 1]({{site.baseurl}}/img/git/practice-fork-5.png)

- `branch` 바로 우측에 있는 `New pull request` 를 눌러 풀리퀘를 수행할 수 있다.

	![그림 1]({{site.baseurl}}/img/git/practice-fork-6.png)

	- 원본 Repository 을 기준으로 어떤 변경사항이 있는지 확인 할 수 있다.
- `Create pull request` 를 누르면 메시지를 입력하고 원본 Repository 에 자신의 변경사항을 `pull request` 보낼 수 있다.

	![그림 1]({{site.baseurl}}/img/git/practice-fork-7.png)
	
## 자식 Repository 변경사항 적용관리
- 원본 Repository 의 계정으로 저장소에 접근하면 요청한 방금 `pull request` 를 확인 할 수 있다.

	![그림 1]({{site.baseurl}}/img/git/practice-fork-8.png)
	
	![그림 1]({{site.baseurl}}/img/git/practice-fork-9.png)
	
	- 메시지를 입력할 수도 있고, 승인, 거절할 수도 있다.
- `Merge pull request` 를 통해 자식 Repository 의 변경사항을 원본 Repository 로 병합한다.

	![그림 1]({{site.baseurl}}/img/git/practice-fork-10.png)

- 원본 Repository 에 변경사항이 적용 된 것을 확인 할 수 있다.
	![그림 1]({{site.baseurl}}/img/git/practice-fork-11.png)

- 자식 Repository 에서는 `git pull upstream master` 를 통해 `fetch`, `merge` 를 수행하고 `git push origin master` 로 자신의 저장소에 `push` 해주면 된다.

---
 
## Reference


