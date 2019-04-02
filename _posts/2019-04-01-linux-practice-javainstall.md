--- 
layout: single
classes: wide
title: "[Linux 실습] Java(jdk) 설치하기"
header:
  overlay_image: /img/linux-bg.png
excerpt: 'Linux 에 Java 를 설치하자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Java
---  

## 환경
- CentOS 6
- Java 8(jdk 1.8)

## jdk 설치 확인

```
[root@windowforsun ~]# java -version
bash: java: command not found
```  

- `command not found` 라는 결과를 통해 설치 되지 않았음을 확인할 수 있다.

```
[root@windowforsun ~]# yum list java*jdk-devel
Loaded plugins: fastestmirror, security
Loading mirror speeds from cached hostfile
 * base: centos4.zswap.net
 * epel: mirror.us.leaseweb.net
 * extras: centos4.zswap.net
 * updates: centos4.zswap.net
Available Packages
java-1.6.0-openjdk-devel.x86_64               1:1.6.0.41-1.13.13.1.el6_8                 base
java-1.7.0-openjdk-devel.x86_64               1:1.7.0.211-2.6.17.1.el6_10                updates
java-1.8.0-openjdk-devel.x86_64               1:1.8.0.201.b09-2.el6_10                   updates
```  

- 리스트에서 java-1.8.0-openjdk-devel 패키지를 확인 할 수 있다.

## jdk 설치

```
[root@windowforsun ~]# yum install java-1.8.0-openjdk-devel.x86_64
생략 ..
Dependencies Resolved

================================================================================================
 Package                          Arch        Version                        Repository    Size
================================================================================================
Installing:
 java-1.8.0-openjdk-devel         x86_64      1:1.8.0.201.b09-2.el6_10       updates       10 M
Installing for dependencies:
 alsa-lib                         x86_64      1.1.0-4.el6                    base         389 k
 atk                              x86_64      1.30.0-1.el6                   base         195 k
 avahi-libs                       x86_64      0.6.25-17.el6                  base          55 k
 cairo                            x86_64      1.8.8-6.el6_6                  base         309 k
 cups-libs                        x86_64      1:1.4.2-80.el6_10              updates      323 k
 fontconfig                       x86_64      2.8.0-5.el6                    base         186 k
 freetype                         x86_64      2.3.11-17.el6                  base         361 k
 giflib                           x86_64      4.1.6-3.1.el6                  base          37 k
 gnutls                           x86_64      2.12.23-22.el6                 base         389 k
 gtk2                             x86_64      2.24.23-9.el6                  base         3.2 M
 hicolor-icon-theme               noarch      0.11-1.1.el6                   base          40 k
 java-1.8.0-openjdk               x86_64      1:1.8.0.201.b09-2.el6_10       updates      216 k
 java-1.8.0-openjdk-headless      x86_64      1:1.8.0.201.b09-2.el6_10       updates       32 M
 jpackage-utils                   noarch      1.7.5-3.16.el6                 base          60 k
 libICE                           x86_64      1.0.6-1.el6                    base          53 k
 libSM                            x86_64      1.2.1-2.el6                    base          37 k
 libXcomposite                    x86_64      0.4.3-4.el6                    base          20 k
 libXcursor                       x86_64      1.1.14-2.1.el6                 base          28 k
 libXdamage                       x86_64      1.1.3-4.el6                    base          18 k
 libXext                          x86_64      1.3.3-1.el6                    base          35 k
 libXfixes                        x86_64      5.0.3-1.el6                    base          17 k
 libXfont                         x86_64      1.5.1-2.el6                    base         145 k
 libXft                           x86_64      2.3.2-1.el6                    base          55 k
 libXi                            x86_64      1.7.8-1.el6                    base          38 k
 libXinerama                      x86_64      1.1.3-2.1.el6                  base          13 k
 libXrandr                        x86_64      1.5.1-1.el6                    base          25 k
 libXrender                       x86_64      0.9.10-1.el6                   base          24 k
 libXtst                          x86_64      1.2.3-1.el6                    base          19 k
 libfontenc                       x86_64      1.1.2-3.el6                    base          29 k
 libthai                          x86_64      0.1.12-3.el6                   base         183 k
 lksctp-tools                     x86_64      1.0.10-7.el6                   base          79 k
 pango                            x86_64      1.28.1-11.el6                  base         351 k
 pcsc-lite-libs                   x86_64      1.5.2-16.el6                   base          28 k
 pixman                           x86_64      0.32.8-1.el6                   base         243 k
 ttmkfdir                         x86_64      3.0.9-32.1.el6                 base          43 k
 tzdata-java                      noarch      2019a-1.el6                    updates      188 k
 xorg-x11-font-utils              x86_64      1:7.2-11.el6                   base          75 k
 xorg-x11-fonts-Type1             noarch      7.2-11.el6                     base         520 k

생략 ..

Installed:
  java-1.8.0-openjdk-devel.x86_64 1:1.8.0.201.b09-2.el6_10

Dependency Installed:
  alsa-lib.x86_64 0:1.1.0-4.el6
  atk.x86_64 0:1.30.0-1.el6
  avahi-libs.x86_64 0:0.6.25-17.el6
  cairo.x86_64 0:1.8.8-6.el6_6
  cups-libs.x86_64 1:1.4.2-80.el6_10
  fontconfig.x86_64 0:2.8.0-5.el6
  freetype.x86_64 0:2.3.11-17.el6
  giflib.x86_64 0:4.1.6-3.1.el6
  gnutls.x86_64 0:2.12.23-22.el6
  gtk2.x86_64 0:2.24.23-9.el6
  hicolor-icon-theme.noarch 0:0.11-1.1.el6
  java-1.8.0-openjdk.x86_64 1:1.8.0.201.b09-2.el6_10
  java-1.8.0-openjdk-headless.x86_64 1:1.8.0.201.b09-2.el6_10
  생략 ..

Complete!
```  

## jdk 설치 확인

```
[root@windowforsun ~]# javac -version
javac 1.8.0_201
```  

---
## Reference
[CentOS JDK 설치](https://zetawiki.com/wiki/CentOS_JDK_%EC%84%A4%EC%B9%98)  
