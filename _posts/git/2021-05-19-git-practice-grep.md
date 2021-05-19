--- 
layout: single
classes: wide
title: "[Git] git grep, git log 명령을 사용해서 검색하기"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'git grep, git log 명령을 사용해서 파일 내용, 커밋 메시지, 변경 사항에 대해 검색을 수행해보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - git log
    - git grep
toc: true
---  

## Git 검색
많은 파일과 커밋 중 특정 내용이 포함된 파일을 검색해야 되거나, 
커밋 로그를 검색하거나, 
특정 변경 내용이 있는 커밋 로그를 검색해야하는 상황이 발생할 수 있다. 
이때 `Git` 에서 이런 상황에 사용할 수 있는 명령어에 대해 알아본다.  

### 테스트 프로젝트
테스트로 사용할 프로젝트의 파일 구성은 아래와 같이 총 4개의 파일로 구성과 각 파일의 내용은 아래와 같다.  

```bash
.
├── hello-git
├── hello-java
├── hello-special
└── hello-world


$ cat hello-git
hello
git
$ cat hello-java
hello
java
$ cat hello-special
hello
!!
??
()
world!!
java~~
$ cat hello-world
hello
world
```  

그리고 프로젝트의 커밋 로그는 아래와 같다.  

```bash
$ git log -p --graph --pretty=short
* commit 9a0d8d38a53eb1cf69d5df66346ed1b70e94d7bf
| Author: windowforsun <window_for_sun@naver.com>
|
|     modify special file 4
|
| diff --git a/hello-special b/hello-special
| index d4a23d8..5aa7ebe 100644
| --- a/hello-special
| +++ b/hello-special
| @@ -2,3 +2,5 @@ hello
|  !!
|  ??
|  ()
| +world!!
| +java~~
|
* commit d9c40ec2b0e93b9ccb2d30ca6d522e4655dc3d17
| Author: windowforsun <window_for_sun@naver.com>
|
|     modify special file 3
|
| diff --git a/hello-special b/hello-special
| index b52f337..d4a23d8 100644
| --- a/hello-special
| +++ b/hello-special
| @@ -1,3 +1,4 @@
|  hello
|  !!
|  ??
| +()
|
* commit 16c73eb65df37086bc2f44c2e40cf96b0df7af0b
| Author: windowforsun <window_for_sun@naver.com>
|
|     modify special file 2
|
| diff --git a/hello-special b/hello-special
| index b303584..b52f337 100644
| --- a/hello-special
| +++ b/hello-special
| @@ -1,2 +1,3 @@
|  hello
|  !!
| +??
|
* commit a84ce5ba05aad3b5cf7941ff4543918ef4312d22
| Author: windowforsun <window_for_sun@naver.com>
|
|     modify special file 1
|
| diff --git a/hello-special b/hello-special
| index ce01362..b303584 100644
| --- a/hello-special
| +++ b/hello-special
| @@ -1 +1,2 @@
|  hello
| +!!
|
* commit 577fae25afd0358d49c52c0d3a89a2c421b2d2b3
| Author: windowforsun <window_for_sun@naver.com>
|
|     new special file
|
| diff --git a/hello-special b/hello-special
| new file mode 100644
| index 0000000..ce01362
| --- /dev/null
| +++ b/hello-special
| @@ -0,0 +1 @@
| +hello
|
* commit 9bf523545cb078c9a429ba31f7b0691f57538a36
| Author: windowforsun <window_for_sun@naver.com>
|
|     modify java file
|
| diff --git a/hello-java b/hello-java
| index ce01362..beab815 100644
| --- a/hello-java
| +++ b/hello-java
| @@ -1 +1,2 @@
|  hello
| +java
|
* commit 9d7288ce51346dfeed82e78471b64f91a5f7b4ab
| Author: windowforsun <window_for_sun@naver.com>
|
|     new java file
|
| diff --git a/hello-java b/hello-java
| new file mode 100644
| index 0000000..ce01362
| --- /dev/null
| +++ b/hello-java
| @@ -0,0 +1 @@
| +hello
|
* commit 89331bd3915c058c8814112762a63fb40d5a05b7
| Author: windowforsun <window_for_sun@naver.com>
|
|     modify git file
|
| diff --git a/hello-git b/hello-git
| index ce01362..23509e0 100644
| --- a/hello-git
| +++ b/hello-git
| @@ -1 +1,2 @@
|  hello
| +git
|
* commit 1630f2b3bfcbf0423d1605dd86bb48c6cdb2e08d
| Author: windowforsun <window_for_sun@naver.com>
|
|     modify world file
|
| diff --git a/hello-world b/hello-world
| index ce01362..94954ab 100644
| --- a/hello-world
| +++ b/hello-world
| @@ -1 +1,2 @@
|  hello
| +world
|
* commit ed31a62799919ace1a28ac3bd66f3b124c2cdb27
| Author: windowforsun <window_for_sun@naver.com>
|
|     new git file
|
| diff --git a/hello-git b/hello-git
| new file mode 100644
| index 0000000..ce01362
| --- /dev/null
| +++ b/hello-git
| @@ -0,0 +1 @@
| +hello
|
* commit b986f2757b260b20e1e17e0304c2065e24c7a706
  Author: windowforsun <window_for_sun@naver.com>

      new world file

  diff --git a/hello-world b/hello-world
  new file mode 100644
  index 0000000..ce01362
  --- /dev/null
  +++ b/hello-world
  @@ -0,0 +1 @@
  +hello
```  

### 파일 내용 검색하기(git grep)
`git grep` 명령을 사용하면 `git` 에서 형상관리 되는 파일 중 포함된 내용에 대한 검색을 할 수 있다. 
간단한 예시로 `hello` 가 포함된 파일을 검색하면 아래와 같이 모든 파일이 검색된다. 
더 자세한 설명은 [공식 문서](https://git-scm.com/docs/git-grep)
에서 확인 할 수 있다.  

```bash
$ git grep hello
hello-git:hello
hello-java:hello
hello-special:hello
hello-world:hello
```  

`-l` 옵션을 주면 파일의 내용은 포함하지 않고 파일내용이 포함된 파일만 결과로 얻을 수 있다.  

```bash
$ git grep -l hello
hello-git
hello-java
hello-special
hello-world
```  

그리고 명령어 뒤에 `git grep <bracn> <working-tree>` 와 같은 형식으로 
특정 브랜치나 특정 경로를 지정해서 검색을 수행할 수 있다.  

```bash
$ git grep hello master .
master:hello-git:hello
master:hello-java:hello
master:hello-special:hello
master:hello-world:hello
```  

정규식도 검색 조건으로 사용 가능하다.  

```bash
$ git grep [a-z]
hello-git:hello
hello-git:git
hello-java:hello
hello-java:java
hello-special:hello
hello-special:world!!
hello-special:java~~
hello-world:hello
hello-world:world
```  

조건이 여러개인 경우 `-e` 옵션과 `--or`, `--and`, `--not` 등을 사용해서 조건을 구성할 수 있다.  

```bash
$ git grep -e [j] --and -e [~]
hello-special:java~~
```  

### 커밋 로그 검색하기 (git log)
`git` 에서 커밋은 사용자가 커밋에 작성한 커밋 메시지와 실제 파일에 대한 변경사항으로 나뉠 수 있다.  

먼저 특정 커밋 메시지를 검색하는 방법에 대해 살펴본다.  
특정 문자열이 포함된 커밋 메시지의 검색은 `git log --grep <regex>` 와 같이 사용할 수 있다.  

```bash
.. new 문자열이 포함된 커밋 메시지 ..
$ git log --oneline --grep new
577fae2 new special file
9d7288c new java file
ed31a62 new git file
b986f27 new world file

.. modify 문자열이 포함된 커밋 메시지 ..
$ git log --oneline --grep modify
9a0d8d3 (HEAD -> master) modify special file 4
d9c40ec modify special file 3
16c73eb modify special file 2
a84ce5b modify special file 1
9bf5235 modify java file
89331bd modify git file
1630f2b modify world file

.. 정규식 [w] 를 만족하는 커밋 메시지 ..
git log --oneline --grep [w]
577fae2 new special file
9d7288c new java file
1630f2b modify world file
ed31a62 new git file
b986f27 new world file
```  

#### 파일 변경사항 검색(git log -G)
다음으로 커밋에서 파일 변경내용에 포함되는 추가되고 삭제된 내용을 검색하는 방법은 
`git log -G<regex>` 와 같이 사용 할 수 있다.  

```bash
.. 파일 변경 내용 중 java 가 포함되는 커밋 ..
$ git log --oneline -p -Gjava
9a0d8d3 (HEAD -> master) modify special file 4
diff --git a/hello-special b/hello-special
index d4a23d8..5aa7ebe 100644
--- a/hello-special
+++ b/hello-special
@@ -2,3 +2,5 @@ hello
 !!
 ??
 ()
+world!!
+java~~
9bf5235 modify java file
diff --git a/hello-java b/hello-java
index ce01362..beab815 100644
--- a/hello-java
+++ b/hello-java
@@ -1 +1,2 @@
 hello
+java

.. 파일 변경 내용 중 ! 가 포함되는 커밋 ..
$ git log --oneline -p -G"!"
9a0d8d3 (HEAD -> master) modify special file 4
diff --git a/hello-special b/hello-special
index d4a23d8..5aa7ebe 100644
--- a/hello-special
+++ b/hello-special
@@ -2,3 +2,5 @@ hello
 !!
 ??
 ()
+world!!
+java~~
a84ce5b modify special file 1
diff --git a/hello-special b/hello-special
index ce01362..b303584 100644
--- a/hello-special
+++ b/hello-special
@@ -1 +1,2 @@
 hello
+!!

.. 파일 변경 내용 중 () 가 포함되는 커밋 ..
$ git log --oneline -p -G'\(\)'
d9c40ec modify special file 3
diff --git a/hello-special b/hello-special
index b52f337..d4a23d8 100644
--- a/hello-special
+++ b/hello-special
@@ -1,3 +1,4 @@
 hello
 !!
 ??
+()
```  





---
 
## Reference
[git-grep Documentation - Git](https://git-scm.com/docs/git-grep)  
[Git 히스토리에서 커밋 된 코드를 grep (검색)하는 방법](https://melkia.dev/ko/questions/2928584)  


