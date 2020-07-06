--- 
layout: single
classes: wide
title: "[Linux 실습] 유용한 Bash 문법"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Bash 명령을 사용할 때 유용하게 쓰이는 문법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - Bash
toc: true
use_math: true  
---  

## 입출력 조작
문법에 사용되는 숫자의 의미는 아래와 같다. 
- `0` : 표준 입력(`stdin`)
- `1` : 표준 출력(`stdout`)
- `2` : 표준 에러(`stderr`)

### 명령 표준 출력 리다이렉션 저장
`>` 은 출력 리다이렉션 역할로 명령 실행의 표준 출력(`stdout`)을 파일로 저장한다. 
리눅스의 장치관리의 특징을 이용하면 명령의 결과를 특정 장치로 보내는 것도 가능하다. 
지정된 파일에 저장하기 때문에 기존 파일이 존재하는 경우 덮어쓰게 된다. 

```bash
$ echo "stdout redirection ~" > ./result.txt
$ cat result.txt
stdout redirection ~
```  

### 명령 표준 출력 리다이렉션 추가
`>>` 은 출력 리다이렉션 역할로 `>` 문법과 비슷한 동작을 수행하지만,
차이로는 명령 실행의 표준 출력(`stdout`)을 파일에 추가하는 부분이다. 
기존 파일이 존재하는 경우 파일 뒷부분에 내용을 추가한다. 

```bash
$ cat result.txt
stdout redirection ~
$ echo "append ~~" >> ./result.txt
$ cat result.txt
stdout redirection ~
append ~~
```  

### 명령 표준 입력 리다이렉션
`<` 은 입력 리다이렉션 역할로 파일의 내용을 표준 입력(`stdin`) 으로 사용한다. 

```bash
$ cat < ./result.txt
stdout redirection ~
append ~~
```  

### 명령 표준 에러 리다이렉션 저장
`2>` 은 명령 실행 표준 에러(`stderr`)을 파일로 저장한다. 
`ls` 명령에 존재하지 않은 옵션인 `--notexist` 를 호출했을 때의 에러를 발생시켜 테스트하면 아래와 같다. 

```bash
$ ls --notexist 2> error.txt
$ cat error.txt
ls: unknown option -- notexist
Try 'ls --help' for more information.
```  

### 명령 표준 에러 리다이렉션 추가
`2>>` 은 `2>` 와 비슷하지만, 차이점은 명령 실행 표준 에러(`stderr`)을 파일에 추가하다는 점이다. 

```bash
$ ls --notexist2 2>> error.txt
$ cat error.txt
ls: unknown option -- notexist
Try 'ls --help' for more information.
ls: unknown option -- notexist2
Try 'ls --help' for more information.
```  

### 명령 표준 출력, 에러 리다이렉션
`&>` 은 명령 실행 표준 출력과 표준 에러를 파일로 저장한다. 

```bash
$ ls --notexist &> log.txt
$ cat log.txt
ls: unknown option -- notexist
Try 'ls --help' for more information.
$ echo "stdout redirection" &> log.txt
$ cat log.txt
stdout redirection
```  

### 명령 표준 출력을 표준 에러로 리다이렉션
`1>&2` 은 표준 출력을 표준 에러로 보낸다. 

```bash
$ echo "this is stdout" 2> log 1>&2
$ cat log
this is stdout
```  

위 예제에서 먼저 `echo "this is stout" 2> log` 는 명령 수행 에러를 `log` 파일로 저장하는 동작이다. 
여기서 `echo "this is stout` 은 에러가 발생하지 않는 정상적인 문법을 수행하고 있지만, 
뒷 부분에 사용된 `1>&2` 로 인해 표준 출력이 표준 에러로 리다이렉션 되어 `log` 파일레 표준 출력값이 저장된다. 


### 명령 표준 에러를 표준 출력으로 리다이렉션
`2>&1` 은 표준 에러를 표준 출력으로 보낸다.

```bash
$ cat not/exist/file > log
cat: not/exist/file: No such file or directory
$ cat not/exist/file > log 2>&1
$ cat log
cat: not/exist/file: No such file or directory
```  

위 예제를 보면 `cat` 명령을 수행하는 `/not/exist/file` 은 존재하지 않는 파일 경로이다. 
해당 파일 내용을 `log` 라는 파일에 저장하기 위해서 `cat not/exist/file > log` 를 수행하면, 
콘솔에 에러 메시지가 출력된다. 
`2>&1` 을 사용해서 표준 에러를 표준 출력으로 보낼 콘솔은 에러 메시지 없이 깔끔하게 유지하고, 에러는 별도의 파일로 저장가능하다.  

명령 수행에 대한 에러 메시지를 머리고 싶을 경우 아래와 같이 수행 가능하다. 

```bash
$ cat not/exist/file > /dev/null 2>&1
```  

명령 수행 결과를 `/dev/null` 로 보내 출력을 버리는데, 이때 에러 또한 `2>&1` 표준 출력으로 리다이렉션 되어 버려진다. 
    
    
### 명령 표준 출력을 다른 명령 표준 입력으로 리다이렉션
`|` 은 파이프라인 문법으로 명령 실행 표준 출력을 다른 표준 입력으로 보낸다. 
`|` 앞 명령의 출력이 뒷 명령의 입력으로 사용되 처리한다. 

```bash
$ touch a aa b bb ab
$ ls
a  aa  ab  b  bb
$ ls | grep a
a
aa
ab
```  


### 문자열을 표준 입력으로 사용
`<<<` 은 문자열을 명령 수행에 필요한 표준 입력으로 보낸다. 

```bash
$ cat <<< "hello ~~"
hello ~~
$ sed 's/o/~/g' <<< "hello world"
hell~ w~rld
```  

두 번째는 `sed` 명령을 사용해서 `<<<` 을 이용한 `hello world` 표쥰 입력에서 `o` 문자을 `~` 로 변경하는 예제이다.  

여러 줄의 문자열을 표준 입력으로 보낼 때는 `<<EOF` 명령을 사용할 수 있다. 

```bash
$ cat <<EOF
> hello
> world
> ~
> ~
> EOF
hello
world
~
~
```  


## 변수
### 변수사용
`$` 은 `Bash` 에서 변수를 사용할 때 사용한다. 
변수에 값 할당 시에는 사용하지 않고, 
변수의 값을 사용할 때 사용한다. 

```bash
$ var1=value1
$ echo $var1
value1
```  


### 변수 치환
`${}` 은 변수 치환할 때 사용한다. 
문자열 `""` 사이에서 변수를 출력하거나 표현해야 할대 사용한다. 

```bash
$ hello="Hello~~"
$ echo "${hello} World"
Hello~~ World
```  

스크립트에서 변수 기본 값을 설정할때도 사용 할 수 있다. 
`${<변수명>-<기본값>}` 은 변수가 존재하면 그대로 사용하고, 존재하지 않으면 설정한 기본값을 대입한다. 

```bash
$ hello=
$ hello=${hello-"Hello~~"}
$ echo $hello
$ notexist=${notexist-"default"}
$ echo $notexist
default
```  

`${<변수명}:-{기본값}}` 은 변수의 값이 `null` 일 경우 설정한 기본 값을 대입한다. 

```bash
$ hello=
$ hello=${hello:-"Hello~~"}
$ echo $hello
Hello~~
```


### 명령 결과 변수화
`` ` ` ``, `$()` 은 명령 수행 결과를 변수화 한다. 
`` ` ` ``, `$()` 모두 사용법은 동일하다. 
명령 수행 결과를 변수에 할당하거나, 매개 변수로 넘겨 줄때, 명령 결과를 문자열로 받을 때 사용 가능하다. 

```bash
$ files=`ls`
$ echo $files
a aa ab b bb
$ echo "$files"
a
aa
ab
b
bb

$ files=$(ls)
$ echo $files
a aa ab b bb
$ echo "$files"
a
aa
ab
b
bb
```  

`ls` 명령 수행의 결과를 `files` 라는 변수에 넣고, `echo` 명령을 통해 출력 가능하다. 
출력시에 `"$files"` 처럼 `"` 로 변수를 감싸주면 개행문자가 붙어 출력된다. 

```bash
$ cat a b aa bb ab
a
b
aa
bb
ab
$ cat $(ls)
a
aa
ab
b
bb
```  

현재 결로에 있는 5개 파일(`a`, `b`, `aa`, `bb`, `ab`) 에는 파일명과 동일한 문자의 내용이 들어있다. 
`cat` 명령으로 현재 경로에 있는 5개 파일명을 직접 사용해서 출력한 결과와 
`$(ls)` 명령을 사용해서 `cat` 명령의 인자값으로 현재 경로의 파일 출력 리스트를 전달해 출력한 결과가 동일한 것을 확인 할 수 있다. 

```bash
$ result=`cat $(ls)`
$ echo $result
a aa ab b bb


$ result2=$(cat `ls`)
$ echo $result2
a aa ab b bb
```  

위 예제처럼 두 문법을 중첩해서 사용하는 것도 가능하다.


## 명령 수행
### 한 줄에서 여러 명령 수행
`&&`, `;` 은 여러 명령을 한줄로 실행가능하다. 
- `&&` : 선행되는 명령에서 에러가 발생하지 않아야 다음 명령을 수행한다. 
- `;` : 선행되는 명령에서 에러가 발생하더라도 다음 명령을 수행한다. 

```bash
$ ls && echo "hello"
a  aa  ab  b  bb
hello
$ ls; echo "hello"
a  aa  ab  b  bb
hello
$ ls --notexist && echo "hello"
ls: unknown option -- notexist
Try 'ls --help' for more information.
$ ls --notexist; echo "hello"
ls: unknown option -- notexist
Try 'ls --help' for more information.
hello
```  

### 한 줄 명령을 여러 출로 표현
`\` 은 한 줄로 된 긴 명령을 여러 줄로 표현 가능하게 한다. 

```bash
$ ls \
> -l \
> -a \
> -s
total 25
 0 drwxr-xr-x 1 ckdtj 197609 0 7월   5 18:39 ./
20 drwxr-xr-x 1 ckdtj 197609 0 7월   5 18:39 ../
 1 -rw-r--r-- 1 ckdtj 197609 2 7월   5 18:39 a
 1 -rw-r--r-- 1 ckdtj 197609 3 7월   5 18:39 aa
 1 -rw-r--r-- 1 ckdtj 197609 3 7월   5 18:39 ab
 1 -rw-r--r-- 1 ckdtj 197609 2 7월   5 18:39 b
 1 -rw-r--r-- 1 ckdtj 197609 3 7월   5 18:39 bb
```  


## 문자열
### 치환 없는 문자열
`''` 은 문자열 표현 중 하나로 `'` 사이에 들어간 문자는 변수나 다른 것으로 치환되지 않고 문자열로만 표현된다. 

```bash
$ test=hello
$ echo '$test'
$test
$ echo '$(ls)'
$(ls)
```  


### 치환 있는 문자열
`""` 은 문자열 표현 중 하나로 `"` 사이에 들어간 문자는 변수나 정해진 조건에 만족하는 것으로 치환 될 수 있다. 

```bash
$ test=hello
$ echo "$test"
hello
$ echo "$(ls)"
a
aa
ab
b
bb
```  

치환 있는 문자열 `"` 사이에는 `'` 가 명령에서 문자열 매개변수를 지정할 때 사용될 수 있다. 

### 문자열 특수문자
문자열 문법 사이에서 특수문자를 사용할때 특수문자 앞에 `\` 을 붙이면 특수문자 그대로 사용이 가능하다. 

```bash
$ echo "{\"key\":\"value\"}"
{"key":"value"}
$ echo '{"key":"value"}'
{"key":"value"}
$ echo "this is a 'str'"
this is a 'str'
$ echo "this is a $str"
this is a
$ echo "this is a \$str"
this is a $str
```  























































---
## Reference
 
	