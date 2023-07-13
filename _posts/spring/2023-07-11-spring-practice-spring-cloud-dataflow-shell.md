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








---  
## Reference
[]()  