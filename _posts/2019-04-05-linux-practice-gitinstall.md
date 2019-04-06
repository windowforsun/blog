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
- Git 2.18

## 설치 확인

```
[root@windowforsun ~]# git
bash: git: command not found
```  

## 설치

```
[root@windowforsun ~]# rpm -Uvh http://opensource.wandisco.com/centos/6/git/x86_64/wandisco-git-release-6-1.noarch.rpm
Retrieving http://opensource.wandisco.com/centos/6/git/x86_64/wandisco-git-release-6-1.noarch.rpm
warning: /var/tmp/rpm-tmp.siwoyW: Header V4 DSA/SHA1 Signature, key ID 3bbf077a: NOKEY
Preparing...                ########################################### [100%]
   1:wandisco-git-release   ########################################### [100%]
```  

```
[root@windowforsun ~]# yum install git
Loaded plugins: fastestmirror, security
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: mirror.ash.fastserv.com
 * centos-sclo-rh: mirror.ash.fastserv.com
 * centos-sclo-sclo: mirror.ash.fastserv.com
 * epel: mirror.rackspace.com
 * extras: mirror.ash.fastserv.com
 * remi-safe: repo1.ash.innoscale.net
 * updates: mirror.ash.fastserv.com
Resolving Dependencies
--> Running transaction check
---> Package git.x86_64 0:2.18.0-1.WANdisco.402 will be installed
--> Processing Dependency: perl-Git = 2.18.0-1.WANdisco.402 for package: git-2.18.0-1.WANdisco.402.x86_64
--> Processing Dependency: perl(Git) for package: git-2.18.0-1.WANdisco.402.x86_64
--> Processing Dependency: perl(Git::I18N) for package: git-2.18.0-1.WANdisco.402.x86_64
--> Running transaction check
---> Package perl-Git.noarch 0:2.18.0-1.WANdisco.402 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=====================================================================================================================
 Package                Arch                 Version                                Repository                  Size
=====================================================================================================================
Installing:
 git                    x86_64               2.18.0-1.WANdisco.402                  WANdisco-git                12 M
Installing for dependencies:
 perl-Git               noarch               2.18.0-1.WANdisco.402                  WANdisco-git                23 k

Transaction Summary
=====================================================================================================================
Install       2 Package(s)

Total download size: 12 M
Installed size: 37 M
Is this ok [y/N]: y
Downloading Packages:
(1/2): git-2.18.0-1.WANdisco.402.x86_64.rpm                                                   |  12 MB     00:00
(2/2): perl-Git-2.18.0-1.WANdisco.402.noarch.rpm                                              |  23 kB     00:00
---------------------------------------------------------------------------------------------------------------------
Total                                                                                 19 MB/s |  12 MB     00:00
warning: rpmts_HdrFromFdno: Header V4 DSA/SHA1 Signature, key ID 3bbf077a: NOKEY
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-WANdisco
Importing GPG key 0x3BBF077A:
 Userid: "WANdisco (http://WANdisco.com - We Make Software Happen...) <software-key@wandisco.com>"
 From  : /etc/pki/rpm-gpg/RPM-GPG-KEY-WANdisco
Is this ok [y/N]: y
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : git-2.18.0-1.WANdisco.402.x86_64                                                                  1/2
  Installing : perl-Git-2.18.0-1.WANdisco.402.noarch                                                             2/2
  Verifying  : perl-Git-2.18.0-1.WANdisco.402.noarch                                                             1/2
  Verifying  : git-2.18.0-1.WANdisco.402.x86_64                                                                  2/2

Installed:
  git.x86_64 0:2.18.0-1.WANdisco.402

Dependency Installed:
  perl-Git.noarch 0:2.18.0-1.WANdisco.402

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
[RHEL/CentOS 에 git 2 설치하기](https://www.lesstif.com/pages/viewpage.action?pageId=14745759)  
