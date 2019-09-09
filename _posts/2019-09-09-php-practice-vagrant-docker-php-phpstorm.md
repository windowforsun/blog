--- 
layout: single
classes: wide
title: "[PHP 실습] Docker Compose + Nginx + PHP Composer + PHPUnit"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Docker Compose 를 사용해서 Nginx + PHP Compose + PHPUnit 실행 및 디버깅 환경을 구성하자'
author: "window_for_sun"
header-style: text
categories :
  - PHP
tags:
    - Practice
    - PHP
    - Docker
    - PHPUnit
    - PHPComposer
    - DockerCompose
    - Nginx
    - Jetbrains
---  

## 환경
- Centos 7
- Docker
- PhpStorm
- Vagrant
- PHP 7
- Nginx
- PHPUnit
- PHP Composer

## Vagrant(VM) 에 Docker 설치

## Docker + PHP
- PHP Composer 를 사용하지 않은 상태에서 환경을 구성한다.

### 프로젝트 구조

![그림 1]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-1.png)

- File -> New Project -> PHP Empty Project 에서 새로운 프로젝트를 생성한다.
- `index.php`, `info.ph` 파일을 생성한다.
	- `index.php`
	
		```php
		<?php
	
		echo "Hi~ Docker PHP";
		```  
	
	- `info.php`
	
		```php
		<?php

		echo phpinfo();
		```  
		
### 기본 Docker 파일 만들기
- 프로젝트 루트 경로에 `docker` 디렉토리를 생성하고 아래와 같은 구조로 만들어 준다.

	![그림 2]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-2.png)

- `docker/docker-compose.yml` 내용

	```yaml
	version: '3'

	services:
	  test2nginx:   # nginx 설정
	    image: nginx:latest     # 사용할 nginx docker image
	    ports:
	      - "80:80"     # port forwarding 설정
	    volumes:    # 공유 폴더 및 파일 설정
	      - /vagrant/server/testdocker2:/opt/project      # 프로젝트 소스코드
	      - ./nginx/site.conf:/etc/nginx/conf.d/site.conf   # nginx 설정 파일
	    networks:   # 컨테이너 네트워크 설정
	      - nginx-php
	  test2php:     # php 설정
	    build:  # 빌드 시 사용할 Dockerfile 설정
	      context: ./php-fpm    # 경로
	      dockerfile: Dockerfile    # 파일 이름
	    volumes:
	      - /vagrant/server/testdocker2:/opt/project      # 프로젝트 소스코드
	    networks:
	      - nginx-php
	
	networks:   # 네트워크 선언
	  nginx-php:
	```  
	
	- Docker PHP Image 의 소스코드 경로는 `/opt/project` 이다.
	
- `docker/php-fpm/Dockerfile`

	```
	# 사용할 docker image
	FROM php:7.3-fpm    
	
	# xdebug 설치
	RUN pecl install xdebug \   
	  && docker-php-ext-enable xdebug
	
	# xdebug 설정 파일 복사
	COPY ./xdebug.ini /usr/local/etc/php/conf.d/
	```  
	
- `docker/php-fpm/xdebug.ini`

	```
	xdebug.default_enable = 1
	xdebug.remote_enable = 1
	xdebug.remote_autostart = 1
	xdebug.remote_connect_back = 0
	xdebug.remote_host = 192.168.99.1
	xdebug.remote_port = 9014
	xdebug.profiler_enable = 0
	xdebug.idekey = PHPSTORM
	xdebug.remote_handler = dbgp
	xdebug.remote_mode = req
	```  
	
	- `xdebug.remote_host` 의 아이피는 Virtualbox 의 Host-Only Ethernet Adapter 아이피를 사용한다.
	
- `docker/nginx/site.conf` 내용

	```
	server {
	    index index.php index.html;
	    server_name 127.0.0.1;
	    error_log  /var/log/nginx/error.log;
	    access_log /var/log/nginx/access.log;
	    root /opt/project;
	
	    location ~ \.php$ {
	        try_files $uri =404;
	        fastcgi_split_path_info ^(.+\.php)(/.+)$;
	        fastcgi_pass test2php:9000;
	        fastcgi_index index.php;
	        include fastcgi_params;
	        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	        fastcgi_param PATH_INFO $fastcgi_path_info;
	    }
	}
	```  
	
	- `fastcgi_pass` 설정에서 호스트 이름은 `docker-compose` 에서 사용했던 PHP 의 `services` 이름을 사용한다.
	
### Docker Compose 빌드 및 웹 요청 테스트
- VM(Vagarnt) 에서 `<프로젝트 경로>/docker` 로 이동한다.
- `docker-compose up --build` 명령어를 통해 `docker-compose.yml` 파일을 빌드한다.

	```
	[root@localhost docker]# docker-compose up --build
	Building test2php
	Step 1/3 : FROM php:7.3-fpm
	 ---> 7d3076bd7d18
	Step 2/3 : RUN pecl install xdebug   && docker-php-ext-enable xdebug
	 ---> Using cache
	 ---> e3e2494b2ae7
	Step 3/3 : COPY ./xdebug.ini /usr/local/etc/php/conf.d/
	 ---> Using cache
	 ---> 5eae354859c1
	Successfully built 5eae354859c1
	Successfully tagged docker_test2php:latest
	Recreating docker_test2nginx_1 ... done
	Recreating docker_test2php_1   ... done
	Attaching to docker_test2php_1, docker_test2nginx_1
	test2php_1    | [09-Sep-2019 07:46:47] NOTICE: fpm is running, pid 1
	test2php_1    | [09-Sep-2019 07:46:47] NOTICE: ready to handle connections
	```  
	
	- Docker Image 다운로드 및 Docker Container 초기 설정에 따라 출력 결과가 달라 질 수 있다.	
	
- `http://<서버IP>:<서버Port>`, `index.php` 결과

	![그림 3]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-3.png)
	
- `http://<서버IP>:<서버Port>/info.php`, `info.php` 결과

	![그림 4]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-4.png)	

### 웹 요청 디버깅 설정
- File -> Settings -> Language & Frameworks -> Debug 설정에서 `Debug port` 를 `xdebug.ini` 에서 설정한 포트로 변경 해준다.

	![그림 5]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-5.png)	

- `http://<서버IP>:<서버Port>:/info.php` 로 요청을 보내면 디버깅이 가능하다.

	![그림 6]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-6.png)		

### Docker Remote API
- Docker Remote API 는 Third-Party 애플리케이션에서 Docker 의 명령어를 사용할 있게 지원해주는 API 이다.

#### Docker Remote API 사용 설정
- `vi /usr/lib/systemd/system/docker.service` 파일 내용은 아래와 같다.

	```
	[Unit]
	Description=Docker Application Container Engine
	Documentation=https://docs.docker.com
	BindsTo=containerd.service
	After=network-online.target firewalld.service containerd.service
	Wants=network-online.target
	Requires=docker.socket
	
	[Service]
	Type=notify
	# the default is not to use systemd for cgroups because the delegate issues still
	# exists and systemd currently does not support the cgroup feature set required
	# for containers run by docker
	ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
	ExecReload=/bin/kill -s HUP $MAINPID
	TimeoutSec=0
	RestartSec=2
	Restart=always
	
	생략 ...
	```  
	
- `xecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:<포트번호>` 를 추가해 준다.

	```
	[Unit]
	Description=Docker Application Container Engine
	Documentation=https://docs.docker.com
	BindsTo=containerd.service
	After=network-online.target firewalld.service containerd.service
	Wants=network-online.target
	Requires=docker.socket
	
	[Service]
	Type=notify
	# the default is not to use systemd for cgroups because the delegate issues still
	# exists and systemd currently does not support the cgroup feature set required
	# for containers run by docker
	# 기존 설정 주석처리
	#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock 
	ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375
	ExecReload=/bin/kill -s HUP $MAINPID
	TimeoutSec=0
	RestartSec=2
	Restart=always
	
	# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
	# Both the old, and new location are accepted by systemd 229 and up, so using the old location
	# to make them work for either version of systemd.
	StartLimitBurst=3

	생략 ...
	```  

#### PhpStorm Docker Remote API 연결
- File -> Settings -> Build, Execution, Deployment -> Docker 에서 `+` 버튼을 누르고 아래와 같이 설정해 준다.

	![그림 7]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-7.png)
	
	- `Engine API URL` 에서 `tcp:localhost:2385` 는 현재 vagrant 를 통해 포트포워딩 설정이 되어 있는 상태이다.
	- `C:\vm_share\centos7\server` 경로를 `/vagrant/server` VM(Vagrant) 경로로 매핑 시켰다.
	- 하단의 출력 처럼 `Connection successful` 메시지가 출력되어야 한다.
	
### PhpStorm 에서 PHP 실행하기
- File -> Settings -> Language & Frameworks -> PHP 에서 `CLI Interpreter` 를 추가해준다.
	
	![그림 8]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-8.png)

- 추가된 Docker Remote API 에서 현재 프로젝트에서 사용하는 PHP 이미지를 선택한다.

	![그림 9]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-9.png)
	
	- Docker Compose 로 생성된 이미지는 `services` 의 이름을 기반으로 생성된다.

- 아래는 `CLI Interpreters` 가 추가된 화면이다.

	![그림 10]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-10.png)
	
- PHP 의 Docker Container 경로의 Host Path 를 아래와 같이 수정하기를 눌러 프로젝트 경로로 설정해 준다.

	![그림 11]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-11.png)
	
- PHP 의 `Path mappings` 에 Host 경로와 Docker Container 의 경로를 매핑 시켜준다.

	![그림 12]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-12.png)
	
- PhpStorm 에서 스크립트를 실행할 수 있다.

	![그림 13]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-13.png)
	
### PhpStorm 에서 디버깅 하기
- 스크립트에서 디버깅을 눌렀을 때 경고가 뜨면 `Configuire xdebug.remote_host` 을 눌러 Virtualbox 의 Host-Only Ethernet Adapter 아이피로 설정해 준다.



## Docker + PHP Composer
### 프로젝트 구조
- Docker + PHP 프로젝트에 PHP Composer 를 적용한 프로젝트이다.

![그림 14]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-14.png)

### composer.json 파일

```json
{
  "name" : "test2",
  "autoload" : {
    "psr-4" : {
      "Src\\" : "src/"
    }
  },
  "require-dev" : {
    "phpunit/phpunit": "^7"
  }
}
```  

- 프로젝트의 이름과, `autuload` 설정 및 Unit 테스트 라이브러리르 추가했다.

### PhpStorm 에 PHP Composer 등록
- File - settings -> Language & Frameworks -> PHP -> Composer

	![그림 15]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-15.png)
	
- `CLI Interpreter` 설정 

	![그림 16]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-16.png)
	
- Synchronize IDE 설정

	![그림 17]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-17.png)
	
### Docker Composer 에 PHP Composer 추가
- `docker-compose.yml` 파일 내용

	```yaml
	version: '3'

	services:
	  test2nginx:
	    image: nginx:latest
	    ports:
	      - "80:80"
	    volumes:
	      - /vagrant/server/testdocker2:/opt/project
	      - ./nginx/site.conf:/etc/nginx/conf.d/site.conf
	    networks:
	      - nginx-php
	  test2php:
	    build:
	      context: ./php-fpm
	      dockerfile: Dockerfile
	    volumes:
	      - /vagrant/server/testdocker2:/opt/project
	    networks:
	      - nginx-php
	  test2composer:
	    image: composer
	    volumes:
	      - /vagrant/server/testdocker2:/app
	    command:
	      - update
	
	networks:
	  nginx-php:
	```  
	
- `docker-compose up --build` 명령어를 통해 빌드하면 PHP Composer 의 결과로 `vendor` 디렉토리가 생성된다.

	```
	[root@localhost docker]# docker-compose up --build
	Building test2php
	Step 1/3 : FROM php:7.3-fpm
	 ---> 7d3076bd7d18
	Step 2/3 : RUN pecl install xdebug   && docker-php-ext-enable xdebug
	 ---> Using cache
	 ---> e3e2494b2ae7
	Step 3/3 : COPY ./xdebug.ini /usr/local/etc/php/conf.d/
	 ---> Using cache
	 ---> 5eae354859c1
	Successfully built 5eae354859c1
	Successfully tagged docker_test2php:latest
	Starting docker_test2nginx_1    ... done
	Recreating docker_test2php_1    ... done
	Creating docker_test2composer_1 ... done
	Attaching to docker_test2nginx_1, docker_test2php_1, docker_test2composer_1
	test2php_1       | [09-Sep-2019 09:22:27] NOTICE: fpm is running, pid 1
	test2php_1       | [09-Sep-2019 09:22:27] NOTICE: ready to handle connections
	test2composer_1  | Deprecation warning: Your package name test2 is invalid, it should have a vendor name, a forward slash, and a package name. The vendor and package name can be words separated by -, . or _. The complete name should match "[a-z0-9]([_.-]?[a-z0-9]+)*/[a-z0-9]([_.-]?[a-z0-9]+)*". Make sure you fix this as Composer 2.0 will error.
	test2composer_1  | Loading composer repositories with package information
	test2composer_1  | Updating dependencies (including require-dev)
	test2composer_1  | Package operations: 28 installs, 0 updates, 0 removals
	
	생략 ..
	
	test2composer_1  | phpunit/phpunit suggests installing ext-xdebug (*)
	test2composer_1  | Writing lock file
	test2composer_1  | Generating autoload files
	docker_test2composer_1 exited with code 0
	```  

### 프로젝트 수정하기
- `src/service` 디렉토리에 `Calculate` 클래스를 만들어 준다.

	```php
	namespace Src\service;

	class Calculate
	{
	    public function plus($a, $b) {
	        return $a + $b;
	    }
	
	    public function minus($a, $b) {
	        return $a - $b;
	    }
	
	    public function multiply($a, $b) {
	        return $a * $b;
	    }
	}
	```  
	
- `index.php` 파일을 아래와 같이 수정해 준다.
	
	```php
	use Src\service\Calculate;

	require_once __DIR__ . '/vendor/autoload.php';
	
	$calculate = new Calculate();
	
	$result = $calculate->plus(1, 2);
	echo "1 + 2 = $result\n";
	
	$result = $calculate->minus(5, 2);
	echo "5 - 3 = $result\n";
	
	$result = $calculate->multiply(3, 2);	
	echo "3 * 2 = $result\n";
	```
	
- `http://<서버IP>:<서버Port>` 요청 결과는 아래와 같다.

	![그림 18]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-18.png)
	
## PHPUnit 적용하기
### 프로젝트 구조

![그림 19]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-19.png)

### PHPUnit 설정하기
- `Docker + PHP Composer` 프로젝트의 `composer.json` 에서 `require-dev` 를 통해 `phpunit` 의 의존성을 추가해 주었다.
- File -> Settings -> Language & Frameworks -> PHP -> Test Frameworks 에서 `+` 를 누르고 `PHPUnit By Remote Interpreter` 을 누른다.

	![그림 20]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-20.png)
	
- `Interpreter` 에서 Docker PHP Image 를 선택한다.

	![그림 21]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-21.png)
	
- PHPUnit Library -> Path to script 에 현재 프로젝트의 `autoload.php` 경로를 입력하고, Refresh 버튼을 누른다.

	![그림 22]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-22.png)
	
### PHP Composer 수정하기
- `composer.json` 을 아래와 같이 수정한다.

	```json
	{
	  "name" : "test2",
	  "autoload" : {
	    "psr-4" : {
	      "Src\\" : "src/"
	    }
	  },
	  "autoload-dev": {
	    "psr-4": {
	      "Test\\" : "tests/"
	    }
	  },
	  "require-dev" : {
	    "phpunit/phpunit": "^7"
	  }
	}
	```  
	
### Unit Test 코드 작성하기
- 프로젝트 루트에서 `tests` 디렉토리를 생성한다.
- `tests/service` 디렉토리 구조를 만들고 `TestCalculate` 테스트 클래스를 생성한다.

	```php
	namespace Test\service;
	
	use PHPUnit\Framework\TestCase;
	use Src\service\Calculate;
	
	class TestCalculate extends TestCase
	{
	    private $calculate;
	
	    /**
	     * @before
	     */
	    public function setUp() {
	        $this->calculate = new Calculate();
	    }
	
	    /**
	     * @test
	     */
	    public function plus_1_2_Expected_3() {
	        // when
	        $actual = $this->calculate->plus(1, 2);
	
	        // then
	        $this->assertEquals(3, $actual);
	    }
	
	    /**
	     * @test
	     */
	    public function minus_3_1_Expected_2() {
	        // when
	        $actual = $this->calculate->minus(3, 1);
	
	        // then
	        $this->assertEquals(2, $actual);
	    }
	
	    /**
	     * @test
	     */
	    public function multiply_3_2_Expected_6() {
	        // when
	        $actual = $this->calculate->multiply(3, 2);
	
	        // then
	        $this->assertEquals(6, $actual);
	    }
	}
	```  
	
- Unit Test 를 실행시키면 아래와 같은 결과를 확인 할 수 있다.

	![그림 23]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-23.png)

- 아래 처럼 디버깅도 가능하다.

	![그림 24]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-24.png)


---
## Reference
[PHPUnit "use Composer autoloader" could not parse PHPUnit version output.](https://intellij-support.jetbrains.com/hc/en-us/community/posts/115000455850-PHPUnit-use-Composer-autoloader-could-not-parse-PHPUnit-version-output-)  
[How to choose a PHP project directory structure?](https://docs.php.earth/faq/misc/structure/)  