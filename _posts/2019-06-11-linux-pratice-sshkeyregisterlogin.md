--- 
layout: single
classes: wide
title: "[Linux 실습] SSH 접속 시 SSH 키 등록 및 자동 로그인"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: '원격 서버에 SSH 로 접속할 때 자동 로그인으로 사용할 SSH 키를 등록하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - SSH
  - SSH Key
---  

## 환경
- CentOS 6

## SSH Key 란
- 서버에 접속 할 때 비밀번호 대신 Key 를 제출하는 방식이다.
- 비밀번호 보다 높은 수준의 보안을 필요로 할 때 또는 로그인 없이 자동으로 서버에 접속 할 때 사용한다.

## SSH Key 의 동작 방식
- SSH Key 는 공개키 암호 알고리즘 RSA 등 과 같은 방식으로 인증을 수행한다.
- 공개키(public key), 비밀키(private key) 로 이루어져 있다.
- SSH Key 를 생성하게 되면 위 2개의 키가 모두 생성되고, 공개키 암호 알고리즘의 방식에 따리 공개키는 외부로 공개하고 비밀키는 자신만 가지고 있는다.
- 공개키는 접속할 원격 서버에 등록하게 되고, 비밀키는 키를 만든 접속 시 사용할 머신에 두고 원격 서버에 접속시에 인증으로 사용한다.

## 접속시 사용할 서버에서 SSH 키(RSA) 생성

- ssh-keygen 명령어를 통해 SSH Key 를 생성한다.

```
[windowforsun@localhost ~]$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/windowforsun/.ssh/id_rsa): # 키가 생성될 위치
Created directory '/home/windowforsun/.ssh'.
Enter passphrase (empty for no passphrase): # ssh key 를 통해 로그인할 때 사용할 비밀번호
Enter same passphrase again: # 위 비밀번호 확인
Your identification has been saved in /home/windowforsun/.ssh/id_rsa.
Your public key has been saved in /home/windowforsun/.ssh/id_rsa.pub.
The key fingerprint is:
4d:3e:c4:77:69:1c:64:ea:fd:76:3b:40:87:46:1c:c6 windowforsun@localhost.localdomain
The key's randomart image is:
+--[ RSA 2048]----+
|            o+=  |
|         .  .E o |
|          + + *  |
|         = o B . |
|        S + + o  |
|           . . . |
|              . +|
|               oo|
|               ..|
+-----------------+
```  

- 키 경로 입력에서 아무것도 입력하지 않았으면 `/home/<유저계정명>/.ssh/` 에 public key 와 private key 가 생성된다.

```
[windowforsun@localhost ~]$ ll /home/windowforsun/.ssh/
total 8
-rw------- 1 windowforsun windowforsun 1743 Jun 10 18:27 id_rsa
-rw-r--r-- 1 windowforsun windowforsun  416 Jun 10 18:27 id_rsa.pub
```  

- public key 에 해당되는 `id_rsa.pub` 을 `authorized_keys` 로 복사 해준다.

```
[windowforsun@localhost ~]$ cp /home/windowforsun/.ssh/id_rsa.pub /home/windowforsun/.ssh/authorized_keys
[windowforsun@localhost ~]$ ll /home/windowforsun/.ssh/
total 12
-rw-r--r-- 1 windowforsun windowforsun  416 Jun 10 18:33 authorized_keys
-rw------- 1 windowforsun windowforsun 1743 Jun 10 18:27 id_rsa
-rw-r--r-- 1 windowforsun windowforsun  416 Jun 10 18:27 id_rsa.pub
```  

## 접속 할 원격 서버에 SSH 키 등록하기
- 접속 할 원격 서버에 등록 시키는 키는 public key 로 `id_rsa.pub` 파일의 내용이다.

### `scp` FTP 명령어를 사용해서 키 등록

```
[windowforsun@localhost ~]$ scp /home/windowforsun/.ssh/id_rsa.pub windowforsun2@remoteserver:/home/windowforsun2/.ssh/authorized_keys
windowforsun2@remoteserver's password: # 접속 서버 계정의 암호
id_rsa.pub                                          100%  233     0.2KB/s   00:00  
```  

- 현재 접속으로 사용할 계정인 `windowforsun` 의 공개키를 remoteserver 의 `windowforsun2` 의 `authorized_keys` 에 복사해 키를 등록한다.

### 원격 서버에 직접 접속해서 키 등록

- 접속 시 사용할 `windowforsun` 계정의 public key `id_rsa.pub` 의 내용을 복사한다.

```
[windowforsun@localhost ~]$ vi /home/windowforsun/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAsEc+lci3mHqjbY11nDAJFp+FP4aSRfj7c6NZ46RxiwWdDEtIMjg4cUkiapHVM9LipNxI0QNCUWzVYvP0tlOlvUXPa/yuW7CfLpH6SOYsZqhieK/3b6poKu+ZtmKiUbgC3xDH3E3ReUhrIP9EFXDfvATbI4vxLiyWW9LOHQ2oQv+t4m1SL6NJjGrHZSWQhA3Dy94LgtLW+1dnLwJCvB+bOUt0BIe62hNGwWgHv9EY8nTUtMb14Cs+PSvLIm4tvqlz+ptLH6PK/Aytfg9Z7oDA3mNQbR6nXqjE+tDoA1LblphStpPhnPBjzLuQzURK5AKeY0FXAZqfe5eRN1h7xKAtTw== windowforsun@localhost.localdomain
```  

- 접속할 원격 서버에 접속한다.

```
[windowforsun@localhost ~]$ ssh windowforsun2@remoteserver
windowforsun2@remoteserver's password: # 접속 서버 계정의 암호
```  

- 원격 서버 계정의 `/hom/<원격서버계정명>/.ssh/authorized_keys` 에 복사한 public key 를 넣어 준다.

```
[windowforsun2@localhost ~]$ vi /home/windowforsun2/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAsEc+lci3mHqjbY11nDAJFp+FP4aSRfj7c6NZ46RxiwWdDEtIMjg4cUkiapHVM9LipNxI0QNCUWzVYvP0tlOlvUXPa/yuW7CfLpH6SOYsZqhieK/3b6poKu+ZtmKiUbgC3xDH3E3ReUhrIP9EFXDfvATbI4vxLiyWW9LOHQ2oQv+t4m1SL6NJjGrHZSWQhA3Dy94LgtLW+1dnLwJCvB+bOUt0BIe62hNGwWgHv9EY8nTUtMb14Cs+PSvLIm4tvqlz+ptLH6PK/Aytfg9Z7oDA3mNQbR6nXqjE+tDoA1LblphStpPhnPBjzLuQzURK5AKeY0FXAZqfe5eRN1h7xKAtTw== windowforsun@localhost.localdomain
```  

## 원격 접속 확인 하기

- 등록된 원격 서버에 접속하고 키 생성시 입력했던 비밀번호가 있다면 입력해 준다.

```
[windowforsun2@localhost ~]$ ssh windowforsun@remoteserver
Enter passphrase for key '/home/windowforsun/.ssh/id_rsa':
```  


---
## Reference
[SSH 접속시 사용할 SSH 키 생성 및 설정](http://www.fun25.co.kr/blog/ssh-key-setup)  
[Linux - ssh-keygen 을 이용한 ssh 공개키 생성및 비밀번호 없이 로그인](http://develop.sunshiny.co.kr/863)  
[SSH 인증키 생성 및 서버 등록](http://www.omani.pe.kr/?p=789)  