--- 
layout: single
classes: wide
title: "[Git] git rebase 사용하기"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'git rebase -i 을 통해 커밋을 정리해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - rebase
---  

## rebase -i
- `rebase -i` 는 이미 커밋한 히스토리를 변경 또는 삭제하고 싶을 때 유용한 명령어이다.
- `-i` 옵션은 `--interactive` 의 약어로 `git rebase` 를 대화형으로 실행하겠다는 의미이다.
- `git rebase -i <수정을 시작할 커밋의 이전 커밋>` 와 같이 명령어를 사용한다.
	- 수정을 시작할 커밋의 이전 커밋 부터 현재 커밋(HEAD) 범위의 리스트가 출력된다.
	- `git rebase -i HEAD~4` 라고 했을 시에, HEAD~(3~1), HEAD 의 커밋 리스트가 출력된다.
- 출력된 커밋 리스트에서 메시지를 수정하거나 커밋을 합치는 등 작업을 할 수 있다.

## rebase 취소하기
- rebase 도중 취소를 해야할 경우 `git rebase --abort` 명령어를 사용한다.
	
## rebase rollback (rebase 되돌리기)
- `git reflog` 를 사용해서 커밋 로그를 확인한다.
	
	```
	f684cb8 (HEAD -> feature/rebase-test) HEAD@{3}: rebase -i (pick): second commit
	f9c521e HEAD@{4}: rebase -i (pick): fourth commit
	45cfdd4 HEAD@{5}: rebase -i (pick): third commit
	52840a2 HEAD@{6}: rebase -i (start): checkout HEAD~3
	80e4b67 HEAD@{7}: commit: fourth commit
	```  
	
- 돌아가고 싶은 `HEAD@{num}` 을 찾고, `git reset HEAD@{7}` 과 같이 입력하면 `fourth commit` 시점으로 리셋된다.


## rebase -i 예제
- `/feature/rebase-test` 브랜치에 아래와 같이 4개의 커밋이 있다.

	![그림 1]({{site.baseurl}}/img/git/practice-gitrebasei-1.png)
	
	- 각 커밋은 first, second, third, fourth 라는 txt 파일을 생성한 커밋이다.
	
- `git rebase -i HEAD~3` 명령어를 수행하면 아래와 같은 화면이 나온다.

	```
	pick 6bf2b88 second commit
	pick 2b4b5d9 third commit
	pick 74c299f fourth commit
	
	# Rebase c06f85b..74c299f onto c06f85b (3 commands)
	#
	# Commands:
	# p, pick <commit> = use commit
	# r, reword <commit> = use commit, but edit the commit message
	# e, edit <commit> = use commit, but stop for amending
	# s, squash <commit> = use commit, but meld into previous commit
	# f, fixup <commit> = like "squash", but discard this commit's log message
	# x, exec <command> = run command (the rest of the line) using shell
	# b, break = stop here (continue rebase later with 'git rebase --continue')
	# d, drop <commit> = remove commit
	# l, label <label> = label current HEAD with a name
	# t, reset <label> = reset HEAD to a label
	# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
	# .       create a merge commit using the original merge commit's
	# .       message (or the oneline, if no original merge commit was
	# .       specified). Use -c <commit> to reword the commit message.
	#
	# These lines can be re-ordered; they are executed from top to bottom.
	#
	# If you remove a line here THAT COMMIT WILL BE LOST.
	#
	# However, if you remove everything, the rebase will be aborted.
	#
	# Note that empty commits are commented out
	```  
	
	- 3개의 커밋 목록을 이제 수정할 수 있다.
	- 아래 주석 부분에 `rebase -i` 관련 명령어에 대한 설명이 작성되어 있다.
	- 작성이 완료되고 저장을 해주면(:wq) 변경사항이 적용된다.
	
	
### pick (p)
- pick 은 커밋을 사용하겠다는 의미이다.
- pick 을 사용해서 커밋의 순서를 변경할 수도 있고, 커밋의 해시값을 이용해 특정 커밋을 가져올 수도 있다.
- 커밋 순서를 변경하고, 저장, 커밋하면 아래와 같이 커밋 순서가 변경된다.

	```
	pick 2b4b5d9 third commit
	pick 74c299f fourth commit
	pick 6bf2b88 second commit
	```  
	
	![그림 2]({{site.baseurl}}/img/git/practice-gitrebasei-2.png)
	
### reword (r)
- reword 는 커밋 메시지를 변경하는 명령어이다.
- 아래와 같이 `reword` 로 변경하고 저장한다.

	```
	reword 6bf2b88 second commit
	pick 2b4b5d9 third commit
	reword 74c299f fourth commit
	```  
	
- `second commit`, `fourth commit` 와 같이 두번 커밋 메시지 입력창이 나오는데 `second2 commit`, `fourth2commit` 으로 변경하고 저장하면 아래와 같이 커밋 메시지가 수정된다.

	![그림 3]({{site.baseurl}}/img/git/practice-gitrebasei-3.png)

- 단순하게 이전 커밋 메시지만 변경해야 할 경우에는 `git commit --amend` 를 사용할 수 있다.

### edit (e)
- `reword` 커멘드는 커밋 메시지만 변경하는 명령어와 달리, edit 은 커밋 메시지와 작업 내용도 변경 가능하다.
- 아래와 같이 `edit` 으로 변경하고 저장한다.

	```	
	pick 6bf2b88 second commit
	edit 2b4b5d9 third commit
	pick 74c299f fourth commit
	```  
	
- 이제 변경할 작업내용을 수행해 주면되는데, third commit 에서 원래 third.txt 파일을 생성했는데, 이를 지우고 third2.txt 파일을 생성했다.
- 작업 내용을 아래와 같이 적용해 준다.
	
	```
	$ git add .
	$ git commit --amend
	tail: cannot open '../HEAD' for reading: No such file or directory
	[detached HEAD 76e8dc9] third commit edit
	 Date: Thu Oct 31 13:37:35 2019 +0900
	 1 file changed, 0 insertions(+), 0 deletions(-)
	 create mode 100644 third2.txt
	$ git rebase --continue
	Successfully rebased and updated refs/heads/feature/rebase-test.
	```  
	
- 커밋이 아래와 같이 변경 된다.

	![그림 4]({{site.baseurl}}/img/git/practice-gitrebasei-4.png)

### squash (s)
- squash 는 해당 커밋을 이전 커밋과 합치는 명령어이다.
- 아래와 같이 `squash` 로 변경하고 저장한다.

	```
	pick 6bf2b88 second commit
	squash 2b4b5d9 third commit
	pick 74c299f fourth commit
	```  

- 아래와 같이 `third commit` 이 이전 커밋인 `second commit` 과 합쳐 지는 커밋 메시지를 작성하는 부분이 나오면 저장해 준다.

	```
	# This is a combination of 2 commits.
	# This is the 1st commit message:
	
	second commit
	
	# This is the commit message #2:
	
	third commit
	
	# Please enter the commit message for your changes. Lines starting
	# with '#' will be ignored, and an empty message aborts the commit.
	#
	# Date:      Thu Oct 31 13:37:22 2019 +0900
	#
	# interactive rebase in progress; onto 52840a2
	# Last commands done (2 commands done):
	#    pick 9e11cba second commit
	#    squash daf2b03 third commit
	# Next command to do (1 remaining command):
	#    pick 80e4b67 fourth commit
	# You are currently rebasing branch 'feature/rebase-test' on '52840a2'.
	#
	# Changes to be committed:
	#       new file:   second.txt
	#       new file:   third.txt
	#
	
	```  
	
- 커밋이 아래와 같이 합쳐진 것을 확인 할 수 있다.

	![그림 5]({{site.baseurl}}/img/git/practice-gitrebasei-5.png)
	
### fixup (f)
- `fixup` 도 `squash` 와 동일하게 커밋을 합치는 명렁어이지만, 커밋 메시지는 합치지 않는다.
- 커밋을 합치지만 이전 커밋 메시지만 남게 된다.

### exec (x)
- exec 명령어는 각 커밋이 적용된 후 실행할 shell 명령어를 지정할 수 있다.
- 아래와 같이 커밋 아래부분에 간단한 shell 명령어를 추가해 준다.

	```
	pick 6bf2b88 second commit
	exec git rev-parse HEAD
	pick 2b4b5d9 third commit
	exec git rev-parse HEAD
	pick 74c299f fourth commit	
	exec git rev-parse HEAD
	```  
	
	- `git rev-parse HEAD` 는 현재 HEAD 가 가리키고 있는 커밋의 해시값을 출력한다.

- 아래와 같이 shell 명령어 결과가 출력된다.
	
	```
	$ git rebase -i HEAD~3
	Executing: git rev-parse HEAD
	9e11cbae9506ba466299ad8d6a0413f98fc5fca3
	Executing: git rev-parse HEAD
	daf2b035efa30297006d293b498f70df6328ea3e
	Executing: git rev-parse HEAD
	80e4b67cb014d27d543f107dc74d7ff7c7421fee
	Successfully rebased and updated refs/heads/feature/rebase-test.
	```  

### drop (d)
- drop 은 커밋 히스토리에서 커밋을 삭제하는 명령어이다.
- 아래와 같이 `drop` 로 변경하고 저장하면 해당 커밋이 삭제된 것이 확인 가능하다.

	```	
	pick 6bf2b88 second commit
	drop 2b4b5d9 third commit
	pick 74c299f fourth commit	
	``` 
	
	![그림 6]({{site.baseurl}}/img/git/practice-gitrebasei-6.png)
	 
## rebase 주의 사항
- rebase 는 이전 커밋에 대해서 변경작업이 수행되기 때문에 사용에 유의해야 한다.
- 이미 remote 저장소에 push 된 커밋을 rebase 한다면, 이는 push 되지 않는다.
	- `git push --force`, `git push -f` 명령어로 강제할 수 있지만, 이런 상황을 git이 꼬였다고 한다.
- 브랜치를 공유하고, 협업인 과정에서 rebase 사용은 지양하고, 자신의 개인 브랜치 커밋을 정리할 때 지향해야 한다.

---
 
## Reference






