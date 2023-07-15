--- 
layout: single
classes: wide
title: "[Spring 실습] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Cloud Data Flow
    - SCDF
toc: true
use_math: true
---  

## Spring Cloud DataFlow Shell
[Spring Cloud DataFlow Shell](https://docs.spring.io/spring-cloud-dataflow/docs/2.10.3/reference/htmlsingle/#shell)
을 사용하면 `UI` 대시보드 대신 `Spring Cloud DataFlow` 를 조작할 수 있다. 
대시보드와 동일한 모든 기능을 수행할 수 있는 멍령어를 제공한다.  

[Shell 설치](https://dataflow.spring.io/docs/installation/local/manual/#shell) 
는 아래와 같다.  

```bash
$ wget -O spring-cloud-dataflow-shell-2.10.2.jar https://repo.maven.apache.org/maven2/org/springframework/cloud/spring-cloud-dataflow-shell/2.10.2/spring-cloud-dataflow-shell-2.10.2.jar
$ java -jar spring-cloud-dataflow-shell-2.10.2.jar
```  

### SCDF 연결
`Shell` 과 `SCDF` 를 연결하는 방법은 2가지 방법이 있다.  

- 실행시 프로퍼티를 사용해서 연결

```bash
$ java -jar spring-cloud-dataflow-shell-2.10.2.jar --dataflow.uri=http://{SCDF_IP}:{SCDF_PORT}
dataflow:>
```  

- 실해 후 명령을 통해 연결

```bash
$ java -jar spring-cloud-dataflow-shell-2.10.2.jar
server-unknown:>dataflow config server http://{SCDF_IP}:{SCDF_PORT}
```  

### 명령어 살펴보기 
`Shell` 에서 사용할 수 있는 쉘에서 `help` 명령을 통해 확인 할 수 있다.  

```bash
dataflow:>help
AVAILABLE COMMANDS

App Registry Commands
       app default: Change the default application version
       app info: Get information about an application
       app unregister: Unregister an application
       app import: Register all applications listed in a properties file
       app all unregister: Unregister all applications
       app register: Register a new application
       app list: List all registered applications

Built-In Commands
       help: Display help about available commands
       stacktrace: Display the full stacktrace of the last error.
       clear: Clear the shell screen.
       quit, exit: Exit the shell.
       history: Display or save the history of previously run commands
       version: Show version info
       script: Read and execute commands from a file.

Config Commands
       dataflow config info: Show the Dataflow server being used
       dataflow config server: Configure the Spring Cloud Data Flow REST server to use

Http Commands
       http post: POST data to http endpoint
       http get: Make GET request to http endpoint

Job Commands
       job execution step display: Display the details of a specific step execution
       job execution step list: List step executions filtered by jobExecutionId
       job instance display: Display the job executions for a specific job instance.
       job execution restart: Restart a failed job by jobExecutionId
       job execution list: List created job executions filtered by jobName
       job execution display: Display the details of a specific job execution
       job execution step progress: Display the details of a specific step progress

Runtime Commands
       runtime actuator post: Invoke actuator POST endpoint on app instance
       runtime apps: List runtime apps
       runtime actuator get: Invoke actuator GET endpoint on app instance

Stream Commands
       stream scale app instances: Scale app instances in a stream
       stream platform-list: List Skipper platforms
       stream all undeploy: Un-deploy all previously deployed stream
       stream history: Get history for the stream deployed using Skipper
       stream update: Update a previously created stream using Skipper
       stream info: Show information about a specific stream
       stream manifest: Get manifest for the stream deployed using Skipper
       stream deploy: Deploy a previously created stream using Skipper
       stream all destroy: Destroy all existing streams
       stream validate: Verify that apps contained in the stream are valid.
       stream list: List created streams
       stream undeploy: Un-deploy a previously deployed stream
       stream rollback: Rollback a stream using Skipper
       stream create: Create a new stream definition
       stream destroy: Destroy an existing stream

Task Commands
       task validate: Validate apps contained in task definitions
       task destroy: Destroy an existing task
       task platform-list: List platform accounts for tasks
       task create: Create a new task definition
       task execution log: Retrieve task execution log
       task execution list: List created task executions filtered by taskName
       task all destroy: Destroy all existing tasks
       task execution cleanup: Clean up any platform specific resources linked to a task execution
       task execution status: Display the details of a specific task execution
       task execution stop: Stop executing tasks
       task list: List created tasks
       task execution current: Display count of currently executin tasks and related information
       task launch: Launch a previously created task

Task Scheduler Commands
       task schedule destroy: Delete task schedule
       task schedule create: Create new task schedule
       task schedule list: List task schedules by task definition name
```  

애플리케이션 추가/삭제 부터 시작해서 스트림 정의, 배포, 롤백, 업데이트 등 대부분 동작이 가능하다. 
아래에서 몇가지 명령에 대해서만 예제로 알아본다. 

### SCDF 정보 (dataflow config info)
`dataflow config info` 를 사용하면 현재 `Shell` 과 연결된 `SCDF` 의 상세정보를 확인 할 수 있다.  

```bash
dataflow:>dataflow config info
╔═════════════════╤══════════════════════════════════════════════════════════════════════════════╗
║     Target      │           http://{SCDF_IP}:{SCDF_PORT}                                       ║
╟─────────────────┼──────────────────────────────────────────────────────────────────────────────╢
║     Result      │Successfully targeted http://{SCDF_IP}:{SCDF_PORT}                            ║
╟─────────────────┼──────────────────────────────────────────────────────────────────────────────╢
║                 │        Analytics Enabled: true                                               ║
║                 │Monitoring Dashboard Type: NONE                                               ║
║    Features     │        Schedules Enabled: true                                               ║
║                 │          Streams Enabled: true                                               ║
║                 │            Tasks Enabled: true                                               ║
╟─────────────────┼──────────────────────────────────────────────────────────────────────────────╢
║                 │spring-cloud-dataflow-server: 2.9.1                                           ║
║    Versions     │ Spring Cloud Data Flow Core: 2.9.1                                           ║
║                 │    Spring Cloud Dataflow UI: 3.2.1                                           ║
║                 │Spring Cloud Data Flow Shell: 2.9.1                                           ║
╟─────────────────┼──────────────────────────────────────────────────────────────────────────────╢
║    Security     │         Authenticated: false                                                 ║
║                 │Authentication Enabled: false                                                 ║
╟─────────────────┼──────────────────────────────────────────────────────────────────────────────╢
║                 │Deployer Implementation Version: 2.8.1                                        ║
║                 │                  Deployer Name: Spring Cloud Skipper Server                  ║
║                 │           Deployer Spi Version: 2.8.1                                        ║
║                 │                   Java Version: 11.0.13                                      ║
║Skipper Deployer │           Platform Api Version:                                              ║
║                 │        Platform Client Version:                                              ║
║                 │          Platform Host Version:                                              ║
║                 │                  Platform Type: Skipper Managed                              ║
║                 │            Spring Boot Version: 2.5.5                                        ║
║                 │                 Spring Version: 5.3.10                                       ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║Platform Specific│default: kubernetes                                                           ║
╟─────────────────┼──────────────────────────────────────────────────────────────────────────────╢
║                 │Deployer Implementation Version: 2.7.1                                        ║
║                 │                  Deployer Name: KubernetesTaskLauncher                       ║
║                 │           Deployer Spi Version: 2.7.1                                        ║
║                 │                   Java Version: 11.0.13                                      ║
║  Task Launcher  │           Platform Api Version: v1                                           ║
║                 │        Platform Client Version: unknown                                      ║
║                 │          Platform Host Version: unknown                                      ║
║                 │                  Platform Type: Kubernetes                                   ║
║                 │            Spring Boot Version: 2.5.5                                        ║
║                 │                 Spring Version: 5.3.10                                       ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║Platform Specific│ namespace: default                                                          ║
║                 │master-url: https://172.23.0.1:443/                                           ║
╚═════════════════╧══════════════════════════════════════════════════════════════════════════════╝
```  

### 애플리케이션 추가 (app register)
`app register --type {application_type} --name {application_name} --uri {application_uri}` 
를 사용하면 새로운 애플리케이션을 추가할 수 있다. 
테스트를 위해 `Processor` 애플리케이션 중 [uppercase-transformer-kafka](https://hub.docker.com/r/springcloudstream/uppercase-transformer-kafka)
라는 애플리케이션을 추가한다.  

```bash
dataflow:>app register --type processor --name uppercase-transformer-kafka --uri docker:springcloudstream/uppercase-transformer-kafka
Successfully registered application 'processor:uppercase-transformer-kafka'
```  

### 애플리케이션 리스트 (app list)
`app list` 명령은 현재 `SCDF` 에 추가된 애플리케이션의 전체 목록을 확인 할 수 있다.  

```bash
dataflow:>app list
╔═══╤═══════════════╤════════════════════════════╤══════════════════╤════╗
║app│    source     │         processor          │       sink       │task║
╠═══╪═══════════════╪════════════════════════════╪══════════════════╪════╣
║   │jms            │transform                   │mqtt              │    ║
║   │jdbc           │bridge                      │rabbit            │    ║
║   │twitter-search │object-detection            │rsocket           │    ║
║   │mqtt           │aggregator                  │websocket         │    ║
║   │time           │script                      │zeromq            │    ║
║   │mail           │header-enricher             │geode             │    ║
║   │load-generator │filter                      │jdbc              │    ║
║   │file           │groovy                      │s3                │    ║
║   │twitter-stream │twitter-trend               │sftp              │    ║
║   │mongodb        │image-recognition           │file              │    ║
║   │http           │http-request                │wavefront         │    ║
║   │sftp           │splitter                    │elasticsearch     │    ║
║   │syslog         │semantic-segmentation       │analytics         │    ║
║   │twitter-message│cdc-pre-processor           │mongodb           │    ║
║   │websocket      │uppercase-transformer-kafka │log               │    ║
║   │rabbit         │                            │pgcopy            │    ║
║   │s3             │                            │twitter-message   │    ║
║   │ftp            │                            │router            │    ║
║   │zeromq         │                            │twitter-update    │    ║
║   │tcp            │                            │tcp               │    ║
║   │geode          │                            │cassandra         │    ║
║   │cdc-debezium   │                            │throughput        │    ║
║   │debezium-source│                            │redis             │    ║
║   │               │                            │ftp               │    ║
╚═══╧═══════════════╧════════════════════════════╧══════════════════╧════╝
```  

`Processor` 리스트를 보면 방금 추가한 `uppercase-transformer-kafka` 를 확인 할 수 있다.  

### 애플리케이션 정보 (app info)
`app info --type {application_type} --name {application_name}` 명령은 
해당 애플리케이션의 버전, 프로퍼티 등 상세 정보를 확인 할 수 있다.  

```bash
dataflow:>app info --type processor --name transform
Information about processor application 'transform':
Version: '3.2.1':
Default application version: 'true':
Resource URI: docker:springcloudstream/transform-processor-kafka:3.2.1
╔══════════════════════════════╤══════════════════════════════╤══════════════════════════════╤══════════════════════════════╗
║         Option Name          │         Description          │           Default            │             Type             ║
╠══════════════════════════════╪══════════════════════════════╪══════════════════════════════╪══════════════════════════════╣
║spel.function.expression      │A SpEL expression to apply.   │<none>                        │java.lang.String              ║
╚══════════════════════════════╧══════════════════════════════╧══════════════════════════════╧══════════════════════════════╝
```  

### 애플리케이션 삭제 (app unregister)
`app unregister --type {application_type} --name {application_name}` 
명령을 통해 등록한 애플리케이션을 삭제 할 수 있다.  

```bash
dataflow:>app unregister --type processor --name uppercase-transformer-kafka
Successfully unregistered application 'uppercase-transformer-kafka' with type 'processor'.
```  

### 스트림 생성 (stream create)
`stream create --definition {stream_definition} --name {stream_name}` 
명령을 사용하면 정의와 이름에 해당하는 새로운 스트림 생성이 가능하다. 

테스트로 생성해볼 스트림은
[Transformer, Router, Tap]({{site.baseurl}}{% link _posts/spring/2023-07-03-spring-practice-spring-scdf-spring-stream-application-transform-router.md %})
와 동일한 스트림을 생성한다. 

```bash
dataflow:>stream create --definition "time | transform | router" --name shell-time-transform-router
Created new stream 'shell-time-transform-router'
dataflow:>stream create --definition ":test-router-even > log" --name shell-test-router-even-log
Created new stream 'shell-test-router-even-log'
dataflow:>stream create --definition ":test-router-odd > log" --name shell-test-router-odd-log
Created new stream 'shell-test-router-odd-log'
```  

### 스트림 리스트 조회 (stream list)
`stream list` 명령으로 현재 `SCDF` 에 생성된 모든 스트림 이름, 설명, 상태를 조회 할 수 있다.  

```bash
dataflow:>stream list
╔═════════════════════════════════════╤═══════════╤═════════════════════════════════════════════════════════════════════════════╤══════════════════════════════════════════════════════════════════════╗
║             Stream Name             │Description│                              Stream Definition                              │                                Status                                ║
╠═════════════════════════════════════╪═══════════╪═════════════════════════════════════════════════════════════════════════════╪══════════════════════════════════════════════════════════════════════╣
║shell-test-router-even-log           │           │:test-router-even > log                                                      │The app or group is known to the system, but is not currently deployed║
║shell-test-router-odd-log            │           │:test-router-odd > log                                                       │The app or group is known to the system, but is not currently deployed║
║shell-time-transform-router          │           │time | transform | router                                                    │The app or group is known to the system, but is not currently deployed║
╚═════════════════════════════════════╧═══════════╧═════════════════════════════════════════════════════════════════════════════╧══════════════════════════════════════════════════════════════════════╝
```  

### 스트림 배포 (stream deploy)
`stream deploy --name {stream_name} --properties {stream_properties}` 
명령으로 배포할 스트림과 배포할 때 애플리케이션에서 사용할 프로퍼티를 넣어주면 해당 내용으로 배포가 가능하다.  

배포할 스트림별 프로퍼티 내용은 아래와 같다. 

```properties
# shell-test-router-even-log 
deployer.*.cpu=100m
deployer.*.kubernetes.limits.cpu=2000m
deployer.*.memory=500Mi
deployer.*.kubernetes.limits.memory=1000Mi
app.time.spring.integration.poller.fixed-rate=1000
app.time.date-format=yyyy-MM-dd HH:mm:ss
app.transform.spel.function.expression=payload.substring(payload.length() - 1)
app.router.expression=new Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'

# shell-test-router-odd-log  
deployer.*.cpu=100m
deployer.*.kubernetes.limits.cpu=2000m
deployer.*.kubernetes.limits.memory=2000Mi
deployer.*.memory=1000Mi
app.log.name=even-log
spring.cloud.dataflow.skipper.platformName=default

# shell-time-transform-router
deployer.*.cpu=100m
deployer.*.kubernetes.limits.cpu=2000m
deployer.*.kubernetes.limits.memory=2000Mi
deployer.*.memory=1000Mi
app.log.name=odd-log
spring.cloud.dataflow.skipper.platformName=default

```  

> Shell 을 통해 배포할떄 프로퍼티 중 `deployer.*` 관련은 가장 앞에 두는게 좋다. 그렇지 않으면 파싱에러가 발생한 경험이 있다. 

```bash
dataflow:>stream deploy --name shell-time-transform-router --properties "deployer.*.cpu=100m,\
deployer.*.kubernetes.limits.cpu=2000m,\
deployer.*.memory=500Mi,\
deployer.*.kubernetes.limits.memory=1000Mi,\
app.time.spring.integration.poller.fixed-rate=1000,\
app.time.date-format=yyyy-MM-dd HH:mm:ss,\
app.transform.spel.function.expression=payload.substring(payload.length() - 1),\
app.router.expression=new Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'"
Deployment request has been sent for stream 'shell-time-transform-router'

ataflow:>stream deploy --name shell-test-router-even-log --properties "app.log.name=even-log,\
deployer.*.cpu=100m,\
deployer.*.kubernetes.limits.cpu=2000m,\
deployer.*.kubernetes.limits.memory=2000Mi,\
deployer.*.memory=1000Mi,\
spring.cloud.dataflow.skipper.platformName=default"
Deployment request has been sent for stream 'shell-test-router-even-log'

dataflow:>stream deploy --name shell-test-router-odd-log --properties "app.log.name=odd-log,\
deployer.*.cpu=100m,\
deployer.*.kubernetes.limits.cpu=2000m,\
deployer.*.kubernetes.limits.memory=2000Mi,\
deployer.*.memory=1000Mi,\
spring.cloud.dataflow.skipper.platformName=default"
Deployment request has been sent for stream 'shell-test-router-odd-log'
```  

다시 `stream list` 로 상태를 조회해서 `Status` 를 보면 아래와 같이 배포가 완료된 것을 확인 할 수 있다.  

```bash
dataflow:>stream list
╔═════════════════════════════════════╤═══════════╤═════════════════════════════════════════════════════════════════════════════╤══════════════════════════════════════════════════════════════════════╗
║             Stream Name             │Description│                              Stream Definition                              │                                Status                                ║
╠═════════════════════════════════════╪═══════════╪═════════════════════════════════════════════════════════════════════════════╪══════════════════════════════════════════════════════════════════════╣
║shell-test-router-even-log           │           │:test-router-even > log                                                      │The stream has been successfully deployed                             ║
║shell-test-router-odd-log            │           │:test-router-odd > log                                                       │The stream has been successfully deployed                             ║
║shell-time-transform-router          │           │time | transform | router                                                    │The stream has been successfully deployed                             ║
╚═════════════════════════════════════╧═══════════╧═════════════════════════════════════════════════════════════════════════════╧══════════════════════════════════════════════════════════════════════╝
```  

### 실행 중인 스트림 애플리케이션 확인 (runtime apps)
`runtime apps` 명령을 사용하면 현재 `SCDF` 을 통해 실행 중인 애플리케이션의 정보를 확인 할 수 있다.  

```bash
dataflow:>runtime apps
╔═══════════════════════════════════════════════════════════════╤═══════════╤══════════════════════════════════════════════════════════════════════════════════════════╗
║                     App Id / Instance Id                      │Unit Status│                              No. of Instances / Attributes                               ║
╠═══════════════════════════════════════════════════════════════╪═══════════╪══════════════════════════════════════════════════════════════════════════════════════════╣
║shell-test-router-even-log-log-v5                              │ deployed  │                                            1                                             ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = 4900c7a0-96b5-4e45-af84-7fc6d9e7e5a3                           ║
║                                                               │           │                 host.ip = 172.172.113.215                                                ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.174.135.104                                                ║
║                                                               │           │                pod.name = shell-test-router-even-log-log-v5-5767bfc974-md2gl             ║
║shell-test-router-even-log-log-v5-5767bfc974-md2gl             │ deployed  │           pod.startTime = 2023-07-13T09:51:50Z                                           ║
║                                                               │           │            service.name = shell-test-router-even-log-log                                 ║
║                                                               │           │skipper.application.name = log                                                            ║
║                                                               │           │    skipper.release.name = shell-test-router-even-log                                     ║
║                                                               │           │ skipper.release.version = 5                                                              ║
║                                                               │           │           spring.app.id = shell-test-router-even-log-log-v5                              ║
║                                                               │           │    spring.deployment.id = shell-test-router-even-log-log-v5                              ║
╟───────────────────────────────────────────────────────────────┼───────────┼──────────────────────────────────────────────────────────────────────────────────────────╢
║shell-test-router-odd-log-log-v2                               │ deployed  │                                            1                                             ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = 1f2a83c4-dbf7-4efd-8ce1-c44a5b2469e6                           ║
║                                                               │           │                 host.ip = 172.174.53.143                                                 ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.175.35.119                                                 ║
║                                                               │           │                pod.name = shell-test-router-odd-log-log-v2-59468fdb6-prhd9               ║
║shell-test-router-odd-log-log-v2-59468fdb6-prhd9               │ deployed  │           pod.startTime = 2023-07-13T09:52:04Z                                           ║
║                                                               │           │            service.name = shell-test-router-odd-log-log                                  ║
║                                                               │           │skipper.application.name = log                                                            ║
║                                                               │           │    skipper.release.name = shell-test-router-odd-log                                      ║
║                                                               │           │ skipper.release.version = 2                                                              ║
║                                                               │           │           spring.app.id = shell-test-router-odd-log-log-v2                               ║
║                                                               │           │    spring.deployment.id = shell-test-router-odd-log-log-v2                               ║
╟───────────────────────────────────────────────────────────────┼───────────┼──────────────────────────────────────────────────────────────────────────────────────────╢
║shell-time-transform-router-router-v8                          │ deployed  │                                            1                                             ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = 66a10609-9f84-4082-8e0d-d11476f7ac02                           ║
║                                                               │           │                 host.ip = 172.113.134.16                                                 ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.175.9.221                                                  ║
║                                                               │           │                pod.name = shell-time-transform-router-router-v8-5c87596779-vwjp5         ║
║shell-time-transform-router-router-v8-5c87596779-vwjp5         │ deployed  │           pod.startTime = 2023-07-13T09:52:00Z                                           ║
║                                                               │           │            service.name = shell-time-transform-router-router                             ║
║                                                               │           │skipper.application.name = router                                                         ║
║                                                               │           │    skipper.release.name = shell-time-transform-router                                    ║
║                                                               │           │ skipper.release.version = 8                                                              ║
║                                                               │           │           spring.app.id = shell-time-transform-router-router-v8                          ║
║                                                               │           │    spring.deployment.id = shell-time-transform-router-router-v8                          ║
╟───────────────────────────────────────────────────────────────┼───────────┼──────────────────────────────────────────────────────────────────────────────────────────╢
║shell-time-transform-router-transform-v8                       │ deployed  │                                            1                                             ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = 5bdb4cbf-b5b3-4d4e-b74e-17e8593fb990                           ║
║                                                               │           │                 host.ip = 172.174.53.91                                                  ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.175.27.80                                                  ║
║                                                               │           │                pod.name = shell-time-transform-router-transform-v8-699d68cbff-nt4v2      ║
║shell-time-transform-router-transform-v8-699d68cbff-nt4v2      │ deployed  │           pod.startTime = 2023-07-13T09:52:00Z                                           ║
║                                                               │           │            service.name = shell-time-transform-router-transform                          ║
║                                                               │           │skipper.application.name = transform                                                      ║
║                                                               │           │    skipper.release.name = shell-time-transform-router                                    ║
║                                                               │           │ skipper.release.version = 8                                                              ║
║                                                               │           │           spring.app.id = shell-time-transform-router-transform-v8                       ║
║                                                               │           │    spring.deployment.id = shell-time-transform-router-transform-v8                       ║
╟───────────────────────────────────────────────────────────────┼───────────┼──────────────────────────────────────────────────────────────────────────────────────────╢
║shell-time-transform-router-time-v8                            │ deployed  │                                            1                                             ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = 666a8226-bc03-4f3d-9211-a1477ca02517                           ║
║                                                               │           │                 host.ip = 172.172.113.211                                                ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.174.133.17                                                 ║
║                                                               │           │                pod.name = shell-time-transform-router-time-v8-58d9b9798b-x95rw           ║
║shell-time-transform-router-time-v8-58d9b9798b-x95rw           │ deployed  │           pod.startTime = 2023-07-13T09:52:00Z                                           ║
║                                                               │           │            service.name = shell-time-transform-router-time                               ║
║                                                               │           │skipper.application.name = time                                                           ║
║                                                               │           │    skipper.release.name = shell-time-transform-router                                    ║
║                                                               │           │ skipper.release.version = 8                                                              ║
║                                                               │           │           spring.app.id = shell-time-transform-router-time-v8                            ║
║                                                               │           │    spring.deployment.id = shell-time-transform-router-time-v8                            ║
╚═══════════════════════════════════════════════════════════════╧═══════════╧══════════════════════════════════════════════════════════════════════════════════════════╝
```  

### 스트림 정보관련 확인하기 (stream info, stream manifest, stream validate)
`stream info --name {stream_name}` 명령은 현재 스트림의 상태, 정의, 배포 프로퍼티 등을 확인 할 수 있다. 

```bash
dataflow:>stream info --name shell-time-transform-router
╔═══════════════════════════╤════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╤═══════════╤════════╗
║        Stream Name        │                                                                                                     Stream Definition                                                                                                      │Description│ Status ║
╠═══════════════════════════╪════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╪═══════════╪════════╣
║shell-time-transform-router│time --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown}                                         │           │deployed║
║                           │--management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --time.date-format='yyyy-MM-dd        │           │        ║
║                           │HH:mm:ss' --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                                                                                                │           │        ║
║                           │--wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                │           │        ║
║                           │--spring.integration.poller.fixed-rate=1000 --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown}                                                                                             │           │        ║
║                           │--management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | transform     │           │        ║
║                           │--management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown}                                              │           │        ║
║                           │--management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}                                       │           │        ║
║                           │--management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                                                                                                          │           │        ║
║                           │--wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                │           │        ║
║                           │--spel.function.expression='payload.substring(payload.length() - 1)' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown}                                                                    │           │        ║
║                           │--management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | router        │           │        ║
║                           │--wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                │           │        ║
║                           │--management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown}                                              │           │        ║
║                           │--management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --router.expression="new              │           │        ║
║                           │Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'" --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown}                                                                     │           │        ║
║                           │--management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}}                                                                                           │           │        ║
║                           │--management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown}                                │           │        ║
╚═══════════════════════════╧════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╧═══════════╧════════╝

Stream Deployment properties: {
  "router" : {
    "spring.cloud.deployer.cpu" : "100m",
    "spring.cloud.deployer.kubernetes.limits.cpu" : "2000m",
    "resource" : "docker:springcloudstream/router-sink-kafka",
    "spring.cloud.deployer.kubernetes.limits.memory" : "1000Mi",
    "spring.cloud.deployer.memory" : "500Mi",
    "spring.cloud.deployer.group" : "shell-time-transform-router",
    "version" : "3.2.1"
  },
  "transform" : {
    "spring.cloud.deployer.cpu" : "100m",
    "spring.cloud.deployer.kubernetes.limits.cpu" : "2000m",
    "resource" : "docker:springcloudstream/transform-processor-kafka",
    "spring.cloud.deployer.kubernetes.limits.memory" : "1000Mi",
    "spring.cloud.deployer.memory" : "500Mi",
    "spring.cloud.deployer.group" : "shell-time-transform-router",
    "version" : "3.2.1"
  },
  "time" : {
    "spring.cloud.deployer.cpu" : "100m",
    "spring.cloud.deployer.kubernetes.limits.cpu" : "2000m",
    "resource" : "docker:springcloudstream/time-source-kafka",
    "spring.cloud.deployer.kubernetes.limits.memory" : "1000Mi",
    "spring.cloud.deployer.memory" : "500Mi",
    "spring.cloud.deployer.group" : "shell-time-transform-router",
    "version" : "3.2.1"
  }
}

```  

`stream manifest --name {stream_name}` 명령은 스트림의 배포 설정 파일을 조회 할 수 있다.  

```bash
dataflow:>stream manifest --name shell-time-transform-router
apiVersion": "skipper.spring.io/v1"
"kind": "SpringCloudDeployerApplication"
"metadata":
  "name": "router"
"spec":
  "resource": "docker:springcloudstream/router-sink-kafka"
  "resourceMetadata": "docker:springcloudstream/router-sink-kafka:jar:metadata:3.2.1"
  "version": "3.2.1"
  "applicationProperties":
    "spring.cloud.dataflow.stream.app.label": "router"
    "management.metrics.tags.application.type": "${spring.cloud.dataflow.stream.app.type:unknown}"
    "management.metrics.tags.stream.name": "${spring.cloud.dataflow.stream.name:unknown}"
    "management.metrics.tags.application": "${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}"
    "spring.cloud.dataflow.stream.name": "shell-time-transform-router"
    "management.metrics.tags.instance.index": "${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}"
    "wavefront.application.service": "${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}"
    "spring.cloud.stream.bindings.input.group": "shell-time-transform-router"
    "router.expression": "new Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'"
    "management.metrics.tags.application.guid": "${spring.cloud.application.guid:unknown}"
    "management.metrics.tags.application.name": "${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}}"
    "spring.cloud.dataflow.stream.app.type": "sink"
    "spring.cloud.stream.bindings.input.destination": "shell-time-transform-router.transform"
    "wavefront.application.name": "${spring.cloud.dataflow.stream.name:unknown}"
  "deploymentProperties":
    "spring.cloud.deployer.cpu": "100m"
    "spring.cloud.deployer.kubernetes.limits.cpu": "2000m"
    "spring.cloud.deployer.kubernetes.limits.memory": "1000Mi"
    "spring.cloud.deployer.memory": "500Mi"
    "spring.cloud.deployer.group": "shell-time-transform-router"
---
"apiVersion": "skipper.spring.io/v1"
"kind": "SpringCloudDeployerApplication"
"metadata":
  "name": "transform"
"spec":
  "resource": "docker:springcloudstream/transform-processor-kafka"
  "resourceMetadata": "docker:springcloudstream/transform-processor-kafka:jar:metadata:3.2.1"
  "version": "3.2.1"
  "applicationProperties":
    "spring.cloud.dataflow.stream.app.label": "transform"
    "management.metrics.tags.application.type": "${spring.cloud.dataflow.stream.app.type:unknown}"
    "management.metrics.tags.stream.name": "${spring.cloud.dataflow.stream.name:unknown}"
    "management.metrics.tags.application": "${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}"
    "spring.cloud.dataflow.stream.name": "shell-time-transform-router"
    "management.metrics.tags.instance.index": "${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}"
    "wavefront.application.service": "${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}"
    "spring.cloud.stream.bindings.input.group": "shell-time-transform-router"
    "spel.function.expression": "payload.substring(payload.length() - 1)"
    "spring.cloud.stream.bindings.output.producer.requiredGroups": "shell-time-transform-router"
    "management.metrics.tags.application.guid": "${spring.cloud.application.guid:unknown}"
    "spring.cloud.stream.bindings.output.destination": "shell-time-transform-router.transform"
    "management.metrics.tags.application.name": "${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}}"
    "spring.cloud.dataflow.stream.app.type": "processor"
    "spring.cloud.stream.bindings.input.destination": "shell-time-transform-router.time"
    "wavefront.application.name": "${spring.cloud.dataflow.stream.name:unknown}"
  "deploymentProperties":
    "spring.cloud.deployer.cpu": "100m"
    "spring.cloud.deployer.kubernetes.limits.cpu": "2000m"
    "spring.cloud.deployer.kubernetes.limits.memory": "1000Mi"
    "spring.cloud.deployer.memory": "500Mi"
    "spring.cloud.deployer.group": "shell-time-transform-router"
---
"apiVersion": "skipper.spring.io/v1"
"kind": "SpringCloudDeployerApplication"
"metadata":
  "name": "time"
"spec":
  "resource": "docker:springcloudstream/time-source-kafka"
  "resourceMetadata": "docker:springcloudstream/time-source-kafka:jar:metadata:3.2.1"
  "version": "3.2.1"
  "applicationProperties":
    "spring.cloud.dataflow.stream.app.label": "time"
    "management.metrics.tags.application.type": "${spring.cloud.dataflow.stream.app.type:unknown}"
    "management.metrics.tags.stream.name": "${spring.cloud.dataflow.stream.name:unknown}"
    "management.metrics.tags.application": "${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}"
    "spring.cloud.dataflow.stream.name": "shell-time-transform-router"
    "time.date-format": "yyyy-MM-dd HH:mm:ss"
    "management.metrics.tags.instance.index": "${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}"
    "wavefront.application.service": "${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}"
    "spring.integration.poller.fixed-rate": "1000"
    "spring.cloud.stream.bindings.output.producer.requiredGroups": "shell-time-transform-router"
    "management.metrics.tags.application.guid": "${spring.cloud.application.guid:unknown}"
    "spring.cloud.stream.bindings.output.destination": "shell-time-transform-router.time"
    "management.metrics.tags.application.name": "${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}}"
    "spring.cloud.dataflow.stream.app.type": "source"
    "wavefront.application.name": "${spring.cloud.dataflow.stream.name:unknown}"
  "deploymentProperties":
    "spring.cloud.deployer.cpu": "100m"
    "spring.cloud.deployer.kubernetes.limits.cpu": "2000m"
    "spring.cloud.deployer.kubernetes.limits.memory": "1000Mi"
    "spring.cloud.deployer.memory": "500Mi"
    "spring.cloud.deployer.group": "shell-time-transform-router"
```  

`stream validate --name {stream_name}` 명령은 현재 설정된 스트림 구성 정보(프로퍼티 포함)이 
옮바른지 검사를 할 수 있다.  

```bash
dataflow:>stream validate --name shell-time-transform-router
╔═══════════════════════════╤═════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║        Stream Name        │                                                                                                                Stream Definition                                                                                                                ║
╠═══════════════════════════╪═════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
║shell-time-transform-router│time --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown}                                                              ║
║                           │--management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --time.date-format='yyyy-MM-dd HH:mm:ss'                   ║
║                           │--management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                                                                                                                               ║
║                           │--wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                                     ║
║                           │--spring.integration.poller.fixed-rate=1000 --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown}                                                                                                                  ║
║                           │--management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | transform                          ║
║                           │--management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown}                                                                   ║
║                           │--management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}                                                            ║
║                           │--management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                                                                                                                               ║
║                           │--wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                                     ║
║                           │--spel.function.expression='payload.substring(payload.length() - 1)' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown}                                                                                         ║
║                           │--management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | router                             ║
║                           │--management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown}                                                                   ║
║                           │--management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}                                                            ║
║                           │--management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}}                                                                                                                               ║
║                           │--wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --router.expression="new            ║
║                           │Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'" --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown}                                                                                          ║
║                           │--management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown}                                      ║
╚═══════════════════════════╧═════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝

╔═══════════════════╤═════════════════╗
║     App Name      │Validation Status║
╠═══════════════════╪═════════════════╣
║processor:transform│valid            ║
║sink:router        │valid            ║
║source:time        │valid            ║
╚═══════════════════╧═════════════════╝


shell-time-transform-router is a valid stream.
```  

### 스트림 업데이트 (stream update)
`stream update --name {stream_name} --properties {stream_update_properties}` 
명령을 사용하면 스트림의 프로퍼티를 수정해서 재배포 할 수 있다.  

```bash
dataflow:>stream update --name shell-time-transform-router --properties "app.time.spring.integration.poller.fixed-rate=100"
Update request has been sent for the stream 'shell-time-transform-router'
```  

`runtime apps` 로 다시 조회하면 `hell-time-transform-router-time` 만 `v9` 로 재시작되면 버전이 올라간것을 확인 할 수 있다.  

```bash
dataflow:>runtime apps
╔═══════════════════════════════════════════════════════════════╤═══════════╤══════════════════════════════════════════════════════════════════════════════════════════╗
║                     App Id / Instance Id                      │Unit Status│                              No. of Instances / Attributes                               ║
╠═══════════════════════════════════════════════════════════════╪═══════════╪══════════════════════════════════════════════════════════════════════════════════════════╣
║shell-time-transform-router-time-v9                            │ deployed  │                                            1                                             ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = adb3d26d-1703-49a5-9388-b050730fccd2                           ║
║                                                               │           │                 host.ip = 172.172.113.218                                                ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.174.136.245                                                ║
║                                                               │           │                pod.name = shell-time-transform-router-time-v9-869f6d87c7-nnrsz           ║
║shell-time-transform-router-time-v9-869f6d87c7-nnrsz           │ deployed  │           pod.startTime = 2023-07-13T10:08:31Z                                           ║
║                                                               │           │            service.name = shell-time-transform-router-time                               ║
║                                                               │           │skipper.application.name = time                                                           ║
║                                                               │           │    skipper.release.name = shell-time-transform-router                                    ║
║                                                               │           │ skipper.release.version = 9                                                              ║
║                                                               │           │           spring.app.id = shell-time-transform-router-time-v9                            ║
║                                                               │           │    spring.deployment.id = shell-time-transform-router-time-v9                            ║
╚═══════════════════════════════════════════════════════════════╧═══════════╧══════════════════════════════════════════════════════════════════════════════════════════╝
```  

`shell-transform-router` 스트림에서 `time` 애플리케이션의 프로퍼티만 변겅 됐기 때문에, 
해당 애플리케이션만 새로운 버전으로 재시작 된다.  

### 스트림 스케일 조정 (stream scale app)
`stream scale app --name {stream_name} --applicationName {app_name_in_stream} --count {scale_count}` 
명령은 스트림에서 특정 애플리케이션의 스케일을 조정할 수 있다.  

`shell-time-transform-router` 스트림의 `time` 애플리케이션을 2개로 늘리면, 
아래와 같이 `runtime apps` 실행 시 2개의 `Pod` 이 조회되는 것을 확인 할 수 있다. 


```bash
dataflow:>stream scale app instances --name shell-time-transform-router --applicationName time --count 2
Scale request has been sent for the stream 'shell-time-transform-router'

dataflow:>runtime apps
╔═══════════════════════════════════════════════════════════════╤═══════════╤══════════════════════════════════════════════════════════════════════════════════════════╗
║                     App Id / Instance Id                      │Unit Status│                              No. of Instances / Attributes                               ║
╠═══════════════════════════════════════════════════════════════╪═══════════╪══════════════════════════════════════════════════════════════════════════════════════════╣
║shell-time-transform-router-time-v9                            │ deployed  │                                            2                                             ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = a882826a-ffe0-461a-aefb-6cd517b14b58                           ║
║                                                               │           │                 host.ip = 172.174.53.98                                                  ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.175.30.240                                                 ║
║                                                               │           │                pod.name = shell-time-transform-router-time-v9-869f6d87c7-b5psm           ║
║shell-time-transform-router-time-v9-869f6d87c7-b5psm           │ deployed  │           pod.startTime = 2023-07-13T11:31:46Z                                           ║
║                                                               │           │            service.name = shell-time-transform-router-time                               ║
║                                                               │           │skipper.application.name = time                                                           ║
║                                                               │           │    skipper.release.name = shell-time-transform-router                                    ║
║                                                               │           │ skipper.release.version = 9                                                              ║
║                                                               │           │           spring.app.id = shell-time-transform-router-time-v9                            ║
║                                                               │           │    spring.deployment.id = shell-time-transform-router-time-v9                            ║
╟┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┼┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈╢
║                                                               │           │  container.restartCount = 0                                                              ║
║                                                               │           │                    guid = adb3d26d-1703-49a5-9388-b050730fccd2                           ║
║                                                               │           │                 host.ip = 172.172.113.218                                                ║
║                                                               │           │                   phase = Running                                                        ║
║                                                               │           │                  pod.ip = 172.174.136.245                                                ║
║                                                               │           │                pod.name = shell-time-transform-router-time-v9-869f6d87c7-nnrsz           ║
║shell-time-transform-router-time-v9-869f6d87c7-nnrsz           │ deployed  │           pod.startTime = 2023-07-13T10:08:31Z                                           ║
║                                                               │           │            service.name = shell-time-transform-router-time                               ║
║                                                               │           │skipper.application.name = time                                                           ║
║                                                               │           │    skipper.release.name = shell-time-transform-router                                    ║
║                                                               │           │ skipper.release.version = 9                                                              ║
║                                                               │           │           spring.app.id = shell-time-transform-router-time-v9                            ║
║                                                               │           │    spring.deployment.id = shell-time-transform-router-time-v9                            ║
╚═══════════════════════════════════════════════════════════════╧═══════════╧══════════════════════════════════════════════════════════════════════════════════════════╝
```  


### 스트림 배포 취소 (stream undeploy)
`stream undeploy --name {stream_name}` 명령을 사용하면 배포된 스트림을 배포 취소 할 수 있다.  

```bash
dataflow:>stream undeploy --name shell-test-router-even-log
Un-deployed stream 'shell-test-router-even-log'
dataflow:>stream undeploy --name shell-test-router-odd-log
Un-deployed stream 'shell-test-router-odd-log'
dataflow:>stream undeploy --name shell-time-transform-router
Un-deployed stream 'shell-time-transform-router'


dataflow:>stream list
╔═════════════════════════════════════╤═══════════╤═════════════════════════════════════════════════════════════════════════════╤══════════════════════════════════════════════════════════════════════╗
║             Stream Name             │Description│                              Stream Definition                              │                                Status                                ║
╠═════════════════════════════════════╪═══════════╪═════════════════════════════════════════════════════════════════════════════╪══════════════════════════════════════════════════════════════════════╣
║shell-test-router-even-log           │           │:test-router-even > log                                                      │The app or group is known to the system, but is not currently deployed║
║shell-test-router-odd-log            │           │:test-router-odd > log                                                       │The app or group is known to the system, but is not currently deployed║
║shell-time-transform-router          │           │time | transform | router                                                    │The app or group is known to the system, but is not currently deployed║
╚═════════════════════════════════════╧═══════════╧═════════════════════════════════════════════════════════════════════════════╧══════════════════════════════════════════════════════════════════════╝
```  

### 스트림 삭제 (stream destroy)
`stream destroy --name {stream_name}` 명령은 이미 정의된 스트림을 삭제할 수 있다.  

```bash
dataflow:>stream destroy --name shell-test-router-even-log
Destroyed stream 'shell-test-router-even-log'
dataflow:>stream destroy --name shell-test-router-odd-log
Destroyed stream 'shell-test-router-odd-log'
dataflow:>stream destroy --name shell-time-transform-router
Destroyed stream 'shell-time-transform-router'

```  


### 명령어 히스토리 조회 (history)
`history` 명령은 그 동안 `Shell` 에서 수행한 명령어 이력을 조회해 볼 수 있다. 

```bash
dataflow:>history
[stream list, help, runtime apps, runtime apps --name time-transform-router, runtime apps time-transform-router, help, stream info --name time-transform-router, stream info --name shell-time-transform-router, stream manifest --name shell-time-transform-router, stream validate --name shell-time-transform-router, stream update --name shell-time-transform-router --properties "app.time.spring.integration.poller.fixed-rate=100", help, runtime apps shell, runtime apps, runtime apps help, stream scale app instances --name shell-time-transform-router --applicationName time --count 2, app list, runtime apps, runtime app list, runtime apps, stream undeploy --name shell-test-router-even-log, stream undeploy --name shell-test-router-odd-log, stream undeploy --name shell-time-transform-router, stream list, runtime apps, stream destroy --name shell-test-router-even-log, stream destroy --name shell-test-router-odd-log, stream destroy --name shell-time-transform-router, stream list, history]
```  




---  
## Reference
[Spring Cloud Dataflow Shell](https://docs.spring.io/spring-cloud-dataflow/docs/2.10.3/reference/htmlsingle/#shell)  
