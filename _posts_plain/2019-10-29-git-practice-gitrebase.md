--- 
layout: single
classes: wide
title: "[Git] git rebase -i 사용하기"
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

git rebase -i 사용법과 각 옵션 및 각 설명(pick, squash 등)




## rebase -i
- `rebase -i` 는 이미 커밋한 히스토리를 변경 또는 삭제하고 싶을 때 유용한 명령어이다.
- `-i` 옵션은 `--interactive` 의 약어로 `git rebase` 를 대화형으로 실행하겠다는 의미이다.
- `git rebase -i <수정을 시작할 커밋의 이전 커밋>` 와 같이 명령어를 사용한다.
	- 수정을 시작할 커밋의 이전 커밋 부터 현재 커밋(HEAD) 범위의 리스트가 출력된다.
	- `git rebase -i HEAD~4` 라고 했을 시에, HEAD~(3~1), HEAD 의 커밋 리스트가 출력된다.
- 출력된 커밋 리스트에서 메시지를 수정하거나 커밋을 합치는 등 작업을 할 수 있다.

## 예제
- `/feature/rebase-test` 브랜치에 아래와 같이 4개의 커밋이 있다.
	









































---
 
## Reference






