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

	![그림 9]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-9_2.png)
	![그림 9]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-9_3.png)
	
	- Docker Compose 로 생성된 이미지는 `services` 의 이름을 기반으로 생성된다.

- 아래는 `CLI Interpreters` 가 추가된 화면이다.

	![그림 10]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-10_2.png)
	
- PHP 의 Docker Container 경로의 Host Path 를 아래와 같이 수정하기를 눌러 프로젝트 경로로 설정해 준다.

	![그림 11]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-12_2.png)
	
- PhpStorm 에서 스크립트를 실행할 수 있다.

	![그림 13]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-13.png)
	
### PhpStorm 에서 디버깅 하기
- 스크립트에서 디버깅을 눌렀을 때 경고가 뜨면 `Configuire xdebug.remote_host` 을 눌러 Virtualbox 의 Host-Only Ethernet Adapter 아이피로 설정해 준다.



## Docker + PHP Composer
- Docker + PHP 프로젝트에 PHP Composer 를 적용한 프로젝트이다.

### 프로젝트 구조

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

	![그림 16]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-16_2.png)
	
- Synchronize IDE 설정

	![그림 17]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-17_2.png)
	
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
	
- `Interpreter` 에서 Docker PHP Container 를 선택한다.

	![그림 21]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-21_2.png)
	
- PHPUnit Library -> Path to script 에 현재 프로젝트의 `autoload.php` 경로를 입력하고, Refresh 버튼을 누른다.

	![그림 22]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-22_2.png)
	
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


## MySQL 연동하기
- 기존 프로젝트에 MySQL 을 연동하고 간단한 테스트 코드를 작성한다.

### 프로젝트 구조

![그림 25]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-25.png)

### MySQL Container 설정
- `docker-compose.yml` 에 아래 내용을 추가한다.
	
	```yaml
	version: '3.7'

	services:
	  test2nginx:
	  	# 생략
	    networks:
	      # 생략
	      - php-my	# php-mysql 네트워크 추가
	  test2composer:
	  	# 생략
	  test2mysql:
	    build:
	      context: ./mysql
	    ports:
	      - "3306:3306"		# mysql 포트
	    env_file:
	      - .env			# 환경변수(초기 비밀번호 등)
	    volumes:
	      - ./mysql/conf:/etc/mysql/conf.d/source		# 커스텀 설정
	      - ./mysql/sql:/docker-entrypoint-initdb.d		# 초기 테이블 설정
	      - ./mysql-data:/var/lib/mysql					# mysql 데이터 호스트에 마운트
	    command:
	      - --default-authentication-plugin=mysql_native_password	# 비밀번호 인증 설정
	    networks:
	      - php-my		# php-mysql 네트워크 설정
	
	networks:
	  # 생략
	  php-my:		# php-mysql 네트워크 선언
	```  
	
- `docker/mysql/Dockerfile` MySQL 컨테이너 빌드 설정

	```
	FROM mysql:8.0.17   # mysql container 에서 사용할 이미지
	```  

- `docker/mysql/custom.cnf` MySQL 커스텀 설정

	```
	bind-address=0.0.0.0    # 모든 아이피에 대해서 접속 허용
	```  
	
- `docker/sql/test_table.sql` 초기 테이블 설정

	```sql
	use m_test

	CREATE TABLE IF NOT EXISTS `test_table` (
	  `id` int(10) DEFAULT 0,
	  `value` varchar(10) DEFAULT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB AUTO_INCREMENT=37 DEFAULT CHARSET=utf8
	```  
	
- `docker/.env` MySQL 환경 변수 설정

	```
	# MYSQL
	MYSQL_DATABASE=m_test
	MYSQL_USER=hello
	MYSQL_PASSWORD=hello
	MYSQL_ROOT_PASSWORD=root
	```  
	
- `docker-compose.yml` 에서 `./mysql-data:/var/lib/mysql` 마운트 시켰기 때문에 호스트 `./mysql-data` 디렉토리에 MySQL 의 데이터 파일이 저장된다.
	
### PHP Container 추가 설정
- `docker/php-fpm/Dockerfile` 내용을 아래와 같이 수정한다.

	```
	FROM php:7.3-fpm

	# xdebug
	RUN pecl install xdebug \
	  && docker-php-ext-enable xdebug
	  
	# pdo_mysql
	RUN apt-get update
	RUN docker-php-ext-install pdo_mysql
	
	COPY conf.d/ /usr/local/etc/php/conf.d/
	```  
	
### PHP Composer 추가 설정
- `compser.yml` 파일을 아래와 같이 수정한다.

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
	  },
	  "require": {
	    "ext-pdo": "*",
	    "ext-pdo_mysql": "*"
	  }
	}
	```  
	
### 소스코드

![그림 26]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-26.png)

- `Value`

	```php
	namespace Src\obj;

	class Value
	{
	    /** @var int id */
	    private $id;
	    /** @var string $value */
	    private $value;
	
	    /**
	     * @return int
	     */
	    public function getId(): int
	    {
	        return $this->id;
	    }
	
	    /**
	     * @param int $id
	     */
	    public function setId(int $id): void
	    {
	        $this->id = $id;
	    }
	
	    /**
	     * @return string
	     */
	    public function getValue(): string
	    {
	        return $this->value;
	    }
	
	    /**
	     * @param string $value
	     */
	    public function setValue(string $value): void
	    {
	        $this->value = $value;
	    }
	}
	```  
	
- `DBConnectionFactory`

	```php
	namespace Src\lib;

	class DBConnectionFactory
	{
	    /** @var \PDO $pdo */
	    private $pdo;
	    /** @var \PDOStatement $statement */
	    private $statement;
	
	    private function __construct()
	    {
	    }
	
	    private function connect() {
	        $this->pdo = new \PDO('mysql:host=test2mysql;port=3306;dbname=m_test','root', 'root');
	    }
	
	    public static function getInstance() {
	        static $instance = null;
	
	        if($instance == null) {
	            $instance = new DBConnectionFactory();
	        }
	
	        return $instance;
	    }
	
	    public function prepareStatement($query) {
	        $this->connect();
	        $this->statement = $this->pdo->prepare($query);
	    }
	
	    public function bindParam($name, $value) {
	        $this->statement->bindParam($name, $value);
	    }
	
	    public function execute() {
	        return $this->statement->execute(null);
	    }
	
	    public function fetchArray() {
	        return $this->statement->fetch(\PDO::FETCH_ASSOC);
	    }
	
	    public function affectedRows() {
	        return $this->statement->rowCount();
	    }
	}
	```  
	
	- PHP 에서 MySQL 에 접속할때 `hostname` 은 MySQL Container 이름인 `test2mysql` 이다.
	
- `ValueData`

	```php
	namespace Src\data;
	
	class ValueData
	{
	    /** @var DBConnectionFactory $db */
	    private $db;
	
	    /**
	     * ValueData constructor.
	     */
	    public function __construct()
	    {
	        $this->db = DBConnectionFactory::getInstance();
	    }
	
	
	    public function insert(Value $value) {
	        $query = <<<SQL
	        INSERT INTO test_table
	        (
	            id,
	            value
	        )
	        VALUES
	        (
	            :id,
	            :value
	        )
	SQL;
	        $this->db->prepareStatement($query);
	        $this->db->bindParam(':id', $value->getId());
	        $this->db->bindParam(':value', $value->getValue());
	        $this->db->execute();
	        $result = $this->db->affectedRows();
	
	        return $result;
	    }
	
	    public function selectById($id) {
	        $query = <<<SQL
	        SELECT
	            id,
	            value
	        FROM
	            test_table
	        WHERE
	            id = :id
	SQL;
	        $this->db->prepareStatement($query);
	        $this->db->bindParam(':id', $id);
	        $result = $this->db->execute();
	        $row = $this->db->fetchArray();
	
	        $obj = false;
	        if($result === true && $row !== false) {
	            $obj = new Value();
	            $obj->setId($row['id']);
	            $obj->setValue($row['value']);
	        }
	
	        return $obj;
	    }
	
	    public function deleteByIds(array $idArray) {
	        $query = <<<SQL
	        DELETE FROM
	            test_table
	        WHERE
	            id in (:ids)
	SQL;
	        $this->db->prepareStatement($query);
	        $this->db->bindParam(':ids', implode(',', $idArray));
	        $this->db->execute();
	        $result = $this->db->affectedRows();
	
	        return $result;
	    }
	}
	```  
	
### Unit Test 
	
```php
namespace Test\data;

class TestValueData extends TestCase
{
    private $valueData;
    private $deleteIdArray;

    /**
     * @before
     */
    public function setUp() {
        $this->valueData = new ValueData();
        $this->deleteIdArray = [];
    }

    /**
     * @after
     */
    public function after()
    {
        $this->valueData->deleteByIds($this->deleteIdArray);
    }

    /**
     * @test
     */
    public function insert_NotExistIdValue_Expected_Return1() {
        // given
        $id = 11122222;
        $value = 'v11122222';
        $obj = new Value();
        $this->deleteIdArray[] = $id;
        $obj->setId($id);
        $obj->setValue($value);

        // when
        $actual = $this->valueData->insert($obj);

        // then
        $this->assertEquals(1, $actual);
    }

    /**
     * @test
     */
    public function insert_ExistIdValue_Expected_Return0() {
        // given
        $id = 2;
        $value = 'v2';
        $this->deleteIdArray[] = $id;
        $obj = new Value();
        $obj->setId($id);
        $obj->setValue($value);
        $this->valueData->insert($obj);

        // when
        $actual = $this->valueData->insert($obj);

        // then
        $this->assertEquals(0, $actual);
    }

    /**
     * @test
     */
    public function selectById_ExistId_Expected_ReturnValue() {
        // given
        $id = 2;
        $value = 'v2';
        $this->deleteIdArray[] = $id;
        $expected = new Value();
        $expected->setId($id);
        $expected->setValue($value);
        $this->valueData->insert($expected);

        // when
        $actual = $this->valueData->selectById($id);

        // then
        $this->assertInstanceOf(Value::class, $actual);
        $this->assertEquals($expected, $actual);
    }

    /**
     * @test
     */
    public function selectById_NotExistId_Expected_ReturnFalse() {
        // given
        $id = 11232323232;

        // when
        $actual = $this->valueData->selectById($id);

        // then
        $this->assertFalse($actual);
    }
}
```  
	
- UnitTest 결과는 아래와 같다.

	![그림 27]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-27.png)
	

## Memcached 연동하기
- 기존 프로젝트에 Memcached 를 연동하고 간단한 테스트 코드를 작성한다.
	
### Memcached Container 설정
- `docker-compose.yml` 파일에 아래 내용을 추가한다.

	```yml
	version: '3.7'

	services:
	  test2nginx:
	  	# 생략
	  test2php:
	  	# 생략
	  	networks:
	      - php-mem		# php-memcached 네트워크 추가
	  test2composer:
	  	# 생략
	  test2memcached:
	    image: memcached:latest		# memcached container image
	    networks:
	      - php-mem			# php-memcached 네트워크 설정
	    command:
	      - '-m 256'		# memcached 메모리 설정
	  test2mysql:
	  	# 생략
	
	networks:
	  # 생략
	  php-mem:			# php-memcached 네트워크 선언
	```  
	
### PHP Container 추가 설정
- `docker/php-fpm/Dockerfile` 내용을 아래와 같이 수정한다.

	```
	FROM php:7.3-fpm

	# xdebug
	RUN pecl install xdebug \
	  && docker-php-ext-enable xdebug
	  
	# pdo_mysql
	RUN apt-get update
	RUN docker-php-ext-install pdo_mysql
	
	# memcached
	RUN apt-get install -y libmemcached-dev zlib1g-dev \
	    && pecl install memcached \
	    && docker-php-ext-enable memcached
	
	COPY conf.d/ /usr/local/etc/php/conf.d/
	```  
	
	
### PHP Composer 추가 설정
- `compser.yml` 파일을 아래와 같이 수정한다.

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
	  },
	  "require": {
	    "ext-pdo": "*",
	    "ext-pdo_mysql": "*",
    	"ext-memcached": "*"
	  }
	}
	```  
	
### 소스코드
- `MemcachedConnectionFactory`

	```php
	namespace Src\lib;	

	class MemcachedConnectionFactory
	{
	    /** @var \Memcached $memcached */
	    protected $memcached;
	
	    private function __construct()
	    {
	        $this->memcached = new \Memcached('docker-test');
	        $this->memcached->addServer('test2memcached', 11211);
	    }
	
	    public static function getInstance()
	    {
	        static $instance = null;
	
	        if ($instance === null) {
	            $instance = new MemcachedConnectionFactory();
	        }
	
	        return $instance;
	    }
	
	    public function get($key)
	    {
	        return $this->memcached->get($key);
	    }
	
	    public function set($key, $value)
	    {
	        return $this->memcached->set($key, $value);
	    }
	
	    public function delete($key) {
	        return $this->memcached->delete($key);
	    }
	
	    public function deletes(array $keys) {
	        return $this->memcached->deleteMulti($keys);
	    }
	}
	```  
	
	- PHP 에서 Memcached 에 접속할때 `hostname` 은 Memcached Container 이름인 `test2memcached` 이다.
	
- `ValueDao`

	```php
	namespace Src\dao;
	
	class ValueDao
	{
	    public function set($key, $value)
	    {
	        // nothing, just set data
	        return MemcachedConnectionFactory::getInstance()->set($key, $value);
	    }
	
	    public function get($key)
	    {
	        // nothing, just get data
	        return MemcachedConnectionFactory::getInstance()->get($key);
	    }
	}
	```  
	
### Unit Test

```php
namespace Test\dao;

class TestValueDao extends TestCase
{
    private $valueDao;
    private $deleteIdArray;

    /**
     * @before
     */
    public function setUp() {
        $this->valueDao = new ValueDao();
        $this->deleteIdArray = [];
    }

    /**
     * @after
     */
    public function after() {
        MemcachedConnectionFactory::getInstance()->deletes($this->deleteIdArray);
    }

    /**
     * @test
     */
    public function set_Expected_ReturnTrue() {
        // given
        $obj = new Value();
        $obj->setId(1);
        $this->deleteIdArray[] = 1;
        $obj->setValue('v1');

        // when
        $actual = $this->valueDao->set(1, $obj);

        // then
        $this->assertTrue($actual);
    }

    /**
     * @test
     */
    public function get_ExistId_Expected_ReturnValue() {
        // given
        $id = 1;
        $this->deleteIdArray[] = $id;
        $expected = new Value();
        $expected->setId($id);
        $expected->setValue('v1');
        $this->valueDao->set($id, $expected);

        // when
        $actual = $this->valueDao->get($id);

        // then
        $this->assertInstanceOf(Value::class, $actual);
        $this->assertEquals($expected, $actual);
    }

    /**
     * @test
     */
    public function get_NotExistId_Expected_ReturnFalse() {
        // given
        $id = 1;

        // when
        $actual = $this->valueDao->get($id);

        // then
        $this->assertFalse($actual);
    }
}
```  

- UnitTest 결과는 아래와 같다.

	![그림 28]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-28.png)
	

## Redis 연동하기

- 기존 프로젝트에 Redis 를 연동하고 간단한 테스트 코드를 작성한다.
	
### 프로젝트 구조

![그림 29]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-29.png)

### Redis Container 설정
- `docker-compose.yml` 파일에 아래 내용을 추가한다.

	```yml
	version: '3.7'

	services:
	  test2nginx:
	  	# 생략
	  test2php:
	  	# 생략
	  	networks:
	      - php-redis		# php-redis 네트워크 추가
	  test2composer:
	  	# 생략
	  test2memcached:
	  	# 생략
	  test2mysql:
	  	# 생략
	  test2redis:
	    build:
	      context: .			# redis container 빌드
	      dockerfile: redis/Dockerfile
	    ports:
	      - "6377:6377"		# redis 포트
	    networks:
	      - php-redis		# php-redis 네트워크 설정
	    volumes:
	      - ./redis-data:/data		# redis 데이터 파일 호스트에 마운트
	
	networks:
	  # 생략
	  php-redis:			# php-redis 네트워크 선언
	```  
	
- `docker/redis/Dockerfile` Redis 컨테이너 빌드 설정

	```
	FROM redis:latest

	COPY ./redis/redis.conf /usr/local/etc/redis/redis.conf     # 설정파일 복사
	CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]     # redis server start
	```  
	
- `docker/redis/redis.conf` Redis Server 커스텀 설정

	```conf
	# 생략
	bind 0.0.0.0
	port 6377
	# 생략
	```  
	
	- [redis.conf](http://download.redis.io/redis-stable/redis.conf) 를 다운 받고 수정해서 사용한다.
	
- `docker-compose.yml` 에서 `./redis-data:/data` 마운트 시켰기 때문에 호스트 `./redis-data` 디렉토리에 Redis 의 데이터 파일이 저장된다.
	
	
### PHP Container 추가 설정
- `docker/php-fpm/Dockerfile` 내용을 아래와 같이 수정한다.

	```FROM php:7.3-fpm

	# xdebug
	RUN pecl install xdebug \
	  && docker-php-ext-enable xdebug
	
	# pdo_mysql
	RUN apt-get update
	RUN docker-php-ext-install pdo_mysql
	
	# redis, memcached
	RUN pecl install redis \
	    && docker-php-ext-enable redis \
	    && :\
	    && apt-get install -y libmemcached-dev zlib1g-dev \
	    && pecl install memcached \
	    && docker-php-ext-enable memcached
	
	COPY conf.d/ /usr/local/etc/php/conf.d/
	```  
	
	
### PHP Composer 추가 설정
- `compser.yml` 파일을 아래와 같이 수정한다.

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
	  },
	  "require": {
	    "ext-pdo": "*",
	    "ext-pdo_mysql": "*",
	    "ext-memcached": "*",
	    "ext-redis": "*"
	  }
	}
	```  
	
### 소스코드
- `RedisConnectionFactory`

	```php
	namespace Src\lib;	
	
	class RedisConnectionFactory
	{
	    /** @var \Redis $redis */
	    private $redis;
	
	    private function __construct()
	    {
	        $this->redis = new \Redis();
	        $this->redis->pconnect('test2redis', 6377);
	        $this->redis->setOption(\Redis::OPT_SERIALIZER, \Redis::SERIALIZER_PHP);
	    }
	
	    public static function getInstance() {
	        static $instance = null;
	
	        if($instance === null) {
	            $instance = new RedisConnectionFactory();
	        }
	
	        return $instance;
	    }
	
	    public function set($id, $value) {
	        return $this->redis->set($id, $value);
	    }
	
	    public function get($id){
	        return $this->redis->get($id);
	    }
	
	    public function deletes(array $ids) {
	        return $this->redis->delete($ids);
	    }
	}
	```  
	
	- PHP 에서 Redis 에 접속할때 `hostname` 은 Redis Container 이름인 `test2redis` 이다.
	
- `ValueRedisDao`

	```php
	namespace Src\dao;
	
	class ValueRedisDao
	{
	    public function set(int $id, Value $value) {
	        return RedisConnectionFactory::getInstance()->set($id, $value);
	    }
	
	    public function get(int $id) {
	        return RedisConnectionFactory::getInstance()->get($id);
	    }
	}
	```  
	
### Unit Test

```php
namespace Test\dao;

class TestValueRedisDao extends TestCase
{
    private $valueRedisDao;
    private $deleteIdArray;

    /**
     * @before
     */
    public function setUp() {
        $this->valueRedisDao = new ValueRedisDao();
        $this->deleteIdArray = [];
    }

    /**
     * @after
     */
    public function after() {
        RedisConnectionFactory::getInstance()->deletes($this->deleteIdArray);
    }

    /**
     * @test
     */
    public function set_Expected_ReturnTrue() {
        // given
        $obj = new Value();
        $obj->setId(1);
        $this->deleteIdArray[] = 1;
        $obj->setValue('v1');

        // when
        $actual = $this->valueRedisDao->set(1, $obj);

        // then
        $this->assertTrue($actual);
    }

    /**
     * @test
     */
    public function get_ExistId_Expected_ReturnValue() {
        // given
        $id = 1;
        $this->deleteIdArray[] = $id;
        $expected = new Value();
        $expected->setId($id);
        $expected->setValue('v1');
        $this->valueRedisDao->set($id, $expected);

        // when
        $actual = $this->valueRedisDao->get($id);

        // then
        $this->assertInstanceOf(Value::class, $actual);
        $this->assertEquals($expected, $actual);
    }

    /**
     * @test
     */
    public function get_NotExistId_Expected_ReturnFalse() {
        // given
        $id = 1;

        // when
        $actual = $this->valueRedisDao->get($id);

        // then
        $this->assertFalse($actual);
    }
}
```  

- UnitTest 결과는 아래와 같다.

	![그림 30]({{site.baseurl}}/img/php/practice-vagrant-docker-php-phpstorm-30.png)
	
	
	
		

---
## Reference
[PHPUnit "use Composer autoloader" could not parse PHPUnit version output.](https://intellij-support.jetbrains.com/hc/en-us/community/posts/115000455850-PHPUnit-use-Composer-autoloader-could-not-parse-PHPUnit-version-output-)  
[How to choose a PHP project directory structure?](https://docs.php.earth/faq/misc/structure/)  