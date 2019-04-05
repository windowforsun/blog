--- 
layout: single
classes: wide
title: "[Linux 실습] Git 설치하기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Linux 에 Git 를 설치하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Git
---  

## 환경
- CentOS 6
- Git 1.7

## 설치 확인

```
[root@windowforsun ~]# git
bash: git: command not found
```  

## 설치

```
[root@windowforsun ~]# yum install git

생략 ..

Dependencies Resolved

================================================================================
 Package            Arch           Version                   Repository    Size
================================================================================
Installing:
 git                x86_64         1.7.1-9.el6_9             base         4.6 M
Installing for dependencies:
 perl-Error         noarch         1:0.17015-4.el6           base          29 k
 perl-Git           noarch         1.7.1-9.el6_9             base          29 k

Transaction Summary
================================================================================
Install       3 Package(s)

생략 ..

=====================================================================================================================
 Package                     Arch                    Version                             Repository             Size
=====================================================================================================================
Installing:
 git                         x86_64                  1.7.1-9.el6_9                       base                  4.6 M
Installing for dependencies:
 perl-Error                  noarch                  1:0.17015-4.el6                     base                   29 k
 perl-Git                    noarch                  1.7.1-9.el6_9                       base                   29 k

Transaction Summary
=====================================================================================================================
Install       3 Package(s)

Total download size: 4.7 M
Installed size: 15 M
Is this ok [y/N]: y
Downloading Packages:
(1/3): git-1.7.1-9.el6_9.x86_64.rpm                                                           | 4.6 MB     00:00
(2/3): perl-Error-0.17015-4.el6.noarch.rpm                                                    |  29 kB     00:00
(3/3): perl-Git-1.7.1-9.el6_9.noarch.rpm                                                      |  29 kB     00:00
---------------------------------------------------------------------------------------------------------------------
Total                                                                                 13 MB/s | 4.7 MB     00:00
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : 1:perl-Error-0.17015-4.el6.noarch                                                                 1/3
  Installing : git-1.7.1-9.el6_9.x86_64                                                                          2/3
  Installing : perl-Git-1.7.1-9.el6_9.noarch                                                                     3/3
  Verifying  : 1:perl-Error-0.17015-4.el6.noarch                                                                 1/3
  Verifying  : git-1.7.1-9.el6_9.x86_64                                                                          2/3
  Verifying  : perl-Git-1.7.1-9.el6_9.noarch                                                                     3/3

Installed:
  git.x86_64 0:1.7.1-9.el6_9

Dependency Installed:
  perl-Error.noarch 1:0.17015-4.el6                          perl-Git.noarch 0:1.7.1-9.el6_9

Complete!

```  

## 설치 확인

```
[root@windowforsun ~]# git --version
git version 1.7.1
```  

---
## Reference
[CentOS git 설치](https://zetawiki.com/wiki/CentOS_git_%EC%84%A4%EC%B9%98)  
