--- 
layout: single
classes: wide
title: "[Git] 커밋 히스토리 로그 조회하기(git log)"
header:
  overlay_image: /img/git-bg.jpg
excerpt: 'git log 명령어를 사용해서 커밋 히스토리를 조회하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Git
tags:
    - Git
    - git log
    - commit history
toc: true
---  

## 커밋 히스토리 조회
저장소의 커밋 히스토리를 조회는 `git log` 명령어를 통해 가능하다. 

예제로 사용하는 저장소의 커밋 히스토리 전체를 `git log` 로 확인하면 아래와 같다.

```bash
$ git log 
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:30:19 2020 +0900

    removed e-file

commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:00:17 2020 +0900

    add e-file

commit 148a5b415db7955ddc23023511a51d272edb77a4
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:01:00 2020 +0900

    changed b-file

commit ee6f58de0bf55d3fc2ea1254735e564d236ca282
Merge: b1d8559 e8731bb
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:00:04 2020 +0900

    Merge branch 'qa'

commit b1d8559538fc1a6bf010d6b00bb4a2c654e6d2ca
Merge: 0c7075b aa14476
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 23:00:16 2020 +0900

    Merge branch 'dev'

commit e8731bb6a26770c2f0ec2afe531ef84bb9560119
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 22:00:20 2020 +0900

    add d-file

commit 0c7075b565de38cf23bc7abd0fb82447b6f2c41c
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 21:00:19 2020 +0900

    changed a-file

commit aa14476e1f94d3caa22ca4d33ee0585cdd37d3f6
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 20:00:20 2020 +0900

    add c-file

commit 63ba625ef447dd36f79a22721f2d30693950712a
Author: root <root@windowforsun-1.localdomain>
Date:   Sat Jun 27 22:00:20 2020 +0900

    add b-file

commit 1050d79b05d1eafcdd53acbdf5723500fb361a25
Author: root <root@windowforsun-1.localdomain>
Date:   Sat Jun 27 20:00:19 2020 +0900

    add a-file
```  

커밋 히스토리 출력에서 각 필드가 의미를 주석으로 적으면 아래와 같다. 

```
# 커밋 해시
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f
# 작성자 
Author: root <root@windowforsun-1.localdomain>
# 날짜
Date:   Mon Jun 29 21:30:19 2020 +0900

# 커밋 메시지
    removed e-file
```  


## 최근 커밋 히스토리 조회
저장소에 있는 전체 커밋 히스토리가 출력되기 때문에 개수를 기준으로 최근 커밋 히스토리는 `git log -<개수>` 로 확인 가능하다. 

```bash
$ git log -2
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f (HEAD -> master)
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:30:19 2020 +0900

    removed e-file

commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:00:17 2020 +0900

    add e-file
```  

## diff 정보를 포함해서 커밋 히스토리 조회
`--path` 또는 `-p` 옵션을 사용하면 각 커밋의 `diff` 결과를 보여준다. 

```bash
$ git log -p -4
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:30:19 2020 +0900

    removed e-file

diff --git a/e-file b/e-file
deleted file mode 100644
index d905d9d..0000000
--- a/e-file
+++ /dev/null
@@ -1 +0,0 @@
-e

commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:00:17 2020 +0900

    add e-file

diff --git a/e-file b/e-file
new file mode 100644
index 0000000..d905d9d
--- /dev/null
+++ b/e-file
@@ -0,0 +1 @@
+e

commit 148a5b415db7955ddc23023511a51d272edb77a4
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:01:00 2020 +0900

    changed b-file

diff --git a/b-file b/b-file
index 6178079..2997ea2 100644
--- a/b-file
+++ b/b-file
@@ -1 +1,2 @@
 b
+bb

commit ee6f58de0bf55d3fc2ea1254735e564d236ca282
Merge: b1d8559 e8731bb
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:00:04 2020 +0900

    Merge branch 'qa'
```  

## 변경된 파일 이름만 포함해서 커밋 히스토리 조회
`--name-only` 옵션을 사용하면 각 커밋에서 변경 이력이 있는 파일 이름을 보여준다. 

```bash
$ git log --name-only -4
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:30:19 2020 +0900

    removed e-file

e-file

commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:00:17 2020 +0900

    add e-file

e-file

commit 148a5b415db7955ddc23023511a51d272edb77a4
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:01:00 2020 +0900

    changed b-file

b-file

commit ee6f58de0bf55d3fc2ea1254735e564d236ca282
Merge: b1d8559 e8731bb
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:00:04 2020 +0900

    Merge branch 'qa'
```  


## 커밋 통계 정보 포함해서 커밋 히스토리 조회
`--stat` 옵션을 사용하면 각 커밋의 통계 정보를 조회 할 수 있다. 
여기서 통계라는 정보는 해당 커밋에서 수정, 삭제, 추가 된 파일 수와 관련된 정보이다. 

```bash
$ git log --stat -4
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:30:19 2020 +0900

    removed e-file

 e-file | 1 -
 1 file changed, 1 deletion(-)

commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:00:17 2020 +0900

    add e-file

 e-file | 1 +
 1 file changed, 1 insertion(+)

commit 148a5b415db7955ddc23023511a51d272edb77a4
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:01:00 2020 +0900

    changed b-file

 b-file | 1 +
 1 file changed, 1 insertion(+)

commit ee6f58de0bf55d3fc2ea1254735e564d236ca282
Merge: b1d8559 e8731bb
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:00:04 2020 +0900

    Merge branch 'qa'
```  

## 출력 형식 변경해서 커밋 히스토리 조회
`--pretty` 옵션을 사용하면 히스토리 내용을 보여줄 때, 
기본 형식이 아닌 커스텀하게 설정해서 조회가 가능하다. 
대표적으로 `--pretty=oneline` 옵션은 각 커밋을 한 라인으로 보여준다. 
`online` 외에도 `short`, `full,` `fuller` 가 있다. 

```bash
$ git log --pretty=oneline
4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f (HEAD -> master) removed e-file
0245a50e158d61fdf7a1cda9d8a7935048f9f58d add e-file
148a5b415db7955ddc23023511a51d272edb77a4 changed b-file
ee6f58de0bf55d3fc2ea1254735e564d236ca282 Merge branch 'qa'
b1d8559538fc1a6bf010d6b00bb4a2c654e6d2ca Merge branch 'dev'
e8731bb6a26770c2f0ec2afe531ef84bb9560119 (qa) add d-file
0c7075b565de38cf23bc7abd0fb82447b6f2c41c changed a-file
aa14476e1f94d3caa22ca4d33ee0585cdd37d3f6 (dev) add c-file
63ba625ef447dd36f79a22721f2d30693950712a add b-file
1050d79b05d1eafcdd53acbdf5723500fb361a25 add a-file
```  

`--pretty=foramt:<format>` 을 사용하면 작성한 `format` 에 맞춰 커밋 히스토리를 출력해 준다. 
별도의 애플리케이션에서 파싱하고자 할때 유용한 옵션이다. 
사용 가능한 관련 옵션은 아래와 같다.

옵션|설명
---|---
`%H`|커밋 해시
`%h`|짧은 길이 커밋 해시
`%T`|트리 해시
`%t`|짧은 길이 트리 해시
`%P`|부모 해시
`%p`|짧은 길이 부모 해시
`%an`|저자 이름
`%ae`|저자 메일
`%ad`|저자 시각 (형식은 –-date=옵션 참고)
`%ar`|저자 상대적 시각
`%cn`|커미터 이름
`%ce`|커미터 메일
`%cd`|커미터 시각
`%cr`|커미터 상대적 시각
`%s`|요약

관련 예시는 아래와 같다.

```bash
$ git log --pretty=format:"%h - %an, %ar : %s"
4ff0ad3 - root, 3 minutes ago : removed e-file
0245a50 - root, 33 minutes ago : add e-file
148a5b4 - root, 2 hours ago : changed b-file
ee6f58d - root, 2 hours ago : Merge branch 'qa'
b1d8559 - root, 23 hours ago : Merge branch 'dev'
e8731bb - root, 24 hours ago : add d-file
0c7075b - root, 25 hours ago : changed a-file
aa14476 - root, 26 hours ago : add c-file
63ba625 - root, 2 days ago : add b-file
1050d79 - root, 2 days ago : add a-file
```  

옵션에서 저자와 커미터라는 구분이 있는데, 저자는 원래 작업을 수행한 원작자이고 
커밋터는 마지막으로 작업을 적용한(저장소에 포함) 사람을 의미한다.  

`A` 가 작업을 프로젝트에 패치하고, 담당자 `B` 가 패치를 적용하면 저자는 `A` 이고 커미터는 `B` 가 된다. 
관련 더 자세한 설명은 [여기](https://git-scm.com/book/ko/v2/%EB%B6%84%EC%82%B0-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C%EC%9D%98-Git-%EB%B6%84%EC%82%B0-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C%EC%9D%98-%EC%9B%8C%ED%81%AC%ED%94%8C%EB%A1%9C#ch05-distributed-git)
에서 확인 가능하다.  

## 트리 구조로 커밋 히스토리 조회
`--graph` 를 사용하면 커밋들의 히스토리를 트리(그래프) 형식으로 보여준다. 

```bash
$ git log --graph -4
* commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f (HEAD -> master)
| Author: root <root@windowforsun-1.localdomain>
| Date:   Mon Jun 29 21:30:19 2020 +0900
|
|     removed e-file
|
* commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
| Author: root <root@windowforsun-1.localdomain>
| Date:   Mon Jun 29 21:00:17 2020 +0900
|
|     add e-file
|
* commit 148a5b415db7955ddc23023511a51d272edb77a4
| Author: root <root@windowforsun-1.localdomain>
| Date:   Mon Jun 29 20:01:00 2020 +0900
|
|     changed b-file
|
*   commit ee6f58de0bf55d3fc2ea1254735e564d236ca282
|\  Merge: b1d8559 e8731bb
| | Author: root <root@windowforsun-1.localdomain>
| | Date:   Mon Jun 29 20:00:04 2020 +0900
| |
| |     Merge branch 'qa'
```  

`--pretty` 옵션과 함께 사용하면 아래와 같이 출력할 수 있다.

```bash
$ git log --pretty=format:"%h %s" --graph
* 4ff0ad3 removed e-file
* 0245a50 add e-file
* 148a5b4 changed b-file
*   ee6f58d Merge branch 'qa'
|\
| * e8731bb add d-file
* |   b1d8559 Merge branch 'dev'
|\ \
| |/
|/|
| * aa14476 add c-file
* | 0c7075b changed a-file
|/
* 63ba625 add b-file
* 1050d79 add a-file
```  

## 출력 관련 옵션 정리
`git log` 명령에서 출력과 관련된 옵션을 정리하면 아래와 같다. 

옵션|설명
---|---
-p|각 커밋에 적용된 패치를 보여준다.
--stat|각 커밋에서 수정된 파일의 통계정보를 보여준다.
--shortstat|--stat 명령의 결과 중에서 수정한 파일, 추가된 라인, 삭제된 라인만 보여준다.
--name-only|커밋 정보중에서 수정된 파일의 목록만 보여준다.
--name-status|수정된 파일의 목록을 보여줄 뿐만 아니라 파일을 추가한 것인지, 수정한 것인지, 삭제한 것인지도 보여준다.
--abbrev-commit|40자 짜리 SHA-1 체크섬을 전부 보여주는 것이 아니라 처음 몇 자만 보여준다.
--relative-date|정확한 시간을 보여주는 것이 아니라 “2 weeks ago” 처럼 상대적인 형식으로 보여준다.
--graph|브랜치와 머지 히스토리 정보까지 아스키 그래프로 보여준다.
--pretty|지정한 형식으로 보여준다. 이 옵션에는 oneline, short, full, fuller, format이 있다. format은 원하는 형식으로 출력하고자 할 때 사용한다.
--oneline|--pretty=oneline --abbrev-commit 두 옵션을 함께 사용한 것과 같다.


## 특정 날짜 범위에 해당하는 커밋 히스토리 조회
앞에서 `-<숫자>` 를 통해 최근 몇개의 커밋을 조회하는 방법에 대해서 알아보았다. 
이처럼 특정 범위에 해당하는 커밋 히스토리를 시간을 기준으로 조회하는 옵션으로 `--since`, `--until`, `--after`, `--before` 이 있다. 
여기서 `--since=<기간>` 는 최근 ~ `최근 - <기간>`, `--until=<기간>` 은 `최근 - <기간>` ~ 시작점 을 의미한다. 

```bash
$ date
Mon Jun 29 21:35:13 KST 2020

$ git log --since=2.hours
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f (HEAD -> master)
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:30:19 2020 +0900

    removed e-file

commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:00:17 2020 +0900

    add e-file

commit 148a5b415db7955ddc23023511a51d272edb77a4
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:01:00 2020 +0900

    changed b-file

commit ee6f58de0bf55d3fc2ea1254735e564d236ca282
Merge: b1d8559 e8731bb
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:00:04 2020 +0900

    Merge branch 'qa'
```  

```bash
$ date
Mon Jun 29 21:35:57 KST 2020

$ git log --until=2.hours
commit b1d8559538fc1a6bf010d6b00bb4a2c654e6d2ca
Merge: 0c7075b aa14476
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 23:00:16 2020 +0900

    Merge branch 'dev'

commit e8731bb6a26770c2f0ec2afe531ef84bb9560119
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 22:00:20 2020 +0900

    add d-file

commit 0c7075b565de38cf23bc7abd0fb82447b6f2c41c
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 21:00:19 2020 +0900

    changed a-file

commit aa14476e1f94d3caa22ca4d33ee0585cdd37d3f6
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 20:00:20 2020 +0900

    add c-file

commit 63ba625ef447dd36f79a22721f2d30693950712a
Author: root <root@windowforsun-1.localdomain>
Date:   Sat Jun 27 22:00:20 2020 +0900

    add b-file

commit 1050d79b05d1eafcdd53acbdf5723500fb361a25
Author: root <root@windowforsun-1.localdomain>
Date:   Sat Jun 27 20:00:19 2020 +0900

    add a-file
```  

```bash
$ date
Mon Jun 29 21:36:35 KST 2020

$ git log --after="2020-06-28"
commit 4ff0ad3931b0a3feb62e3556c1fc2a4a4985760f
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:30:19 2020 +0900

    removed e-file

commit 0245a50e158d61fdf7a1cda9d8a7935048f9f58d
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 21:00:17 2020 +0900

    add e-file

commit 148a5b415db7955ddc23023511a51d272edb77a4
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:01:00 2020 +0900

    changed b-file

commit ee6f58de0bf55d3fc2ea1254735e564d236ca282
Merge: b1d8559 e8731bb
Author: root <root@windowforsun-1.localdomain>
Date:   Mon Jun 29 20:00:04 2020 +0900

    Merge branch 'qa'

commit b1d8559538fc1a6bf010d6b00bb4a2c654e6d2ca
Merge: 0c7075b aa14476
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 23:00:16 2020 +0900

    Merge branch 'dev'

commit e8731bb6a26770c2f0ec2afe531ef84bb9560119
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 22:00:20 2020 +0900

    add d-file
```  

```bash
$ date
Mon Jun 29 21:37:41 KST 2020

$ git log --before="2020-06-28
commit 0c7075b565de38cf23bc7abd0fb82447b6f2c41c
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 21:00:19 2020 +0900

    changed a-file

commit aa14476e1f94d3caa22ca4d33ee0585cdd37d3f6
Author: root <root@windowforsun-1.localdomain>
Date:   Sun Jun 28 20:00:20 2020 +0900

    add c-file

commit 63ba625ef447dd36f79a22721f2d30693950712a
Author: root <root@windowforsun-1.localdomain>
Date:   Sat Jun 27 22:00:20 2020 +0900

    add b-file

commit 1050d79b05d1eafcdd53acbdf5723500fb361a25
Author: root <root@windowforsun-1.localdomain>
Date:   Sat Jun 27 20:00:19 2020 +0900

    add a-file
```  

## 수정된 내용을 바탕으로 커밋 히스토리 조회
`-S` 를 사용하면 



























































git log --since

git log --until

git log -S

git log -- path1 path2




















































---
 
## Reference
[Git의 기초 - 커밋 히스토리 조회하기](https://git-scm.com/book/ko/v2/Git%EC%9D%98-%EA%B8%B0%EC%B4%88-%EC%BB%A4%EB%B0%8B-%ED%9E%88%EC%8A%A4%ED%86%A0%EB%A6%AC-%EC%A1%B0%ED%9A%8C%ED%95%98%EA%B8%B0#pretty_format)


