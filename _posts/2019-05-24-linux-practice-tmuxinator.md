--- 
layout: single
classes: wide
title: "[Linux 실습] tmux 를 설치하고 사용하자"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'tmux 를 설치, 사용 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - tmux
  - Tmuxinator
---  

## 환경
- CentOS 6
- tmux 2.3
- Ruby 2.4 를 설치하는 걸로 수정하기
- Gem 2.5 ?


## Ruby / Gem 설치

- tmuxinator 설치를 위해서 ruby 2.3 이상 버전으로 설치한다.

- rpm 다운로드

```
[root@windowforsun ~]# wget http://www.torutk.com/attachments/download/589 --content-disposition
--2019-06-13 03:07:52--  http://www.torutk.com/attachments/download/589
Resolving www.torutk.com... 133.242.171.227, 2401:2500:102:1210:133:242:171:227
Connecting to www.torutk.com|133.242.171.227|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [audio/x-pn-realaudio-plugin]
--2019-06-13 03:07:52--  http://www.torutk.com/attachments/download/589
Connecting to www.torutk.com|133.242.171.227|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 11437292 (11M) [audio/x-pn-realaudio-plugin]
Saving to: “ruby-2.3.4-1.el6.x86_64.rpm”

100%[===========================================================================>] 11,437,292  3.79M/s   in 2.9s

2019-06-13 03:07:56 (3.79 MB/s) - “ruby-2.3.4-1.el6.x86_64.rpm” saved [11437292/11437292]

```  

- Ruby 설치

```
[root@windowforsun ~]# yum install ruby-2.3.4-1.el6.x86_64.rpm
Loaded plugins: fastestmirror, security
Setting up Install Process
// 생략
Dependencies Resolved

=====================================================================================================================
 Package               Arch                 Version                     Repository                              Size
=====================================================================================================================
Installing:
 ruby                  x86_64               2.3.4-1.el6                 /ruby-2.3.4-1.el6.x86_64                36 M
Installing for dependencies:
 libyaml               x86_64               0.1.3-4.el6_6               base                                    52 k

Transaction Summary
=====================================================================================================================
Install       2 Package(s)

Total size: 36 M
Total download size: 52 k
Installed size: 36 M
Is this ok [y/N]: y
Downloading Packages:
libyaml-0.1.3-4.el6_6.x86_64.rpm                                                              |  52 kB     00:00
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : libyaml-0.1.3-4.el6_6.x86_64                                                                      1/2
  Installing : ruby-2.3.4-1.el6.x86_64                                                                           2/2
  Verifying  : libyaml-0.1.3-4.el6_6.x86_64                                                                      1/2
  Verifying  : ruby-2.3.4-1.el6.x86_64                                                                           2/2

Installed:
  ruby.x86_64 0:2.3.4-1.el6

Dependency Installed:
  libyaml.x86_64 0:0.1.3-4.el6_6

Complete!
```  

- 설치 버전 확인

```
[root@windowforsun ~]# ruby -v
ruby 2.3.4p301 (2017-03-30 revision 58214) [x86_64-linux-gnu]
[root@windowforsun ~]# gem -v
2.5.2
```  

## Tmuxinator 설치

- 

---
## Reference
[CentOS6 ruby-2.3 설치](https://zetawiki.com/wiki/CentOS6_ruby-2.3_%EC%84%A4%EC%B9%98)  