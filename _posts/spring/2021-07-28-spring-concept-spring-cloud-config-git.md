--- 
layout: single
classes: wide
title: "[Spring 개념] Spring Cloud Config for Git Repository"
header:
  overlay_image: /img/spring-bg.jpg
excerpt: 'Spring Cloud Config 의 소개와 Git 사용한 활용방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Boot
    - Spring Cloud Config
toc: true
use_math: true
---  

## Spring Cloud Config
`Spring Cloud Config` 는 원격으로 `application.yaml`, `application.properties`(설정 파일) 의 설정을 
배포없이 수정하고 갱신할 수 있다. 
`Spring Boot` 애플리케이션에서 설정파일에는 많은 정보가 담겨있다. 
`DB`, `Redis` 등 외부 저장소에 대한 정보, 애플리케이션에서 사용하는 설정 값 등이 그 예가 될 수 있다.  

만약 애플리케이션의 특정 기능이 `On` 인 설정에서, `Off` 설정으로 변경 해야된다고 가정해보자.
가장 보편적인 방법은 코드에 설정을 `Off` 로 변경하고, 빌드, 배포를 수행하는 절차가 있을 것이다. 
설정 값 변경을 위해 코드 수정 부터 배포까지 많은 절차를 거쳐야만 실제 리얼환경에 적용이 가능하다. 
그리고 `Blue/Green` 등의 배포 방식을 적용했더라도, 배포를 하는 것은 다양한 상황을 고려해야하는 민감한 작업이다. 
단순히 설정 값을 `Off` 로 변경을 위해서만 배포 한번을 수행하기에는 부담이 클수 있다. 
만약 다음에 다시 `On` 을 해야 한다면 다시 배포해야 하고, 빈번하면 배포 횟수는 늘어날 것이다.  

이러한 상황에서 효율적으로 적용할 수 있는 방법이 바로 `Spring Cloud Config` 이다. 
별도의 배포 없이 설정을 변경해주고 갱신 작업만 이뤄지면 모든 애플리케이션에 변경된 설정이 적용되고, 
배포없이 아주 `Graceful` 하게 `On` 에서 `Off` 로 설정을 변경할 수 있다.  

### Spring Cloud Config for Git 구조
먼저 `Git Repository` 를 설정을 저장하는 저장소로 사용하는 `Spring Cloud Config` 에 대하서 알아 볼 것이다. 
예제를 진행하기 전에 아래 그림을 통해 전체적인 구조를 파악해 보자.  

![그림 1]({{site.baseurl}}/img/spring/concept-spring-cloud-config-git-1.png)

크게 아래와 같은 구성이 필요하다. 
1. 설정 저장소 `Git Repository`
1. `Spring Cloud Config Client`(서비스 애플리케이션)
1. `Spring Cloud Config Server`(설정 전파 용 `Server`)

현재 구조를 잘 살펴보면 한가지 비효율적인 부분이 있다. 
사용자가 `Spring Cloud Config Client` 갱신을 위해 `N` 번 갱신 호출을 해주어야 한다는 점이다. 
이부분은 추후에 개선해 나가는 방향으로 두고 기억에 남겨 두도록 한다.  

### 설정 저장소
설정 저장소로 사용할 `Git Repository` 를 `Github`, `Bitbucket` 등 외부에서 접근가능한 저장소에 생성해 준다. 
필자는 테스트를 위해 `public-cloud-config`, `private-cloud-config` 두 개의 저장소를 생성했다. 
저장소에 대한 설명과 각 저장소에 있는 설정 파일 구성 및 내용은 아래와 같다. 

> 설정 파일 이름 규칙은 <애플리케이션 이름>-<환경>.yaml 이다. 

- `public-cloud-config`(공개된 설정)

    ```bash
    $ tree .
    .
    ├── lowercase-dev.yaml
    |		visibility: public
    |		group:
    |			key1: dev1
    |			key2: dev2
    ├── lowercase-real.yaml
    |		visibility: public
    |		group:
    |			key1: real1
    |			key2: real2
    ├── uppercase-dev.yaml
    |		visibility: PUBLIC
    |		group:
    |			key1: DEV1
    |			key2: DEV2
    └── uppercase-real.yaml
    		visibility: PUBLIC
    		group:
    			key1: REAL1
    			key2: REAL2
    ```  

- `private-cloud-config`(비공개된 설정)

    ```bash
    $ tree .
    .
    ├── lowercase-dev.yaml
    |		visibility: private
    |		group:
    |			key1: dev1
    |			key2: dev2
    ├── lowercase-real.yaml
    |		visibility: private
    |		group:
    |			key1: real1
    |			key2: real2
    ├── uppercase-dev.yaml
    |		visibility: PRIVATE
    |		group:
    |			key1: DEV1
    |			key2: DEV2
    └── uppercase-real.yaml
    		visibility: PRIVATE
    		group:
    			key1: REAL1
    			key2: REAL2
    ```  
  
### Spring Cloud Config Server
#### build.gradle
  - 아래와 같이 `Spring Cloud Config Server` 의존성을 추가해 준다.  
	
```groovy
plugins {
    id 'java'
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
    id 'org.springframework.boot' version '2.4.2'
}

ext {
    springCloudVersion = '2020.0.1'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'
group 'com.windowforsun.springcloudconfig.configserver'
version '1.0-SNAPSHOT'


repositories {
    mavenCentral()
}

dependencies {
    compile('org.springframework.cloud:spring-cloud-config-server')
    implementation ('org.springframework.boot:spring-boot-starter-actuator')

}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
test {
    useJUnitPlatform()
}
```  

#### GitConfigServerApplication
  - `Spring Cloud Config Server` 의 메인 클래스를 아래와 같이 정의해 준다.
  - `@EnableConfigServer` 을 선언이 필요하다. 
	
```java
@SpringBootApplication
@EnableConfigServer
public class GitConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(GitConfigServerApplication.class, args);
    }
}
```  

#### application.yaml
- 저장소가 `Public` 이라면 아래와 같이 간단하게 가능하다. 
	
	```groovy
	server:
		port: 8070
	spring:
		application:
			name: config-server
		cloud:
			config:
				server:
					git:
						# 기본 브랜치 설정
						default-label: main
						# 저장소 주소
						uri: https://github.com/windowforsun/public-cloud-config
	```  
 
- 저장소가 `Private` 인 경우 아래 절차를 따른다.
	- 아래 명령어로 `SSH Key` 를 생성하면, `ssh-key`, `ssh-key.pub` 파일 2개가 생성된다. 
	
		```bash
		$ ssh-keygen -m PEM -t rsa -b 4096 -C "git-cloud-config"
		Generating public/private rsa key pair.
		Enter file in which to save the key (/home/windowforsun/.ssh/id_rsa): ./ssh-key
		Enter passphrase (empty for no passphrase):
		Enter same passphrase again:
		Your identification has been saved in ./ssh-key
		Your public key has been saved in ./ssh-key.pub
		The key fingerprint is:
		SHA256:wcf0xvtkBI4lLwUIpp1t0Q/QFyt83lzynyKxqa4VeTw git-cloud-config
		The key's randomart image is:
		+---[RSA 4096]----+
		|      o.o=+.=.   |
		|     + +.=+X.o   |
		|    . o = B+O o .|
		|       . oo*.= + |
		|        So Eo = .|
		|          o =+  o|
		|         . + ....|
		|        . . . .  |
		|       .oo       |
		+----[SHA256]-----+  
		```
	
	- `ssh-key.pub` 파일 전체 내용을 아래 링크 가이드에 따라 `GitHub` 의 `SSH key or Add SSH key` 로 등록해 준다.
	  - [링크](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
	- `ssh-key` 는 비밀키 이므로 아래와 같이 `application.yaml` 에 등록해주고 `저장소 주소` 는 `SSH` 용 주소로 변경해 준다. 
	
		```yaml
		server:
		  port: 8070
		spring:
		  application:
			name: config-server
		  cloud:
			config:
			  server:
				git:
				  default-label: main
				  # SSH 용 주소
				  uri: git@github.com:windowforsun/private-cloud-config.git
				  # ssh-key 파일 전체 내용
				  private-key: |
					  -----BEGIN RSA PRIVATE KEY-----
					  MIIJJwIBAAKCAgEAuK7cnfwiq24eU0fQ+N2e7PtaTUXtZ+6yriQf4LiAtK+pmrUM
					  
					  .. 생략 ..
					  
					  NwPm0xPRw/uv0yiycSkvmmpgrMB621CqQi1NuSH4h+1VymZugVbQSLTv7A==
					  -----END RSA PRIVATE KEY-----
		```  
  
#### 테스트
저장소까지 등록이 되었으면, 애플리케이션으 실행한다. 
`Spring Cloud Config Server` 에서 관리되고 있는 저장소의 설정정보를 확인하기 위해서는 아래와 같은 규칙으로 요청하면 된다.  

```bash
http://<도메인>:<포트>/<애플리케이션 이름>/<환경>
```  

예시로 몇개의 호출 결과를 나열하면 아래와 같다. 

```bash
$ curl http://172.21.0.1:8070/lowercase/dev | jq ''
{
  "name": "lowercase",
  "profiles": [
    "dev"
  ],
  "label": null,
  "version": "61587b224750a91400cb60da4b85ab4dd2382340",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/windowforsun/public-cloud-config/lowercase-dev.yaml",
      "source": {
        "visibility": "public",
        "group.key1": "dev1",
        "group.key2": "dev2"
      }
    }
  ]
}

$ curl http://172.21.0.1:8070/uppercase/real | jq ''
{
  "name": "uppercase",
  "profiles": [
    "real"
  ],
  "label": null,
  "version": "61587b224750a91400cb60da4b85ab4dd2382340",
  "state": null,
  "propertySources": [
    {
      "name": "https://github.com/windowforsun/public-cloud-config/uppercase-real.yaml",
      "source": {
        "visibility": "PUBLIC",
        "group.key1": "REAL1",
        "group.key2": "REAL2"
      }
    }
  ]
}
```  

  

### Spring Cloud Config Client
마지막으로 `Spring Cloud Config` 를 실제로 사용하게 될 클라이언트만 구성해주면 된다. 
이후 구성하는 클라이언트의 몇개의 설정 값은 로컬에 존재하는 설정파일이 아닌, 
외부 저장소(`Git Repository`)에 저장된 설정값을 불러와 애플리케이션이 동작 가능하고 
필요에 따라 설정 값을 배포 없이 변경할 수도 있다.  

#### build.gradle

- `Spring Boot` 용 `Spring Cloud Config` 의존성을 추가한다. 
- 애플리케이션 실행 도중 갱신작업을 위해 `Actuator` 도 함께 의존성을 추가한다.  

```groovy
plugins {
    id 'java'
    id "io.spring.dependency-management" version "1.0.10.RELEASE"
    id 'org.springframework.boot' version '2.4.2'
}

apply plugin: 'java'
apply plugin: 'io.spring.dependency-management'
group 'com.windowforsun.springcloudconfig.gitrepo'
version '1.0-SNAPSHOT'


ext {
    springCloudVersion = '2020.0.1'
}

repositories {
    mavenCentral()
}

dependencies {
    implementation ('org.springframework.boot:spring-boot-starter-actuator')
    implementation ('org.springframework.boot:spring-boot-starter-web')
    implementation('org.springframework.cloud:spring-cloud-starter-config')
    implementation group: 'org.eclipse.jgit', name: 'org.eclipse.jgit', version: '5.12.0.202106070339-r'
    implementation group: 'commons-io', name: 'commons-io', version: '2.11.0'
    
    compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.12'
    annotationProcessor group: 'org.projectlombok', name: 'lombok', version: '1.18.14'

    testImplementation('org.springframework.boot:spring-boot-starter-test')
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
test {
    useJUnitPlatform()
}
```  

#### GitConfigClientApplication

- 외부 저장소 설정(`Properties`) 내용을 클래스로 자동 매핑을 위해 `@EnableConfigurationProperties` 과 `@ConfigurationPropertiesScan`
- 애플리케이션 구동 후 적용된 설정 값 확인을 위해 `/static`, `/dynamic` 요청을 받을 수 있도록 컨트롤러를 추가한다. 

```java
@SpringBootApplication
@EnableConfigurationProperties
@ConfigurationPropertiesScan(value = "com.windowforsun.springcloudconfig.gitconfigclient")
@RestController
@RequiredArgsConstructor
public class GitConfigClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(GitConfigClientApplication.class, args);
    }

    private final StaticProperties staticProperties;
    private final DynamicProperties dynamicProperties;

    @GetMapping("/static")
    public Map<String, String> staticProperties() {
        return this.staticProperties.getConfig();
    }

    @GetMapping("/dynamic")
    public Map<String, String> dynamicProperties() {
        return this.dynamicProperties.getConfig();
    }
}
```  


#### StaticProperties

- `Spring Cloud Config` 에 의해 값이 설정 되지만 애플리케이션이 구동된 이후에는 갱신되지 않는 설정 값을 표현한 클리스이다. 
  - 이후 내용 부터 `visibility` 필드와 `group` 필드의 차이를 확인해둘 필요가 있다. 

```java
@Component
@Data
@RequiredArgsConstructor
public class StaticProperties {
    @Value("${visibility}")
    private String visibility;
    private final Group group;

    public Map<String, String> getConfig() {
        return Map.of(
                "visibility", this.visibility,
                "group.key1", this.group.getKey1(),
                "group.key2", this.group.getKey2()
        );
    }
}
```  

#### DynamicProperties

- `Spring Cloud Config` 에 의해 값이 설정되면서 애플리케이션 구동 이후에도 설정 값이 외부 저장소에 맞춰 변경되는 클래스이다. 
- 대부분 `StaticProperties` 와 비슷하고, `@RefreshScope` 어노테이션 유무에 동작에서 차이를 보인다. 

```java
@Component
@RefreshScope
@Data
@RequiredArgsConstructor
public class DynamicProperties {
    @Value("${visibility}")
    private String visibility;
    private final Group group;

    public Map<String, String> getConfig() {
        return Map.of(
                "visibility", this.visibility,
                "group.key1", this.group.getKey1(),
                "group.key2", this.group.getKey2()
        );
    }
}
```  

#### Group
- `StaticProperties`, `DynamicProperties` 에서 모두 사용하는 설정값 매핑 클래스이다. 
- 해당 클래스는 설정 파일에서 `group` 으로 시작하는 이름에 대해서 매핑 작업이 진행되는데, 
`StaticProperties` 와 `DynamicProperties` 에서 모두 사용 되고 있기 때문에 그 차이에 대해서 인지할 필요가 있다.


```java
@Data
@NoArgsConstructor
@ConfigurationProperties(prefix = "group")
public class Group {
    private String key1;
    private String key2;
}
```  

#### application.yaml

- 본 예제는 `Spring Boot 2.4` 이상인 환경에서 진행 되기 때문에 `Spring Cloud Config Server` 설정을 아래와 같이 수행한다.  

	```yaml
	spring:
	  config:
		import: "optional:configserver:http://spring-cloud-config-server"
	```  
 
- 참고로 `Spring Boot 2.4` 이전 버전에서는 아래와 같이 설정 한다.  

	```yaml
	spring:
	  cloud:
		config:
		  uri: http://spring-cloud-config-server
	```  

- `application.yaml`

```yaml
server:
  port: 8071

management:
  endpoints:
    web:
      exposure:
        include: refresh

```

#### 테스트
테스트에서는 앞서 미리 구성한 `Spring Cloud Config Server` 는 실제로도 구동중이고, 구동중인 상태라는 가정을 두고 진행한다. 
먼저 각 환경별 설정값이 잘 설정되는지 테스트 해보면 아래와 같다.  


- `application-lowercase-dev.yaml`

```yaml
spring:
  config:
    import: "optional:configserver:http://localhost:8070"
  cloud:
    config:
      name: lowercase
      profile: dev
```  

- `application-uppercase-real.yaml`

```yaml
spring:
  config:
    import: "optional:configserver:http://localhost:8070"
  cloud:
    config:
      name: uppercase
      profile: real
```  

```java
@ActiveProfiles("lowercase-dev")
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class GitConfigClientLowerCaseDevTest {
    @Autowired
    private StaticProperties staticProperties;
    @Autowired
    private DynamicProperties dynamicProperties;

    @Test
    public void not_update_staticProperties() {
        assertThat(staticProperties.getVisibility(), is("public"));
        assertThat(staticProperties.getGroup().getKey1(), is("dev1"));
        assertThat(staticProperties.getGroup().getKey2(), is("dev2"));
    }

    @Test
    public void not_update_dynamicProperties() {
        assertThat(dynamicProperties.getVisibility(), is("public"));
        assertThat(dynamicProperties.getGroup().getKey1(), is("dev1"));
        assertThat(dynamicProperties.getGroup().getKey2(), is("dev2"));
    }
}

@ActiveProfiles("uppercase-real")
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class GitConfigClientUpperCaseRealTest {
    @Autowired
    private StaticProperties staticProperties;
    @Autowired
    private DynamicProperties dynamicProperties;

    @Test
    public void not_update_staticProperties() {
        assertThat(staticProperties.getVisibility(), is("PUBLIC"));
        assertThat(staticProperties.getGroup().getKey1(), is("REAL1"));
        assertThat(staticProperties.getGroup().getKey2(), is("REAL2"));
    }

    @Test
    public void not_update_dynamicProperties() {
        assertThat(dynamicProperties.getVisibility(), is("PUBLIC"));
        assertThat(dynamicProperties.getGroup().getKey1(), is("REAL1"));
        assertThat(dynamicProperties.getGroup().getKey2(), is("REAL2"));
    }
}
```  

앞서 설명했지만 설정 값을 변경하고 싶다면 아래와 같이 수행해 준다. (`Spring Cloud Config Server` 는 구동 중인 상태라고 가정)  
1. 설정 저장소에 값을 변경한다.
1. 변경된 파일에 대해서 `Commit`, `Push` 를 수행한다. 
1. 직접 `Spring Cloud Config Client` 의 엔드포인트의 `/actuator/refresh` 호출

위 시나리오에 대해서 테스트를 수행하면 아래와 같다.

```java
@ActiveProfiles("lowercase-dev")
@AutoConfigureMetrics
@SpringBootTest(
        properties = {
          "spring.config.import=classpath:privateInfo.yaml"
        },
        webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT
)
@ExtendWith(SpringExtension.class)
public class GitConfigClientRefreshTest {
    @Autowired
    private StaticProperties staticProperties;
    @Autowired
    private DynamicProperties dynamicProperties;
    @Value("${git.username}")
    private String username;
    @Value("${git.password}")
    private String password;
    private String originContent;
    @LocalServerPort
    private int port;
    @Autowired
    private TestRestTemplate testRestTemplate;
    
    // 테스트에서 설정 업데이트 용으로 사용할 파일 이름
    private final String fileName = "/lowercase-dev.yaml";

    @BeforeAll
    public static void setAll() throws Exception {
        // 외부 설정 저장소 로컬에 클론
        // 외부 설정 저장소를 로컬에 클론하는 목적은 테스트 코드상에서 설정 값을 변경해서 반영하기 위함이다. 
        // 클론한 저장소가 애플리케이션 설정값으로 적용되지는 않는다. 
        Util.gitClone();
    }

    @AfterAll
    public static void clearAll() throws Exception {
        // 로컬에 클론한 저장소 삭제 
        Util.deleteRepo();
    }

    @BeforeEach
    public void setUp() throws Exception {
        // 설정 파일 원래 값 저장
        this.originContent = Util.readFile(this.fileName);
    }

    @AfterEach
    public void tearDown() throws Exception {
        // 설정 파일 원래 값으로 파일 다시 복구
        Util.writeFile(this.fileName, this.originContent);
        // 복구된 파일 내용으로 Commit, Push
        Util.gitAddCommitPush(this.username, this.password, "test rollback");
    }

    @Test
    public void update_staticProperties() throws Exception {
        // given
        assertThat(staticProperties.getVisibility(), is("public"));
        assertThat(staticProperties.getGroup().getKey1(), is("dev1"));
        assertThat(staticProperties.getGroup().getKey2(), is("dev2"));
        long timestamp = System.currentTimeMillis();
        String newProperties = "" +
                "visibility: public-" + timestamp + "\n" +
                "group:\n" +
                "  key1: dev1-" + timestamp + "\n" +
                "  key2: dev2-" + timestamp + "\n";
        // 설정 파일 내용 변경
        Util.writeFile(fileName, newProperties);
        // 변경된 파일 Commit, Push
        Util.gitAddCommitPush(this.username, this.password, "test change");

        // when
        // 현재 구동 중인 애플리케이션 설정 값 갱신을 위해 /actuator/refresh 호출
        this.testRestTemplate.postForObject(String.format("http://localhost:%d/actuator/refresh", this.port), null, Object.class);

        // then
        assertThat(staticProperties.getVisibility(), is("public"));
        assertThat(staticProperties.getGroup().getKey1(), is("dev1-" + timestamp));
        assertThat(staticProperties.getGroup().getKey2(), is("dev2-" + timestamp));
    }

    @Test
    public void update_dynamicProperties() throws Exception {
        // given
        assertThat(dynamicProperties.getVisibility(), is("public"));
        assertThat(dynamicProperties.getGroup().getKey1(), is("dev1"));
        assertThat(dynamicProperties.getGroup().getKey2(), is("dev2"));
        long timestamp = System.currentTimeMillis();
        String newProperties = "" +
                "visibility: public-" + timestamp + "\n" +
                "group:\n" +
                "  key1: dev1-" + timestamp + "\n" +
                "  key2: dev2-" + timestamp + "\n";
        // 설정 파일 내용 변경
        Util.writeFile(fileName, newProperties);
        // 변경된 파일 Commit, Push
        Util.gitAddCommitPush(this.username, this.password, "test change");

        // when
        // 현재 구동 중인 애플리케이션 설정 값 갱신을 위해 /actuator/refresh 호출
        this.testRestTemplate.postForObject(String.format("http://localhost:%d/actuator/refresh", this.port), null, Object.class);

        // then
        assertThat(dynamicProperties.getVisibility(), is("public-" + timestamp));
        assertThat(dynamicProperties.getGroup().getKey1(), is("dev1-" + timestamp));
        assertThat(dynamicProperties.getGroup().getKey2(), is("dev2-" + timestamp));
    }
}
```  

`DynamicProperties` 의 동작에 대해서 검증하는 검증문을 보면 정상적으로 설정 값이 갱신된 것을 확인 할 수 있다. 
하지만 `StaticProperties` 의 동작은 예상과는 벗어난 결과가 나온 것을 확인 할 수 있다. 
`visibility` 는 변경되지 않았지만, `group` 으로 시작하는 이름의 설정 값들이 모두 변경되 었기 때문이다. 
이는 `DynamicProperties` 에서도 `group` 설정 값을 사용 중이고, 
둘다 모두 동일한 `Group` 오브젝트를 사용하기 때문에 `DynamicProperties` 에서 `/actuator/refresh` 호출에 따라 
값을 갱신한 것이다. 
그리고 `StaticProperties` 는 `DynamicProperties` 가 갱신한 값을 사용하게 되기 때문에 발생한 현상이다.  

애플리케이션이 `Spring Cloud Config` 가 이미 적용된 상태에서 배포 중인 상황을 가정해보자. 
이때 `Spring Cloud Config Server` 가 어떤 이슈에 의해 다운됐다면 배포중인 애플리케이션 서버는 설정 값을 불러오지 못하기 때문에, 
에러가 발생하면서 정상적으로 구동되지 못할 것이다. 
이런 상황에서 `Failover` 처리로 가능한 방법이 로컬 설정 파일에 미리 기본 값 등으로 설정 값을 채워 두는 것이다.  

- `application-failover.yaml`

```yaml
spring:
  config:
    import: "optional:configserver:http://notexist"
  cloud:
    config:
      name: notexist
      profile: notexist


visibility: local
group:
    key1: local1
    key2: local2
```  

```java
@ActiveProfiles("failover")
@SpringBootTest
@ExtendWith(SpringExtension.class)
public class GitCloudConfigFailoverTest {
    @Autowired
    private StaticProperties staticProperties;
    @Autowired
    private DynamicProperties dynamicProperties;

    @Test
    public void failover_staticProperties() {
        assertThat(this.staticProperties.getVisibility(), is("local"));
        assertThat(this.staticProperties.getGroup().getKey1(), is("local1"));
        assertThat(this.staticProperties.getGroup().getKey2(), is("local2"));
    }

    @Test
    public void failover_dynamicProperties() {
        assertThat(this.dynamicProperties.getVisibility(), is("local"));
        assertThat(this.dynamicProperties.getGroup().getKey1(), is("local1"));
        assertThat(this.dynamicProperties.getGroup().getKey2(), is("local2"));
    }
}
```  

---
## Reference
[Spring Cloud Config](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_quick_start)  
