--- 
layout: single
classes: wide
title: "[Git] Git 저장소를 SSH 를 사용해서 접근하자"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'SSH 를 사용해서 Git 저장소에 아이디, 비밀번호 없이 접근하자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - SSH
---  

## Linux 환경에서 SSH 키 설정
- CentOS 7

### 사전 확인
- `ll ~/.ssh/` 명령어로 생성된 SSH 키가 있는지 확인한다.
	- 리스트에 `id_rsa`, `id_rsa.pub` 이 없다면 다시 생성한다.

	```
	$ ll ~/.ssh/
	```  
	
- Private 저장소인 `git-test` 저장소를 클론을 받아 본다.

	```
	$ git clone https://github.com/windowforsun/git-test.git
	Cloning into 'git-test'...
	Username for 'https://github.com': ^C
	[windowforsun_com@ssd .ssh]$ git clone git@github.com:windowforsun/git-test.git
	Cloning into 'git-test'...
	The authenticity of host 'github.com (192.30.253.113)' can't be established.
	RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
	RSA key fingerprint is MD5:16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
	Are you sure you want to continue connecting (yes/no)? yes
	Warning: Permanently added 'github.com,192.30.253.113' (RSA) to the list of known hosts.
	Permission denied (publickey).
	fatal: Could not read from remote repository.
	
	Please make sure you have the correct access rights
	and the repository exists.
	```  
	
	- 권한 문제로 클론을 받을 수 없다.
	
### SSH 만들기
- `ssh-keygen -t rsa -b 4096 -C "my@email.com"` 명렁어를 사용해서 SSH 키를 생성한다.

	```
	$ ssh-keygen -t rsa -b 4096 -C "my@email.com"
	Generating public/private rsa key pair.
	Enter file in which to save the key (/home/windowforsun_com/.ssh/id_rsa): # 별도의 경로에 저장을 원할 경우 경로 입력
	Enter passphrase (empty for no passphrase): # SSH 비밀번호를 설정할 경우 입력
	Enter same passphrase again:
	Your identification has been saved in /home/windowforsun_com/.ssh/id_rsa.
	Your public key has been saved in /home/windowforsun_com/.ssh/id_rsa.pub.
	The key fingerprint is:
	SHA256:ol6A0h4H0/37zLbPFdaL4+rxi1z5Z8rACC63DCuaFXg my@email.com
	The key's randomart image is:
	+---[RSA 4096]----+
	|                 |
	|   . .           |
	|  o . .          |
	| . =   .       . |
	|. = E . S     o .|
	| o + + o o o .o..|
	|  . o + + ..o=.. |
	|   +.. * =o.*+o o|
	|  o.... o.*B++++.|
	+----[SHA256]-----+
	```  
	
- 다시 `ll ~/.ssh/` 명렁어로 확인해보면 아래와 같은 결과를 확인 할 수 있다.

	```
	$ ll ~/.ssh/
	total 12
	-rw-------. 1 windowforsun_com windowforsun_com 3243 Jan 13 02:53 id_rsa
	-rw-r--r--. 1 windowforsun_com windowforsun_com  738 Jan 13 02:53 id_rsa.pub
	```  
	
	- `id_rsa` 는 생성된 비밀키이다.
	- `id_rsa.pub` 는 생성된 공개키이다.

### SSH 키 사용하기
- 생성된 SSH 비밀키를 SSH-Agent 에 등록해야 사용가능하다.
- `eval "$(ssh-agent -s)"` 명렁어로 SSH-Agent 를 실행 시킨다.

	```
	$ eval "$(ssh-agent -s)"
	Agent pid 1858

	$ eval `ssh-agent -s` # 동일한 동작을 이 명령어로도 가능하다.
	```  
	
- `ssh-add ~/.ssh/id_rsa` 명렁어로 SSH-Agent 에 생성한 비밀키를 등록 시킨다.

	```
	$ ssh-add ~/.ssh/id_rsa
	Identity added: /home/windowforsun_com/.ssh/id_rsa (/home/windowforsun_com/.ssh/id_rsa)
	```  
	
- 이후 저장소에 공개키를 등록하기 위해 `cat ~/.ssh/id_rsa.pub` 명령어로 출력되는 공개키를 복사한다.

	```
	[windowforsun_com@ssd ~]$ cat ~/.ssh/id_rsa.pub
	ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQD1C8QNUMbkmDe3bP07lGPuRWTtwW14ddwnblho3WZMBkX9BntjYCbhSLK2cjvShZb02J0z5x5nGPVSXIbuc589e+RIjB/Tari3okIVKWBFZ4CkN91qR17F6hGMfwkITT1u1U6LAOdYRjsoc+REaxNRtbb8RVv9ithYJG417eynGHpXCkEtD/SYDSPhTuB+fDQmTRmr/37xwYBdNjihmpoiQWW/SjM2eABcMSQBHDHq2CCqYxtDSiAS3KnkA7einBglDaLzHUhuV7/tQyAZ40M6ZSmqTV155wJx/+zbLQYkwQQ5IRB0ZLoyft/fNdOdVZcZ+/ybVuXi2RmdeG+ftcbF+anKXGfhtUM30/Gl/YWqILj13IAb/8jVj4oMfGkFARKDmV09H2pmTbtF1/SIwpy1AJvRTr36Xt0MyLO9bfTva41tfoHBmn/INXgHjwMwHykvyAuSimFbPCXefjD8Y2gNQhEOgDEIkBSJUUMzfv8tS6HU0CC6gTa+/yVbg4r97humwIpa7hAJlT4M1yimW10Kk2JPZRMk7tJVasmfWMV4yhbhP4eQFzCGm2euQSR6FNSR24FggrwB569ajsiMoMXTUliikvhW/gxcef4FncGCrKi9a3LEhFx7nwvzeoGhXhMfd9vEeyA+Z9M17vNJNW0odxrWB3xp/wq9sc+njA8eHw== my@email.com
	```  

### SSH 키에 도메인 명시가 직접적으로 필요한 경우

```
$ touch ~/.ssh/known_hosts
$ ssh-keyscan github.com >> ~/.ssh/known_hosts
```  

## 저장소에 공개키 등록하기

- Github 설정페이지로 이동한다.

	![그림 1]({{site.baseurl}}/img/git/practice-ssh-1.png)
	
- 좌측리스트에서 `SSH and GPG keys` 를 누른다.

	![그림 1]({{site.baseurl}}/img/git/practice-ssh-2.png)
	
- SSH keys 부분에서 `New SSH key` 를 누른다

	![그림 1]({{site.baseurl}}/img/git/practice-ssh-3.png)

- `Title` 에는 등록하려는 공개키의 제목, `Key` 에는 복사했던 공개키를 붙여넣기 해준다.

	![그림 1]({{site.baseurl}}/img/git/practice-ssh-4.png)

- 이제 SSH 를 사용해서 저장소 접근을 할때는 `https://github.com/<계정명>/<저장소명>` 형식의 URL 을 사용하지 않고, 아래와 같은 URL을 사용한다.
	
	![그림 1]({{site.baseurl}}/img/git/practice-ssh-5.png)


## Linux 에서 SSH 를 이용해서 접근 확인하기
- 등록된 공개키에 해당하는 비밀키가 있는 Linux 에서 수행해야 한다.
- 다시 `git clone <저장소URL>` 을 통해 저장소를 클론 받아 본다.

	```
	$ git clone git@github.com:windowforsun/git-test.git
	Cloning into 'git-test'...
	Warning: Permanently added the RSA host key for IP address '140.82.113.4' to the list of known hosts.
	remote: Enumerating objects: 11, done.
	remote: Counting objects: 100% (11/11), done.
	remote: Compressing objects: 100% (10/10), done.
	remote: Total 15 (delta 3), reused 8 (delta 0), pack-reused 4
	Receiving objects: 100% (15/15), done.
	Resolving deltas: 100% (3/3), done.
	```  
	
- `ssh -T git@github.com` 으로도 테스트가 가능하다.

	```
	$ ssh -T git@github.com
	Hi windowforsun! You've successfully authenticated, but GitHub does not provide shell access.
	```  

---
 
## Reference



