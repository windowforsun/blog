--- 
layout: single
classes: wide
title: "[Kubernetes 개념] 인그레스(Ingress) 무중단 배포"
header:
  overlay_image: /img/kubernetes-bg.jpg
excerpt: '쿠버네티스에서 무중단 배포와 인그레스를 사용한 무중단 배포에 대해 살펴보자'
author: "window_for_sun"
header-style: text
categories :
  - Kubernetes
tags:
  - Kubernetes
  - Concept
  - Ingress
  - Nonstop
  - HA
toc: true
use_math: true
---  

## 무중단 배포
쿠버네티스 클러스터에서 배포를 할때 상황을 살펴본다. 
클러스터에서 새로운 파드를 배포하고 교체 전의 상황은 아래와 같다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_ingress_nonstop_1_plant.png)

각 노드에는 이전 배포 버전인 `Pod(v1)` 과 새로운 배포 버전인 `Pod(v2)` 가 같이 있는 상황이면서, 
`Nginx` 는 `Pod(v1)` 에게 요청을 전달 하고 있다.  

`Pod(v2)` 의 구성과 실행이 완료되면 아래 그림과 같이 `Nginx` 에서 전달하는 요청은 `Pod(v2)` 가 받게 되고, 
이전 배포 버전인 `Pod(v1)` 은 삭제된다. 

![그림 1]({{site.baseurl}}/img/kubernetes/concept_ingress_nonstop_2_plant.png)

쿠버네티스 클러스터에서 위와 같이 자연스러운 배포를 수행하기 위해서는 아래와 같은 설정와 확인 사항이 필요하다. 

### maxSurge, maxUnavailable
각 노드의 파드 관리를 `RollingUpdate` 로 설정 했을 경우, `maxSurge`, `maxUnavailable` 필드의 설정이 필요하다.  

`maxSurge` 는 디플로이먼트를 사용해서 배포할 때 템플릿에 설정한 파드의 수보다 몇개의 파드를 추가로 생성 할 수 있는지에 대한 설정이다.  

그리고 `maxUnavailable` 은 디플로이먼트를 통해 업데이트를 수행할 때 최대 몇개의 파드가 비활성화 상태가 될 수 있는지 설정하는 필드이다.  

서비스 중 트래픽 유실 없이 배포를 수행하기 위해서는 현재 구성 환경에 알맞게 두 필드의 값을 설정해 줘야 한다.  


### readinessProbe
[파드]({{site.baseurl}}{% link _posts/kubernetes/2020-06-29-kubernetes-concept-pod.md %})
에서 언급한 것과 같이, 현재 파드의 상태 파악을 위해서는 `livenessProbe` 와 `readinessProbe` 두 가지 프로브 설정이 필요하다. 
`livenessProbe` 는 파드의 컨테이너가 정상 실행 여부를 파악해서 `kubelet` 에서 다시 재시작 설정에 맞게 관리하도록 한다.  

무중단 배포에서는 `readinessProbe` 을 주의 깊게 설정해야 한다. 
`readinessProbe` 는 파드의 컨테이너가 서비스 요청을 처리할 준비가 돼있는지 파악할 수 있는 설정이다. 
`readinessPribe` 가 성공해야 파드와 연결된 서비스에서 트래픽을 전달하게 된다.  

특정 애플리케이션의 컨테이너의 경우 컨테이너의 실행과는 별개로 실제 서비스를 수행 하기까지 걸리는 초기화 과정이 소요된다. 
이런 상황에서 바로 파드쪽으로 트래픽이 전달되면 서비스를 정상적으로 처리할 수 없기 때문에, 
`readinessProbe` 설정을 통해 이를 극복해야 한다.  

만약 `readinessProbe` 의 설정이 어려운 경우 `.spec.minReadySeconds` 필드를 사용해서 파드가 준비 상태가 될 때까지 대기 시간을 설정 할 수 있다. 
대기시간 전까지는 해당 파드로 트래픽이 전달되지 않는다. 
만약 `readinessProbe` 와 `.spec.minReadySeconds` 필드가 모두 설정된 경우라면,
`readinessProbe` 가 성공하게 되면 `.spec.minReadySeconds` 의 대기 시간은 무시된다. 

### Graceful 종료
각 노드에서 컨테이너를 관리하는 `kubelet` 은 배포 과정에서 새로운 파드를 생성하고 이전 파드를 종료 할때, 
이전 파드에게 먼저 `SIGTERM` 신호를 보낸다. 
트래픽 유실 없는 무중단 배포를 위해서는 컨테이너에서는 `SIGTERM` 신호를 받았을 때 현재 처리중인 요청 까지만 수행하고, 
다음 요청은 받지 않는 `Graceful` 한 종료 처리가 필요하다.  

만약 아래 그림과 같이 `Nginx` 에서 아직 `Pod(v1)` 쪽으로  트래픽을 전송하고 해당 파드가 요청을 처리 중에 `SIGTERM` 을 받아 종료가 된다면, 
해당 요청에서는 에러가 발생하게 된다. 
또한 `kubelet` 에서는 `SIGTERM` 을 전송 한 후, 
`.terminationGracePeriodSeconds` 필드에 설정 가능한 기본 대기 시간(30초)이 지나면 `SIGKILL` 신호를 보내 강제 종료 시킨다.  

![그림 1]({{site.baseurl}}/img/kubernetes/concept_ingress_nonstop_3_plant.png)

특정 경우 컨테이너에 `Graceful` 한 종료 처리를 설정하지 못할 수도 있다.
이런 상황에서는 파드 생명 주기의 훅을 설정해 사용할 수 있다. 
흑은 파드 실행 후 실행되는 `Poststart hook` 과 종료 직전에 실행되는 `Prestop hook` 이 있다. 
여기서 `Prestop hook` 은 `SIGTERM` 신호를 보내기 전에 실행되기 때문에, 컨테이너 설정과는 별개로 `Graceful` 효과를 낼 수 있다. 
`Prestop hook` 이 완료 되기 전까지는 컨테이너에 `SIGTERM` 신호를 보내지 않기 때문에, 파드에서 이를 조정 할 수 있다.  

앞서 설명한 `.terminationGracePeriodSeconds` 시간이 지나면 `SIGKILL` 을 통해 강제 종료를 시킨다는 점에 유의 해야 한다.


## Ingress 무중단 배포
`Spring` 기반으로 테스트용 애플리케이션을 구현하고 무중단 배포 테스트를 수행한다.  

### 테스트 애플리케이션
- 프로젝트 구조

    ```
    .
    ├── HELP.md
    ├── build.gradle
    ├── gradlew
    ├── gradlew.bat
    ├── settings.gradle
    └── src
        ├── main
        │   ├── java
        │   │   └── com
        │   │       └── windowforsun
        │   │           └── nonstop
        │   │               ├── AppConfig.java
        │   │               ├── CountFilter.java
        │   │               ├── NonstopApplication.java
        │   │               └── TestController.java
        │   └── resources
        │       ├── application.properties
        │       ├── static
        │       └── templates
        └── test
            └── java
                └── com
                    └── windowforsun
                        └── nonstop
                            └── NonstopApplicationTests.java
    
    ```  

- `build.gradle`

    ```groovy
    import java.time.Instant
    
    plugins {
        id 'org.springframework.boot' version '2.3.1.RELEASE'
        id 'io.spring.dependency-management' version '1.0.9.RELEASE'
        id 'java'
        id 'com.google.cloud.tools.jib' version '1.6.0'
    }
    
    group = 'com.windowforsun'
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '1.8'
    
    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }
    
    repositories {
        mavenCentral()
    }
    
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-web'
        compileOnly 'org.projectlombok:lombok'
        developmentOnly 'org.springframework.boot:spring-boot-devtools'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
            exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
    }
    
    test {
        useJUnitPlatform()
    }
    
    
    jib {
        version = 'v1'
        from {
            image = "openjdk:8-jre-alpine"
        }
        to {
            image = "windowforsun/nonstop-spring"
            tags = [version.toString()]
            auth {
                username = "<Docker Hub username>"
                password = "<Docker Hub user passwd>"
            }
        }
        container {
            mainClass = "com.windowforsun.nonstop.NonstopApplication"
            ports = ["8080"]
            creationTime = Instant.now()
            environment = [
                'version' : version.toString()
            ]
        }
    }
    ```  
  
    - 빌드 도구로는 `Gradle` 을 사용한다. 
    - 애플리케이션을 `Docker` 이미지로 빌드하기 위해 `jib` 플러그인을 사용한다. 
    - 테스트를 위해 환경변수 `version` 의 값과 태그의 값은 동일하게 한다. 

- `NonstopApplication.java`

    ```java
    @SpringBootApplication
    public class NonstopApplication {
        public static AtomicInteger CONNECTION = new AtomicInteger(0);
    
        // print current Connection
        private final static Thread MONITORING_THREAD = new Thread(() -> {
            while(true) {
                try {
                    System.out.println("connection : " + CONNECTION.get());
                    Thread.sleep(1000);
                } catch(Exception e) {
                    e.printStackTrace();
                }
            }
        });
    
    
        public static void main(String[] args) {
            Signal sigInt = new Signal("INT");
            Signal sigTerm = new Signal("TERM");
    
            // Ignore SIGINT
            Signal.handle(sigTerm, Signal -> {
                System.out.println("signal term : " + sigTerm.getNumber());
            });
    
            // Ignore SIGTERM
            Signal.handle(sigInt, Signal -> {
                System.out.println("signal int : " + sigInt.getNumber());
            });
    
            SpringApplication.run(NonstopApplication.class, args);
            MONITORING_THREAD.start();
        }
    
    }
    ```  
  
    - 테스트 애플리케이션의 `Main` 클래스로 1초마다 현재 커넥션 수를 출력하는 모니터링 쓰레드에 대한 부분이 있다. 
    - 배포 과정에서 컨테이너로 보내지는 `SIGINT` 와 `SIGTERM` 을 처리하는 부분은 출력 처리만 수행한다. 
    
- `TestController.java`

    ```java
    @RestController
    public class TestController {
    
        @GetMapping("/sleep/{sleepSecond}")
        public ResponseEntity<String> sleep(@PathVariable int sleepSecond) throws Exception {
            String version = System.getenv("version");
            int statusCode;
    
            // Sleep
            Thread.sleep(sleepSecond * 1000);
    
            if(version.equals("v1")) {
                statusCode = 201;
            } else {
                statusCode = 202;
            }
    
            return ResponseEntity.status(statusCode).body(version);
        }
    
        @GetMapping("/liveness")
        public ResponseEntity<Void> liveness() {
            return ResponseEntity.ok().build();
        }
    
        @GetMapping("/readiness")
        public ResponseEntity<Void> readiness() {
            return ResponseEntity.ok().build();
        }
    
        @GetMapping("/prestop")
        public ResponseEntity<Void> prestop() {
            return ResponseEntity.ok().build();
        }
    }
    ```  
  
    - 테스트 애플리케이션에서 요청을 받아 경로에 따른 처리를 수행한다. 
    - `/sleep/<초>` 경로의 요청은 `<초>` 시간만큼 `sleep` 을 수행하고, 
    `v1` 버전일 경우 상태코드를 `201`로 설정하고 `v2` 버전일 경우 상태코드를 `202` 로 설정해서 응답한다. 
    - `/liveness`, `/readiness` 파드 상태를 확인하는 용도로 사용되는 `Probe` 처리를 위해 `200` 상태코드를 응답한다. 
    그리고 `/prestop` 은 컨테이너를 중지시키기 전에 호출 되는 `Hook` 을 처리한다. 
    
- `CounterFilter.java`

    ```java
    public class CountFilter implements Filter {
        @Override
        public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
            HttpServletRequest httpServletRequest = (HttpServletRequest)request;
    
            // Connection increment
            NonstopApplication.CONNECTION.incrementAndGet();
            // Request info
            System.out.println("path : " + httpServletRequest.getRequestURI());
            // Process request
            chain.doFilter(request, response);
            // Connection decrement
            NonstopApplication.CONNECTION.decrementAndGet();
        }
    
        @Override
        public void init(FilterConfig filterConfig) throws ServletException {
    
        }
    
        @Override
        public void destroy() {
        }
    }
    ```  
  
    - `/sleep` 경로 요청의 커넥션을 카운트하는 `Filter` 이다. 
    - 요청이 들어오면 커넥션을 증가시키고, 응답 전에 커넥션을 감소 시킨다. 
    
- `AppConfig.java`

    ```java
    @Configuration
    public class AppConfig implements WebMvcConfigurer {
        @Bean
        public FilterRegistrationBean<CountFilter> countFilterBean() {
            FilterRegistrationBean<CountFilter> bean = new FilterRegistrationBean<>(new CountFilter());
            bean.addUrlPatterns("/*");
    
            return bean;
        }
    }
    ```  
  
    - 애플리케이션에서 `Filter` 관련 설정을 수행 한다. 
    
구성한 테스트 프로젝트의 `Docker` 이미지는 `jib` 플러그인을 사용해서 아래와 같이 빌드해서 생성할 수 있다. 

```bash
$ ./gradlew jib

.. 출력 ..
```  

위 명령어를 사용하면 프로젝트를 이미지로 빌드하고, 설정된 `Docker Hub` 계정으로 `Push` 한다.  

명령을 사용해서 환경변수 `version` 값이 `v1` 인 `v1` 태그, `version` 값이 `v2` 인 `v2` 태그로 총 2개 빌드 버전을 준비한다.  

```bash
$ docker image ls | grep nonstop-spring
windowforsun/nonstop-spring                                      v1                                               0818ead61150        3 hours ago         102MB
windowforsun/nonstop-spring                                      v2                                               1ebeed7b0b9e        3 hours ago         102MB
```  

### 테스트 툴
정상적으로 무중단 배포가 수행되는지 테스트하기 위해 [vegeta](https://github.com/tsenart/vegeta) 
라는 툴을 사용한다. 
호스트에 직접 설치해서 사용도 할 수 있지만, `Docker` [이미지](https://hub.docker.com/r/peterevans/vegeta/)
를 사용해서 테스트를 진행한다. 


### 쿠버네티스 구성
쿠버네티스 클러스터에서 무중단 배포를 위해 디플로이먼트와 인그레스를 구성한다. 
테스트를 위한 디플로이먼트 템플릿은 아래와 같다. 

```yaml
# deployment-nonstop.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nonstop-web
spec:
  selector:
    matchLabels:
      run: nonstop-web
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:
    metadata:
      labels:
        run: nonstop-web
    spec:
      containers:
        - image: windowforsun/nonstop-spring:v1
          imagePullPolicy: Always
          name: nonstop-web
          ports:
            - containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /liveness
              port: 8080
          readinessProbe:
            httpGet:
              path: /readiness
              port: 8080
          lifecycle:
            preStop:
              httpGet:
                path: /prestop
                port: 8080
      terminationGracePeriodSeconds: 30
```  

- `.spec.strategy.rollingUpdate` :  디플로이먼트의 파드를 롤링 업데이트와 관련된 설정 필드이다. 
하위 `maxSurge` 필드 값은 `25%` 로 설정된 기본 파드 개수에서 `25%` 까지 추가할 수 있다. 
하위 `maxUnavailable` 필드 값도 `25%` 로 업데이터를 수행하며 설정된 기본 파드개수에서 `25%` 만큼이 이용불가 상태가 된다. 
- `.spec.template.spec.containers[].image` : 컨테이너에서 사용하는 이미지를 설정하는 필드로, 
앞서 빌드하고 `Docker Hub` 에 올라간 테스트 애플리케이션 이미지를 설정한다. 
- `.spec.template.spec.containers[].livenessProbe` : 파드의 활성 상태를 체크하는 필드이다. 
`HTTP GET` 메소드를 사용하고, `/liveness` 경로와 `8080` 포트로 요청을 보내 컨테이너 상태를 확인한다. 
- `.spec.template.spec.containers[].readinessProbe` : 파드의 요청 처리 가능 상태를 체크하는 필드이다. 
`HTTP GET` 메소드를 사용하고, `/readiness` 경로와 `8080` 포트로 요청을 보내 켄테이너 상태를 확인 한다.  

인그레스 템플릿은 아래와 같다. 

```yaml
# ingress-nonstop.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nonstop-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: www.ingress-test.com
      http:
        paths:
          - backend:
              serviceName: nonstop-web
              servicePort: 8080
            path: /
```  

- `spec.rules[].host` : 인그레스에서 사용하는 호스트이름으로 `www.ingress-test.com` 을 사용 한다. 

`/etc/hosts` 파일에 인그레스에서 사용할 호스트 이름을 추가해 준다. 

```bash
$ vi /etc/hosts

127.0.0.1   www.ingress-test.com
```  

먼저 디플로이먼트 템플릿을 적용하고, 서비스를 사용해서 외부로 노출 시킨다. 

```bash
$ kubectl apply -f deployment-nonstop.yaml
deployment.apps/nonstop-web created
$ kubectl expose deployment nonstop-web
service/nonstop-web exposed
$ kubectl get deploy,svc
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nonstop-web   1/1     1            1           56s

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/nonstop-web   ClusterIP   10.107.250.254   <none>        8080/TCP   15s
```  

인그레스 템플릿도 클러스터에 적용해 준다. 

```bash
$ kubectl apply -f ingress-nonstop.yaml
ingress.extensions/nonstop-ingress created
$ kubectl get deploy,svc,ingress
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nonstop-web   1/1     1            1           2m4s

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/nonstop-web   ClusterIP   10.107.250.254   <none>        8080/TCP   83s

NAME                                 HOSTS                           ADDRESS       PORTS   AGE
ingress.extensions/nonstop-ingress   www.ingress-test.com            10.96.77.73   80      16s
```  

[ingress-nginx]({{site.baseurl}}{% link _posts/kubernetes/2020-07-16-kubernetes-concept-ingress.md %})
에서 적용한 컨트롤러의 포트를 다시 한번 확인해본다. 

```bash
$ kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.96.77.73   <none>        80:32339/TCP,443:31379/TCP   3d22h
```  

앞서 언급한 것과 같이, 
배포 테스트는 `vegeta` 툴을 사용하는데 테스트 애플리케이션에 사용하는 간단한 예시는 아래와 같다. 

```bash
$ docker run \
> --rm \
> --network=host \
> --add-host www.ingress-test.com:127.0.0.1 \
> -i \
> peterevans/vegeta \
> sh -c \
> "echo 'GET http://www.ingress-test.com:32339/sleep/3' | vegeta attack -rate=2 -keepalive=false -duration=10s | vegeta report"
Requests      [total, rate, throughput]         20, 2.11, 1.60
Duration      [total, attack, wait]             12.511s, 9.5s, 3.011s
Latencies     [min, mean, 50, 90, 95, 99, max]  3.006s, 3.01s, 3.008s, 3.015s, 3.027s, 3.035s, 3.035s
Bytes In      [total, mean]                     40, 2.00
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      201:20
```  

예시 명령은 `-rate=2` 를 통해 매초당 2개의 요청을 보내고, `-duration=10s` 으로 동작을 10초 동안 반복한다. 
그리고 `vegeta report` 로 요청에 대한 결과를 출력한다. 
`vegeta report` 명령으로 출력되는 `Success` 필드를 보면 모든 요청이 성공한 것과 모든 요청의 응답 상태코드가 `201`(`v1`) 인것을 확인 할 수 있다. 

배포 테스트의 과정은 아래와 같다.
- `vegeta` 명령을 통해 일정 시간 요청을 지속적으로 보낸다. 
- 특정 시점에 `kubectl set image` 명령으로 디플로이먼트 이미지를 업데이트 한다. 
- `vegeta report` 에 출력되는 결과로 모든 요청이 성공했는지와 응답 상태코드를 확인한다. 

테스트를 위해 2개의 쉘을 준비한다. 
하나는 `vegeta` 명령을 수행하는 쉘이고, 하나는 요청 중 디플로이먼트의 이미지를 업데이트하는 명령을 수행하는 쉘이다.  

첫 번째 쉘에서 `/sleep/5` 경로와 `-rate=2`, `-duration=60s` 아래와 같은 명령으로 테스트를 시작한다.  

```bash
$ docker run \
--rm \
--network=host \
--add-host www.ingress-test.com:127.0.0.1 \
-i \
peterevans/vegeta \
sh -c \
"echo 'GET http://www.ingress-test.com:32339/sleep/5' | vegeta attack -rate=2 -keepalive=false -duration=60s | vegeta report"
```  

두 번째 쉘에서 바로 `kubectl set image` 명령으로 이미지를 `v2` 로 변경한다. 

```bash
$ kubectl set image deployment/nonstop-web nonstop-web=windowforsun/nonstop-spring:v2
deployment.apps/nonstop-web image updated
```  

시간이 지나 첫 번째 쉘에서 테스트 결과를 확인하면 아래와 같다. 

```bash
$ docker run \
--rm \
--network=host \
--add-host www.ingress-test.com:127.0.0.1 \
-i \
peterevans/vegeta \
sh -c \
"echo 'GET http://www.ingress-test.com:32339/sleep/5' | vegeta attack -rate=2 -keepalive=false -duration=60s | vegeta report"
Requests      [total, rate, throughput]         120, 2.02, 1.86
Duration      [total, attack, wait]             1m5s, 59.5s, 5.006s
Latencies     [min, mean, 50, 90, 95, 99, max]  5.005s, 5.008s, 5.007s, 5.013s, 5.018s, 5.023s, 5.024s
Bytes In      [total, mean]                     240, 2.00
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      201:36  202:84
```  

`Success` 필드에서 모든 요청이 정상적으로 수행된 것을 확인 할 수 있다. 
그리고 `Status Codes` 필드를 보면 `v1` 버전에 해당하는 `201` 상태 코드가 36개 요청, 
`v2` 버전에 해당하는 `202` 상태 코드가 84 개 인것도 확인 가능하다. 
중간에 이미지 변경으로 파드가 재시작 되었지만, 
유실되는 요청없이 모든 요청이 정상 처리 된 것으로 판단할 수 있다. 

---
## Reference
