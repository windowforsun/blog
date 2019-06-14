--- 
layout: single
classes: wide
title: "[Linux 실습] Ruby 설치"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Ruby 를 설치해 보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Ruby
  - RVM
---  

## 환경
- CentOS 6
- RVM
- Ruby 2.6 

## RVM / Ruby 설치

- tmuxinator 설치를 위해서는 ruby 2.4 이상 버전이 설치되어야 한다.
- Centos 에서 안전하게 2.4 이상의 Ruby 를 설치하기 위해 RVM(Ruby Version Manager) 를 사용한다.

- ruby 설치를 위한 기본적인 패키지 설치

```
[root@windowforsun ~]# yum install gcc-c++ patch readline readline-devel zlib zlib-devel \
>    libyaml-devel libffi-devel openssl-devel make \
>    bzip2 autoconf automake libtool bison iconv-devel sqlite-devel
// 생략
Dependencies Resolved

=====================================================================================================================
 Package                            Arch                  Version                          Repository           Size
=====================================================================================================================
Installing:
 autoconf                           noarch                2.63-5.1.el6                     base                781 k
 automake                           noarch                1.11.1-4.el6                     base                550 k
 bison                              x86_64                2.4.1-5.el6                      base                637 k
 gcc-c++                            x86_64                4.4.7-23.el6                     base                4.7 M
 libffi-devel                       x86_64                3.0.5-3.2.el6                    base                 18 k
 libtool                            x86_64                2.2.6-15.5.el6                   base                564 k
 libyaml-devel                      x86_64                0.1.3-4.el6_6                    base                 85 k
 openssl-devel                      x86_64                1.0.1e-57.el6                    base                1.2 M
 patch                              x86_64                2.6-8.el6_9                      base                 91 k
 readline-devel                     x86_64                6.0-4.el6                        base                134 k
 sqlite-devel                       x86_64                3.6.20-1.el6_7.2                 base                 81 k
 zlib-devel                         x86_64                1.2.3-29.el6                     base                 44 k
Installing for dependencies:
 keyutils-libs-devel                x86_64                1.4-5.el6                        base                 29 k
 krb5-devel                         x86_64                1.10.3-65.el6                    base                504 k
 libcom_err-devel                   x86_64                1.41.12-24.el6                   base                 33 k
 libkadm5                           x86_64                1.10.3-65.el6                    base                143 k
 libselinux-devel                   x86_64                2.0.94-7.el6                     base                137 k
 libsepol-devel                     x86_64                2.0.41-4.el6                     base                 64 k
 libstdc++-devel                    x86_64                4.4.7-23.el6                     base                1.6 M

Transaction Summary
=====================================================================================================================
Install      19 Package(s)

Total download size: 11 M
Installed size: 35 M
Is this ok [y/N]: y
// 생략
Installed:
  autoconf.noarch 0:2.63-5.1.el6         automake.noarch 0:1.11.1-4.el6           bison.x86_64 0:2.4.1-5.el6
  gcc-c++.x86_64 0:4.4.7-23.el6          libffi-devel.x86_64 0:3.0.5-3.2.el6      libtool.x86_64 0:2.2.6-15.5.el6
  libyaml-devel.x86_64 0:0.1.3-4.el6_6   openssl-devel.x86_64 0:1.0.1e-57.el6     patch.x86_64 0:2.6-8.el6_9
  readline-devel.x86_64 0:6.0-4.el6      sqlite-devel.x86_64 0:3.6.20-1.el6_7.2   zlib-devel.x86_64 0:1.2.3-29.el6

Dependency Installed:
  keyutils-libs-devel.x86_64 0:1.4-5.el6                      krb5-devel.x86_64 0:1.10.3-65.el6
  libcom_err-devel.x86_64 0:1.41.12-24.el6                    libkadm5.x86_64 0:1.10.3-65.el6
  libselinux-devel.x86_64 0:2.0.94-7.el6                      libsepol-devel.x86_64 0:2.0.41-4.el6
  libstdc++-devel.x86_64 0:4.4.7-23.el6

Complete!
```  

- rvm 설치

```
[root@windowforsun ~]# curl -sSL https://rvm.io/mpapis.asc | gpg2 --import -
gpg: directory `/root/.gnupg' created
gpg: new configuration file `/root/.gnupg/gpg.conf' created
gpg: WARNING: options in `/root/.gnupg/gpg.conf' are not yet active during this run
gpg: keyring `/root/.gnupg/secring.gpg' created
gpg: keyring `/root/.gnupg/pubring.gpg' created
gpg: /root/.gnupg/trustdb.gpg: trustdb created
gpg: key D39DC0E3: public key "Michal Papis (RVM signing) <mpapis@gmail.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
gpg: no ultimately trusted keys found


[root@windowforsun ~]# curl -sSL https://rvm.io/pkuczynski.asc | gpg2 --import -
gpg: key 39499BDB: public key "Piotr Kuczynski <piotr.kuczynski@gmail.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)


[root@windowforsun ~]# curl -L get.rvm.io | bash -s stable
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 24168  100 24168    0     0  89855      0 --:--:-- --:--:-- --:--:-- 89855
Downloading https://github.com/rvm/rvm/archive/1.29.8.tar.gz
Downloading https://github.com/rvm/rvm/releases/download/1.29.8/1.29.8.tar.gz.asc
gpg: Signature made Wed 08 May 2019 02:14:49 PM UTC using RSA key ID 39499BDB
gpg: Good signature from "Piotr Kuczynski <piotr.kuczynski@gmail.com>"
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 7D2B AF1C F37B 13E2 069D  6956 105B D0E7 3949 9BDB
GPG verified '/usr/local/rvm/archives/rvm-1.29.8.tgz'
Creating group 'rvm'
Installing RVM to /usr/local/rvm/
Installation of RVM in /usr/local/rvm/ is almost complete:

  * First you need to add all users that will be using rvm to 'rvm' group,
    and logout - login again, anyone using rvm will be operating with `umask u=rwx,g=rwx,o=rx`.

  * To start using RVM you need to run `source /etc/profile.d/rvm.sh`
    in all your open shell windows, in rare cases you need to reopen all shell windows.
  * Please do NOT forget to add your users to the rvm group.
     The installer no longer auto-adds root or users to the rvm group. Admins must do this.
     Also, please note that group memberships are ONLY evaluated at login time.
     This means that users must log out then back in before group membership takes effect!
Thanks for installing RVM 🙏
Please consider donating to our open collective to help us maintain RVM.

👉  Donate: https://opencollective.com/rvm/donate
```  

- rvm 환경 로드

```
[root@windowforsun ~]# source /etc/profile.d/rvm.sh


[root@windowforsun ~]# rvm reload
RVM reloaded!
```  

- rvm 의존성 확인하기

```
[root@windowforsun ~]# rvm requirements run
Checking requirements for centos.
Requirements installation successful.
```  

- 설치할 수 있는 ruby 버전 확인

```
[root@windowforsun ~]# rvm list known
# MRI Rubies
[ruby-]1.8.6[-p420]
[ruby-]1.8.7[-head] # security released on head
[ruby-]1.9.1[-p431]
[ruby-]1.9.2[-p330]
[ruby-]1.9.3[-p551]
[ruby-]2.0.0[-p648]
[ruby-]2.1[.10]
[ruby-]2.2[.10]
[ruby-]2.3[.8]
[ruby-]2.4[.6]
[ruby-]2.5[.5]
[ruby-]2.6[.3]
```  

- ruby 설치하기

```
[root@windowforsun ~]# rvm install 2.4
Searching for binary rubies, this might take some time.
No binary rubies available for: centos/6/x86_64/ruby-2.4.6.
Continuing with compilation. Please read 'rvm help mount' to get more information on binary rubies.
Checking requirements for centos.
Requirements installation successful.
Installing Ruby from source to: /home/windowforsun_com/.rvm/rubies/ruby-2.4.6, this may take a while depending on your cpu(s)...
ruby-2.4.6 - #downloading ruby-2.4.6, this may take a while depending on your connection...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 12.0M  100 12.0M    0     0  10.3M      0  0:00:01  0:00:01 --:--:-- 21.9M
ruby-2.4.6 - #extracting ruby-2.4.6 to /home/windowforsun_com/.rvm/src/ruby-2.4.6.....
ruby-2.4.6 - #configuring..................................................................
ruby-2.4.6 - #post-configuration..
ruby-2.4.6 - #compiling..............................................................................................
ruby-2.4.6 - #installing..........................
ruby-2.4.6 - #making binaries executable..
ruby-2.4.6 - #downloading rubygems-3.0.4
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  862k  100  862k    0     0  2161k      0 --:--:-- --:--:-- --:--:-- 5198k
No checksum for downloaded archive, recording checksum in user configuration.
ruby-2.4.6 - #extracting rubygems-3.0.4.....
ruby-2.4.6 - #removing old rubygems........
ruby-2.4.6 - #installing rubygems-3.0.4.........................................
ruby-2.4.6 - #gemset created /home/windowforsun_com/.rvm/gems/ruby-2.4.6@global
ruby-2.4.6 - #importing gemset /home/windowforsun_com/.rvm/gemsets/global.gems................................................................
ruby-2.4.6 - #generating global wrappers.......
ruby-2.4.6 - #gemset created /home/windowforsun_com/.rvm/gems/ruby-2.4.6
ruby-2.4.6 - #importing gemsetfile /home/windowforsun_com/.rvm/gemsets/default.gems evaluated to empty gem list
ruby-2.4.6 - #generating default wrappers.......
ruby-2.4.6 - #adjusting #shebangs for (gem irb erb ri rdoc testrb rake).
Install of ruby-2.4.6 - #complete
Ruby was built without documentation, to build it run: rvm docs generate-ri
``` 

- 현재 설치된 ruby 버전 확인

```
[root@windowforsun ~]# rvm list
=* ruby-2.4.6 [ x86_64 ]
   ruby-2.6.3 [ x86_64 ]

# => - current
# =* - current && default
#  * - default
```  

- default ruby 버전 설정

```
[root@windowforsun ~]# rvm use 2.4 --default
Using /usr/local/rvm/gems/ruby-2.4.6
```  

- ruby 버전 확인

```
[root@windowforsun ~]# ruby --version
ruby 2.4.6p354 (2019-04-01 revision 67394) [x86_64-linux]
```  

---
## Reference
[How to Install Ruby on CentOS/RHEL 7/6](https://tecadmin.net/install-ruby-latest-stable-centos/)  
[CentOS6 ruby-2.3 설치](https://zetawiki.com/wiki/CentOS6_ruby-2.3_%EC%84%A4%EC%B9%98)  