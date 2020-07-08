--- 
layout: single
classes: wide
title: "[Linux 실습] Bash 결과 변수에 담기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Bash 명령어 실행 결과를 변수에 담아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Practice
  - Bash
---  

## 환경
- CentOS 7

## 결과 전체 담기

### 변수명=$(명령어)

```
$ pwd
/home/vagrant
$ CUR_PATH_2=$(pwd)
$ echo $CUR_PATH_2
/home/vagrant
```  

```
$ git rev-parse HEAD
eaec89b9bb0345821d1b362074d1b99651f9dec0
$ CUR_REVISION_2=$(git rev-parse HEAD)
$ echo $CUR_REVISION_2
eaec89b9bb0345821d1b362074d1b99651f9dec0
```  

### 변수명=`명령어`

```
$ pwd                             
/home/vagrant                     
$ CUR_PATH=`pwd`                  
$ echo $CUR_PATH                  
/home/vagrant    
```  

```
$ git rev-parse HEAD
eaec89b9bb0345821d1b362074d1b99651f9dec0
$ CUR_REVISION=`git rev-parse HEAD`
$ echo $CUR_REVISION
eaec89b9bb0345821d1b362074d1b99651f9dec0
```  

```
$ df
Filesystem       1K-blocks      Used Available Use% Mounted on
/dev/sda1         41921540  29813304  12108236  72% /
devtmpfs           4919788         0   4919788   0% /dev
tmpfs              4927228         0   4927228   0% /dev/shm
tmpfs              4927228     41692   4885536   1% /run
tmpfs              4927228         0   4927228   0% /sys/fs/cgroup
tmpfs               985448         0    985448   0% /run/user/1000
vagrant_server   480378876 417405964  62972912  87% /vagrant/server
vagrant_gitrepos 480378876 417405964  62972912  87% /vagrant/gitrepos
overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/236a03f55a2489dca2c749ddd2cd1fe71fe20786c9baf6012d9bbd3874ece4d8/merged
overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/6bf739da37cde03d1ae3c0e8111cde755a38684cb3f1407f3747bcfc47aacbaa/merged
shm                  65536         0     65536   0% /var/lib/docker/containers/f3680a9416703e5cdcf900ffd4dad6b35793ed05f1d2fe6b179fcf8e6c0f3ecd/mounts/shm
overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/f5c0059b89fb03f4dd20ab2ddfec3705ca47f5803e48b3e4ce4d46177b916848/merged
shm                  65536         0     65536   0% /var/lib/docker/containers/51d9f6225a7a4a67a0bf7388440267015235c492a558ae02ba9faa99a0a5e576/mounts/shm
$ CUR_DF=`df`
$ echo $CUR_DF
Filesystem 1K-blocks Used Available Use% Mounted on /dev/sda1 41921540 29813324 12108216 72% / devtmpfs 4919788 0 4919788 0% /dev tmpfs 4927228 0 4927228 0% /dev/shm tmpfs 4927228 41692 4885536 1% /run tmpfs 4927228 0 4927228 0% /sys/fs/cgroup tmpfs 985448 0 985448 0% /run/user/1000 vagrant_server 480378876 417406860 62972016 87% /vagrant/server vagrant_gitrepos 480378876 417406860 62972016 87% /vagrant/gitrepos overlay 41921540 29813324 12108216 72% /var/lib/docker/overlay2/236a03f55a2489dca2c749ddd2cd1fe71fe20786c9baf6012d9bbd3874ece4d8/merged overlay 41921540 29813324 12108216 72% /var/lib/docker/overlay2/6bf739da37cde03d1ae3c0e8111cde755a38684cb3f1407f3747bcfc47aacbaa/merged shm 65536 0 65536 0% /var/lib/docker/containers/f3680a9416703e5cdcf900ffd4dad6b35793ed05f1d2fe6b179fcf8e6c0f3ecd/mounts/shm overlay 41921540 29813324 12108216 72% /var/lib/docker/overlay2/f5c0059b89fb03f4dd20ab2ddfec3705ca47f5803e48b3e4ce4d46177b916848/merged shm 65536 0 65536 0% /var/lib/docker/containers/51d9f6225a7a4a67a0bf7388440267015235c492a558ae02ba9faa99a0a5e576/mounts/shm
```  

- 위와 같이 여러 줄의 명령어 결과를 변수에 담으면 한줄로 출력 된다.


## 결과 배열에 담기
- 아래 명령어를 사용하면 여러 줄의 명령어 결과를 배열 변수에 담을 수 있다.

	```
	IFS=$'\n' 배열변수명=(`명령어`)
	```  
	
```
$ df
Filesystem       1K-blocks      Used Available Use% Mounted on
/dev/sda1         41921540  29813304  12108236  72% /
devtmpfs           4919788         0   4919788   0% /dev
tmpfs              4927228         0   4927228   0% /dev/shm
tmpfs              4927228     41692   4885536   1% /run
tmpfs              4927228         0   4927228   0% /sys/fs/cgroup
tmpfs               985448         0    985448   0% /run/user/1000
vagrant_server   480378876 417410752  62968124  87% /vagrant/server
vagrant_gitrepos 480378876 417410752  62968124  87% /vagrant/gitrepos
overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/236a03f55a2489dca2c749ddd2cd1fe71fe20786c9baf6012d9bbd3874ece4d8/merged
overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/6bf739da37cde03d1ae3c0e8111cde755a38684cb3f1407f3747bcfc47aacbaa/merged
shm                  65536         0     65536   0% /var/lib/docker/containers/f3680a9416703e5cdcf900ffd4dad6b35793ed05f1d2fe6b179fcf8e6c0f3ecd/mounts/shm
overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/f5c0059b89fb03f4dd20ab2ddfec3705ca47f5803e48b3e4ce4d46177b916848/merged
shm                  65536         0     65536   0% /var/lib/docker/containers/51d9f6225a7a4a67a0bf7388440267015235c492a558ae02ba9faa99a0a5e576/mounts/shm
$ IFS=$'\n' CUR_DF_ARR=(`df`)
$ echo ${CUR_DF_ARR[0]}
Filesystem       1K-blocks      Used Available Use% Mounted on
$ echo ${CUR_DF_ARR[1]}
/dev/sda1         41921540  29813304  12108236  72% /
$ echo ${CUR_DF_ARR[@]}
Filesystem       1K-blocks      Used Available Use% Mounted on /dev/sda1         41921540  29813304  12108236  72% / devtmpfs           4919788         0   4919788   0% /dev tmpfs              4927228         0   4927228   0% /dev/shm tmpfs              4927228     41692   4885536   1% /run tmpfs              4927228         0   4927228   0% /sys/fs/cgroup tmpfs               985448         0    985448   0% /run/user/1000 vagrant_server   480378876 417410752  62968124  87% /vagrant/server vagrant_gitrepos 480378876 417410752  62968124  87% /vagrant/gitrepos overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/236a03f55a2489dca2c749ddd2cd1fe71fe20786c9baf6012d9bbd3874ece4d8/merged overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/6bf739da37cde03d1ae3c0e8111cde755a38684cb3f1407f3747bcfc47aacbaa/merged shm                  65536         0     65536   0% /var/lib/docker/containers/f3680a9416703e5cdcf900ffd4dad6b35793ed05f1d2fe6b179fcf8e6c0f3ecd/mounts/shm overlay           41921540  29813304  12108236  72% /var/lib/docker/overlay2/f5c0059b89fb03f4dd20ab2ddfec3705ca47f5803e48b3e4ce4d46177b916848/merged shm                  65536         0     65536   0% /var/lib/docker/containers/51d9f6225a7a4a67a0bf7388440267015235c492a558ae02ba9faa99a0a5e576/mounts/shm
```  
	
---
## Reference