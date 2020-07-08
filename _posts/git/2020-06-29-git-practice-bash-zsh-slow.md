--- 
layout: single
classes: wide
title: "[Git] Bash, Zsh 사용시 Git 저장소에서 명령어 수행 속도 올리기"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'Git 저장소에서 명령어 수행 속도를 Git Config 를 통해 해결해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - Git Bash
    - Zsh
    - Oh My Zsh
toc: true
---  

`Git Bash` 를 사용하거나, `Zsh`(`Oh My Zsh`) 를 사용할 떄 `Git` 저장소 디렉토리에 접근하거나, 
해당 디렉토리에서 명령어를 수행할때 느리게 실행되는 경우가 있다. 
이를 `Git` 설정을 통해 해결하는 방법에 대해 간단하게 설명한다. 

## 방법
해결하는 방법은 간단하다. 
수행이 느린 이유 중 가장 큰 부분이 `git status` 관련된 부분을 명령어 수행마다 갱신하거나 체크하는 부분이 있기 때문이다. 

## Git Bash
특정 저장소 디렉토리에만 적용하려면 아래와 같이 `Git config` 를 설정한다. 

```bash
$ git config --add bash-it.hide-status 1

$ git config --add bash-it.hide-dirty 1
```  

모든 저장소 디렉토리에 적용하려면 아래와 같이 `--global` 을 추가로 입력한다. 

```bash
$ git config --global --add bash-it.hide-status 1

$ git config --global --add bash-it.hide-dirty 1
```  

## Zsh(Oh My Zsh)
설정하는 방법은 각각 아래와 같다. 

```bash
$ git config --add oh-my-zsh.hide-status 1

$ git config --add oh-my-zsh.hide-dirty 1
```  

```bash
$ git config --global --add oh-my-zsh.hide-status 1

$ git config --global --add oh-my-zsh.hide-dirty 1
```  


---
 
## Reference



