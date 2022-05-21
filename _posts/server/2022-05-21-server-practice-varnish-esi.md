--- 
layout: single
classes: wide
title: "[Varnish] Varnish ESI"
header:
  overlay_image: /img/server-bg.jpg
excerpt: 'Varnish 에서 동적 페이지를 위한 기능 Edge Size Includes 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Server
tags:
  - Server
  - Varnish
  - HTTP Accelerator
  - Web Caching
  - ESI
  - Tomcat Log
toc: true
use_math: true
---  

## Varnish ESI
[ESI(Ege Side Includes)](https://varnish-cache.org/docs/6.0/users-guide/esi.html)
는 동적 컨텐츠에 적용할 수 있는 캐시 기능이다. 
동적 컨텐츠를 여러 부분으로 분리해서 조각 단위로 캐싱을 수행하는 방식이다. 

하나의 페이지가 아래와 같은 구조로 구성된다고 가정해 보자. 

![그림 1]({{site.baseurl}}/img/server/practice-varnish-esi-1.png)

각 영역은 다음과 같은 성격을 지니고 있는 상태이다.  

- Personal Content : 사용자 별로 다른 컨텐츠를 보여주고 있고, 캐싱은 수행하지만 짧은 시간의 캐싱이 필요하다. 
- Dynamic Content : 모든 사용자에게 동일한 컨텐츠를 제공하지만 페이지 접근시마다 항상 다른 컨텐츠를 노출해야 하기 때문에 캐싱을 적용해서는 안된다. 
- Main Content : 사용자가 요청한 컨텐츠를 보여주는 영역이고, 정적인 성격이 강하기 때문에 긴 시간 캐싱이 필요하다. 

위 페이지는 총 3개의 영역으로 구성돼 있지만, 
영역별 성격의 차리로 캐싱에 대한 요구사항이 각기 다른 상황이다. 
이런 상황에서 적용해 볼 수 있는 것이 바로 `ESI` 이다.  

### ESI 사용
`ESI` 사용을 위해서는 `Varnish` 설정 파일인 `VCL` 파일에 아래와 같은 설정이 필요하다. 

```
# varnish 3 이하 
sub vcl_fetch {
  set beresp.do_esi = true;
}

# varnish 4 이상
sub vcl_backend_response {
  set beresp.do_esi = true;
}
```  

그리고 `ESI` 의 실제 적용은 아래 2개의 태그를 사용한다. 

- `esi:include` : `ESI` 가 적용될 영역을 추가한다. 
- `esi:remove` : `Varnish` 에서 `ESI` 가 활성화 되지 않았을 때 `fallback` 으로 사용될 결과를 추가한다. 

그리고 앞서 설명한 3개 영역을 담고 있는 파일을 `index.html` 이라고 할때 아래와 같이 작성 할 수 있다.   

```html
<body>
    <esi:include src="http://localhost/personal"/>
    <esi:include src="http://localhost/dynamic"/>
    <esi:include src="http://localhost/article"/>
    <esi:remove>
        No ESI!
    </esi:remove>
</body>
```  

`ESI` 설정이 활성화된 상태라면 `personal`, `dynamic`, `article` 3개의 영역이 `Varnish` 설정에 따라 캐싱된 결과로 페이지가 구성될 것이다. 
하지만 `ESI` 설정이 비활성화된 상태라면 페이지에는 `No ESI!` 만 노출 된다.  

## 예제
`Varnish ESI` 에 대한 예제는 아래와 같은 환경과 스펙으로 구성한다. 

- `Docker 20.10.14`
- `docker-compose 1.29.2`
- `Spring Boot 2.6.4`
- `Thymeleaf`
- `Varnish 6.6`

`Varnish - Spring Boot` 의 구조로 
`Varnish` 의 원본서버로 `Spring Boot` 앱을 사용한다.  

예제에서 구현할 페이지는 앞서 설명한 `Personal`, `Dynamic`, `Main` 3개의 영역으로 구성된 페이지이고, 
각 영역의 성격 또한 설명한 내용을 바탕으로 한다.  

### Spring Boot 구성 및 빌드
먼저 `Spring Boot` 관련 코드는 아래와 같다. 

- `build.gradle`
  - `jib` 를 사용해서 `Docker`, `docker-compose` 에서 테스트 환경 구성으로 사용할 이미지를 빌드한다. 

```groovy
plugins {
    id 'org.springframework.boot' version '2.6.4'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
    id 'com.google.cloud.tools.jib' version '3.2.0'
}

group 'com.windowforsun.varnish.esitest'
version '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {
    implementation "org.springframework.boot:spring-boot-starter-web"
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation "org.springframework.boot:spring-boot-devtools"
    implementation 'org.springframework.boot:spring-boot-starter-tomcat'
    implementation 'ch.qos.logback:logback-access'
    implementation 'nz.net.ultraq.thymeleaf:thymeleaf-layout-dialect'

}

test {
    useJUnitPlatform()
}

jib {
    from {
        image = "openjdk:11-jre-slim"
    }
    to {
        image = "esi-test"
        tags = ["${project.version}".toString()]
    }
    container {
        mainClass = "com.windowforsun.varnish.esitest.EsiTestApplication"
        ports = ["8080"]
    }
}
```  

- `index.html` 은 `esi:include` 태그를 사용해서 3개의 영역을 포함하는 파일이다.  

```html
<!-- resources/templates/index.html -->

<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
	<meta charset="UTF-8">
	<title>ESI Test</title>
</head>
<body>
<!---->
<div style="height: 60vh">
	<esi:include th:src="${personalUrl}"/>
	<esi:include src="http://localhost:8080/dynamic"/>
	<esi:include th:src="${articleUrl}"/>
	<esi:remove>
		<div style="background-color: aqua; height: 100%">
			<span>No ESI!!</span>
		</div>
	</esi:remove>
</div>
</body>
</html>
```  

- `peronal.html` 은 사용자 마다 제공되는 컨텐츠로 사용자 이름과 데이터 업데이트 시간을 출력한다.  

```html
<!-- resources/templates/fragments/personal.html -->

<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<div style="background-color: burlywood; height: 20%">
  <div>Personal Content(Short Cache)</div>
  <div><span>name : </span><span th:text="${name}"/></div>
  <div><span>update : </span><span th:text="${current}"/></div>
</div>
</html>
```  

- `dynamic.html` 은 모든 사용자에게 동일하지만 동적인 콘텐츠로 현재시간을 출력한다. 

```html
<!-- resources/templates/fragments/dynamic.html -->

<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<div style="background-color: brown; height: 40%">
  <div>Dynamic Content(No Cache)</div>
  <div><span>current : </span><span th:text="${current}"/></div>
</div>
</html>
```  

- `article.html` 은 사용자가 요청한 컨텐츠로 아티클 아이디와 마지막 업데이트 시간을 출력한다. 

```html
<!-- resources/templates/fragments/article.html -->

<!DOCTYPE html>
<html lang="en">
<div style="background-color: cornflowerblue; height: 40%">
  <div>Main Content(Long Cache)</div>
  <div><span>update : </span><span th:text="${current}"/></div>
  <div><span>Main Article, articleId : </span><span th:text="${articleId}"/></div>
</div>
</html>
```  

- `Spring Boot` 애플리케이션 코드에는 설정 코드와 각 페이지 요청을 처리하는 `Controller` 구현으로 구성돼 있다.  
  - 각 사용자의 구분은 `QueryString` 의 `name` 으로 사용한다. 
  - 사용자가 요청하는 `Main Content` 의 구분은 `QueryString` 의 `articleId` 로 사용한다. 

```java
@SpringBootApplication
@Controller
public class EsiTestApplication {
    public static void main(String[] args) {
        SpringApplication.run(EsiTestApplication.class, args);
    }

    // Tomcat Access Log STDOUT 설정
    @Bean
    public TomcatServletWebServerFactory servletWebServerFactory() {
        TomcatServletWebServerFactory tomcatServletWebServerFactory = new TomcatServletWebServerFactory();
        tomcatServletWebServerFactory.addContextValves(new LogbackValve());

        return tomcatServletWebServerFactory;
    }

    @GetMapping("/personal")
    public ModelAndView personal(@RequestParam(name = "name", required = false) String name) {
        ModelAndView mv = new ModelAndView();
        mv.addObject("current", LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        mv.addObject("name", name);
        mv.setViewName("fragments/personal");

        return mv;
    }

    @GetMapping("/dynamic")
    public ModelAndView dynamic() {
        ModelAndView mv = new ModelAndView();
        mv.addObject("current", LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        mv.setViewName("fragments/dynamic");

        return mv;
    }

    @GetMapping("/article")
    public ModelAndView article(@RequestParam(name = "articleId", required = false) String articleId) {
        ModelAndView mv = new ModelAndView();
        mv.addObject("current", LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        mv.addObject("articleId", articleId);
        mv.setViewName("fragments/article");

        return mv;
    }

    @GetMapping(value = {"/index", "noesiindex"})
    public ModelAndView index(
            @RequestParam(name = "name") String name,
            @RequestParam(name = "articleId", required = false) String articleId
    ) {
        ModelAndView mv = new ModelAndView();
        mv.addObject("personalUrl", "http://localhost:8080/personal?name=" + name);
        mv.addObject("articleUrl", "http://localhost:8080/article?articleId=" + articleId);
        mv.setViewName("index");

        return mv;
    }
}
```  

- `application.yaml`

```yaml
spring:
  devtools:
    livereload:
      enabled: true
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
    cache: false
    check-template: true

server:
  tomcat:
    accesslog:
      enabled: true
```  

- `Spring Boot` 프로젝트 루트 경로에서 아래 명령어를 통해 `Spring Boot` 애플리케이션 이미지 빌드가 가능하다.  

```bash
$ ./gradlew jibDockerBuild

.. 생략 ..

Built image to Docker daemon as esi-test, esi-test:1.0-SNAPSHOT
Executing tasks:
[==============================] 100.0% complete


BUILD SUCCESSFUL in 9s
3 actionable tasks: 2 executed, 1 up-to-date
```  


### Varnish 구성 및 빌드
`Varnish` 설정 파일인 `esi.vcl` 내용은 아래와 같다.  

```
# docker/varnish/esi.vcl

vcl 4.0;

backend default {
  .host = "esi-test";
  .port = "8080";
}

sub vcl_backend_response {
    if(bereq.url ~ "^/noesiindex") {
        set beresp.do_esi = false;
    } else {
        set beresp.do_esi = true;
    }

    # no cache
    if(bereq.url ~ "^/dynamic") {
        set beresp.uncacheable = true;

        return (deliver);
    }

    # short cache
    if(bereq.url ~ "^/personal") {
        set beresp.ttl = 12s;
    }
}

sub vcl_deliver {
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
    return (deliver);
}
```  

`noesiindex` 경로로 들어오는 요청만 `ESI` 를 비활성화 하고, 
그외 모든 경로는 `ESI` 를 활성화 한다. 
추후 `noesiindex` 에 요청해서 `esi:remove` 에 대한 테스트를 수행할 계획이다.  

그리고 캐시에 대한 설정은 `Main Content` 영역에 해당하는 요청은 `Varnish` 기본 `TTL` 인 `120s` 를 사용한다. 
`Personal Content` 영역에 해당하는 요청은 비교적 짧은 캐싱을 위해 `TTL` 을 `12s` 로 설정한다. 
`Dynamic Content` 영역의 요청은 캐싱이 수행되면 안되기 때문에 캐싱 동작을 비활성화 했다.  

다음은 위 설정파일을 바탕으로 `Varnish` 이미지를 빌드하는 `Dockerfile` 내용이다. 

```dockerfile
# docker/varnish/Dockerfile

FROM varnish:6.6

RUN apt -y update
RUN apt -y install curl

COPY esi.vcl /etc/varnish/default.vcl
```  

### docker-compose 구성 및 실행
`docker-compose` 는 `Varnish` 서버와 원본 서버 역할을 하는 `Spring Boot` 앱으로 구성된다.  

```yaml
# docker/docker-compose.yaml

version: '3.7'

services:
  # spring boot app
  esi-test:
    image: esi-test
    ports:
      - 8081:8080

  # varnish server
  varnish-esi:
    container_name: "varnish-esi"
    hostname: "varnish-esi"
    build: ./varnish
    ports:
      - 80:80
```  

`docker-compose.yaml` 실행은 아래 명령으로 가능하다.  

```bash
$ docker-compose up --build
Building varnish-esi
Sending build context to Docker daemon  3.584kB
Step 1/4 : FROM varnish:6.6
 ---> 820ecc12d5ee
Step 2/4 : RUN apt -y update
 ---> Using cache
 ---> 27a81ac34e02
Step 3/4 : RUN apt -y install curl
 ---> Using cache
 ---> 9e7e5619cf86
Step 4/4 : COPY esi.vcl /etc/varnish/default.vcl
 ---> Using cache
 ---> 838564104c43
Successfully built 838564104c43
Successfully tagged docker_varnish-esi:latest
Starting varnish-esi       ... done
Starting docker_esi-test_1 ... done
Attaching to docker_esi-test_1, varnish-esi

.. 생략 ..

```  

`http://{host}/index?name={username}&articleId={articleId}` 로 요청하면 구성된 페이지 확인이 가능하다. 

### 테스트
`name` 은 `test1` 으로 `articleId` 는 `0001` 로 요청하면 아래와 같이 
`Personal Content` 영역에 `name`은 `test1` 이 노출되고, 
`Main Content` 영역에 `articleId` 는 `0001` 이 노출된다. 
그리고 모두 새로 요청한 결과 이기 때문에 모든 시간 값이 `Dynamic Content` 의 시간 값과 동일한 것을 확인 할 수 있다.  


![그림 1]({{site.baseurl}}/img/server/practice-varnish-esi-2.png)  

위 상태에서 `articleId` 만 `0010` 으로 변경해서 요청하면 아래와 같이
`Personal Content` 영역은 이전 요청 캐시 결과를 사용하기 때문에 `update` 시간 값이 동일한 것을 확인 할 수 있다. 
하지만 `Main Content` 영역의 `articleId` 는 `0001` 이고 `update` 시간 값이 최신 값인 `Dynamic Content` 의 시간 값과 동일한 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/server/practice-varnish-esi-3.png)  

이번엔 `name` 을 `test2` 로 하고 `articleId` 는 `0010` 으로 변경해서 요청하면 아래와 같이 
`Personal Content` 영역의 `name` 은 `test2` 로 변경되고, `update` 시간 값 또한 최신 값인 `Dynamic Content` 의 시간 값과 동일하다. 
하지만 `Main Content` 는 이전에 캐싱된 결과를 사용하고 있는 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/server/practice-varnish-esi-4.png)  

그리고 `Personal Content` 의 만료 시간인 12초 후에 다시 `name` 은 `test1`, `articleId` 는 `0001` 로 요청하면, 
`Personal Content` 의 `update` 시간 값은 최신 값인 `Dynamic Content` 의 시간 값과 동일 하지만 `Main Content` 는 아직 `TTL` 만료 전이기 때문에 이전 캐싱을 사용한 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/server/practice-varnish-esi-5.png)  

마지막으로 `noesiindex` 로 요청하면 아래와 같이 `esi:remove` 태그에 작성한 내용이 출력 된 것을 확인 할 수 있다.  

![그림 1]({{site.baseurl}}/img/server/practice-varnish-esi-6.png)


`Varnish` 의 `ESI` 기능을 사용해서 한 페이지에서 영역의 특징에 맞는 캐싱을 적용해 보았다. 
이를 사용해서 기존 보다 높은 가용량을 확보 할 수는 있겠지만, 
페이지 구현 내용에 `Varnish` 의 태그가 들어가게 된다는 점과 기존에 구성된 페이지를 다시 캐싱 관점에서 다시 구분하고 구조화 해야 한다는 점이 있다. 
이는 기존에 사용 중인 `Tiles`, `freemaker`, `velocity` 등 처럼 `Template Engine` 등의 구조에도 큰 변화가 필요할 수 있기 때문에 전반적인 구조 개편이 필요할 수 있다. 
하지만 분명한 점은 잘 구성한다면, 정적인 영역은 모두 캐시를 활용하고 동적인 영역만 짧은 `TTL` 혹은 캐시를 사용하지 않는 방법으로 추가 가용량을 확보 할 수 있다는 점이다.  



---
## Reference
[Controlling The Cache: Using Edge Side Includes In Varnish](https://www.smashingmagazine.com/2015/02/using-edge-side-includes-in-varnish/)  
[Content composition with Edge Side Includes](https://varnish-cache.org/docs/6.0/users-guide/esi.html)  
    