--- 
layout: single
classes: wide
title: "[Spring 실습] Spring Cloud Data Flow(SCDF) Kubernetes 테스트 환경 구축"
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
toc: true
use_math: true
---  

## Spring Cloud Data Flow(SCDF) Kubernetes Installation
본 포스트는 [가이드](https://dataflow.spring.io/docs/installation/kubernetes/kubectl/) 
문서를 바탕으로 작성한 글임을 알린다.  

`Spring Cloud Data Flow` 를 로컬에서 간단하게 구현해 테스트 해볼 수 있는 환경을 구축해 본다. 
공식 가이드에서는 풀 스택을 가이드하고 있는데 여기서 테스트에 필요한 아래 구성만 사용해서 간단한 환경을 구축한다. 

- `Messaging Middleware` : `Kafka`, `Zookeeper`
- `SCDF Skipper`
- `SCDF Server`

`MySQL` 은 별도로 구성하지 않고 `SCDF` 에서는 기본으로 사용되는 내장 `H2` 를 사용하도록 한다. 

구성에 사용한 환경은 아래와 같다. 

- `Docker 23.0.6`
- `Minikube v1.30.1`
- `Kubernetes v1.20.15`

구성에 사용한 파일의 디렉토리 구조는 아래와 같다.  

```
.
├── kafka
│   ├── kafka-deploy.yaml
│   └── zookeeper-deploy.yaml
├── server
│   ├── server-config.yaml
│   ├── server-deployment.yaml
│   ├── server-rolebinding.yaml
│   ├── server-roles.yaml
│   ├── server-svc.yaml
│   └── service-account.yaml
└── skipper
    ├── skipper-config-kafka.yaml
    ├── skipper-deployment.yaml
    └── skipper-svc.yaml

```  

### Messaging Middleware 구성
`SCDF` 에서 사용할 `Messaging Middleware` 는 `Kafka` 로 아래 `Kafka` 와 `Zookeeper` 템플릿을 사용한다. 
`SCDF` 에서 배포되는 `Stream Application` 은 해당 `Messaging Middleware` 를 사용해 동작 메시지를 주고 받으며 실행된다. 

```yaml
# kafka/zookeeper-deploy.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: zookeeper-deployment
  labels:
    app: zookeeper
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
        - name: zookeeper
          image: confluentinc/cp-zookeeper:7.0.1
          ports:
            - containerPort: 2181
          env:
            - name: ZOOKEEPER_CLIENT_PORT
              value: "2181"
            - name: ZOOKEEPER_TICK_TIME
              value: "2000"
---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper-service
spec:
  selector:
    app: zookeeper
  ports:
    - protocol: TCP
      port: 2181
      targetPort: 2181
```  

```yaml
# kafka/kafka-deploy.yaml

kind: Deployment
apiVersion: apps/v1
metadata:
  name: kafka-deployment
  labels:
    app: kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
        - name: broker
          image: confluentinc/cp-kafka:7.0.1
          ports:
            - containerPort: 9092
          env:
            - name: KAFKA_BROKER_ID
              value: "1"
            - name: KAFKA_ZOOKEEPER_CONNECT
              value: 'zookeeper-service:2181'
            - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
              value: PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT
            - name: KAFKA_ADVERTISED_LISTENERS
              value: PLAINTEXT://:29092,PLAINTEXT_INTERNAL://kafka-service:9092
            - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
              value: "1"
            - name: KAFKA_TRANSACTION_STATE_LOG_MIN_ISR
              value: "1"
            - name: KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR
              value: "1"
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-service
spec:
  selector:
    app: kafka
  ports:
    - protocol: TCP
      port: 9092
      targetPort: 9092
```  

아래 명령어 순으로 실행하고 `zookeeper`, `kafka` 의 `Pod` 이 모두 `Ready` 상태가 될때까지 기다려 준다.  

```bash
$ kubectl apply -f kafka/zookeeper-deploy.yaml
deployment.apps/zookeeper-deployment created
service/zookeeper-service created

$ kubectl apply -f kafka/kafka-deploy.yaml
deployment.apps/kafka-deployment created
service/kafka-service created

$ kubectl get all -l app=kafka
NAME                                    READY   STATUS    RESTARTS   AGE
pod/kafka-deployment-5d9fff8db8-js5ql   1/1     Running   1          84s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kafka-deployment   1/1     1            1           84s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/kafka-deployment-5d9fff8db8   1         1         1       84s

$ kubectl get all -l app=zookeeper
NAME                                        READY   STATUS    RESTARTS   AGE
pod/zookeeper-deployment-5967786f47-58xk6   1/1     Running   0          94s

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/zookeeper-deployment   1/1     1            1           94s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/zookeeper-deployment-5967786f47   1         1         1       94s
```  

### SCDF Skipper and Server
`SCDF` 에서 `Skipper` 는 `Server` 에 요청에 따라 `Stream Application` 을 직접 관리하고 컨트롤하는 역할을 한다. 
`Stream Application` 을 배포, 롤백, 업데이트, 삭제 등의 동작을 `Server` 로 부터 위임 받아 수행해 준다.  

```yaml
# skipper/skipper-config-kafka.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: skipper
  labels:
    app: skipper
data:
  application.yaml: |-
    spring:
      cloud:
        skipper:
          server:
            platform:
              kubernetes:
                accounts:
                  default:
                    environmentVariables: 'SPRING_CLOUD_STREAM_KAFKA_BINDER_BROKERS=kafka-service:9092,SPRING_CLOUD_STREAM_KAFKA_BINDER_ZK_NODES=${KAFKA_ZK_SERVICE_HOST}:${KAFKA_ZK_SERVICE_PORT}'
                    limits:
                      memory: 1g
                      cpu: 1.0                    
                    readinessProbeDelay: 1
                    readinessProbeTimeout: 5
                    livenessProbeDelay: 1
                    livenessProbeTimeout: 2
                    startupProbeDelay: 20
                    startupProbeTimeout: 5
                    startupProbeFailure: 50

```  

```yaml
# skipper/skipper-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: skipper
  labels:
    app: skipper
spec:
  selector:
    matchLabels:
      app: skipper
  replicas: 1
  template:
    metadata:
      labels:
        app: skipper
    spec:
      containers:
        - name: skipper
          image: springcloud/spring-cloud-skipper-server:2.11.0
          imagePullPolicy: Always
          volumeMounts:
            - name: config
              mountPath: /workspace/config
              readOnly: true
          ports:
            - containerPort: 7577
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 7577
            initialDelaySeconds: 1
            periodSeconds: 60
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /actuator/info
              port: 7577
            initialDelaySeconds: 1
            periodSeconds: 10
            timeoutSeconds: 2
          startupProbe:
            tcpSocket:
              port: 7577
            initialDelaySeconds: 15
            periodSeconds: 3
            failureThreshold: 120
            timeoutSeconds: 3
          resources:
            requests:
              cpu: 1.0
              memory: 1024Mi
          env:
            - name: LANG
              value: 'en_US.utf8'
            - name: LC_ALL
              value: 'en_US.utf8'
            - name: JDK_JAVA_OPTIONS
              value: '-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8'
            - name: SERVER_PORT
              value: '7577'
            - name: SPRING_CLOUD_CONFIG_ENABLED
              value: 'false'
            - name: SPRING_CLOUD_KUBERNETES_CONFIG_ENABLE_API
              value: 'false'
            - name: SPRING_CLOUD_KUBERNETES_SECRETS_ENABLE_API
              value: 'false'
            - name: SPRING_CLOUD_KUBERNETES_SECRETS_PATHS
              value: /etc/secrets
      serviceAccountName: scdf-sa
      volumes:
        - name: config
          configMap:
            name: skipper
            items:
              - key: application.yaml
                path: application.yaml
```  

```yaml
# skipper/skipper-svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: skipper
  labels:
    app: skipper
    spring-deployment-id: scdf
spec:
  # If you are running k8s on a local dev box, using minikube, or Kubernetes on docker desktop you can use type NodePort instead
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 7577
  selector:
    app: skipper
```  

그리고 `Server` 는 `SCDF` 에서 주로 `Web UI` 역할을 수행해주며, 
`DSL` 을 바탕으로 스트림을 정의 할 수 있도록 한다. 
그리고 배포시에 프로퍼티를 받아 이를 실제 스트림 애플리케이션에 전달하는 역할과 앞서 언급한 것처럼 스트림 애플리케이션에 대한 제어 동작을 `Skipper` 에게 위임한다.  

```yaml
# server/server-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: scdf-server
  labels:
    app: scdf-server
data:
  application.yaml: |-
    spring:
      cloud:
        dataflow:
          task:
            platform:
              kubernetes:
                accounts:                  
                  default:
                    limits:
                      memory: 1000Mi
```  

```yaml
# server/server-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: scdf-server
  labels:
    app: scdf-server
spec:
  selector:
    matchLabels:
      app: scdf-server
  replicas: 1
  template:
    metadata:
      labels:
        app: scdf-server
    spec:
      containers:
        - name: scdf-server
          image: springcloud/spring-cloud-dataflow-server:2.11.0
          imagePullPolicy: Always
          volumeMounts:
            - name: config
              mountPath: /workspace/config
              readOnly: true
          ports:
            - containerPort: 9393
          livenessProbe:
            httpGet:
              path: /management/health
              port: 9393
            initialDelaySeconds: 45
          readinessProbe:
            httpGet:
              path: /management/info
              port: 9393
            initialDelaySeconds: 45
          startupProbe:
            tcpSocket:
              port: 9393
            initialDelaySeconds: 15
            periodSeconds: 3
            failureThreshold: 120
            timeoutSeconds: 3
          resources:
            requests:
              cpu: 1.0
              memory: 2048Mi
          env:
            - name: LANG
              value: 'en_US.utf8'
            - name: LC_ALL
              value: 'en_US.utf8'
            - name: JDK_JAVA_OPTIONS
              value: '-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8'
            - name: KUBERNETES_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: "metadata.namespace"
            - name: SERVER_PORT
              value: '9393'
            - name: SPRING_CLOUD_CONFIG_ENABLED
              value: 'false'
            - name: SPRING_CLOUD_DATAFLOW_FEATURES_ANALYTICS_ENABLED
              value: 'true'
            - name: SPRING_CLOUD_DATAFLOW_FEATURES_SCHEDULES_ENABLED
              value: 'true'
            - name: SPRING_CLOUD_DATAFLOW_TASK_COMPOSEDTASKRUNNER_URI
              value: 'docker://springcloud/spring-cloud-dataflow-composed-task-runner:2.9.1'
            - name: SPRING_CLOUD_KUBERNETES_CONFIG_ENABLE_API
              value: 'false'
            - name: SPRING_CLOUD_KUBERNETES_SECRETS_ENABLE_API
              value: 'false'
            - name: SPRING_CLOUD_KUBERNETES_SECRETS_PATHS
              value: /etc/secrets
            - name: SPRING_CLOUD_DATAFLOW_SERVER_URI
              value: 'http://${SCDF_SERVER_SERVICE_HOST}:${SCDF_SERVER_SERVICE_PORT}'
              # Provide the Skipper service location
            - name: SPRING_CLOUD_SKIPPER_CLIENT_SERVER_URI
              value: 'http://${SKIPPER_SERVICE_HOST}:${SKIPPER_SERVICE_PORT}/api'
              # Add Maven repo for metadata artifact resolution for all stream apps
            - name: SPRING_APPLICATION_JSON
              value: "{ \"maven\": { \"local-repository\": null, \"remote-repositories\": { \"repo1\": { \"url\": \"https://repo.spring.io/libs-snapshot\"} } } }"
      serviceAccountName: scdf-sa
      volumes:
        - name: config
          configMap:
            name: scdf-server
            items:
              - key: application.yaml
                path: application.yaml
```  

```yaml
# server/server-rolebinding.yaml

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: scdf-rb
subjects:
  - kind: ServiceAccount
    name: scdf-sa
roleRef:
  kind: Role
  name: scdf-role
  apiGroup: rbac.authorization.k8s.io
```  

```yaml
# server/server-roles.yaml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: scdf-role
rules:
  - apiGroups: [""]
    resources: ["services", "pods", "replicationcontrollers", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "delete", "update"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "delete", "update", "patch"]
  - apiGroups: ["extensions"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "delete", "update", "patch"]
  - apiGroups: ["batch"]
    resources: ["cronjobs", "jobs"]
    verbs: ["create", "delete", "get", "list", "watch", "update", "patch"]
```  

```yaml
# server/server-svc.yaml

kind: Service
apiVersion: v1
metadata:
  name: scdf-server
  labels:
    app: scdf-server
    spring-deployment-id: scdf
spec:
  # If you are running k8s on a local dev box or using minikube, you can use type NodePort instead
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 9393
      name: scdf-server
  selector:
    app: scdf-server
```  

```yaml
# server/server-account.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: scdf-sa
```  

`Skipper` 와 `Server` 를 아래 명령어 순으로 실행하고 `Ready` 상태 까지 확인 한다.  

```bash
$ kubectl apply -f skipper 
configmap/skipper created
deployment.apps/skipper created
service/skipper created

$ kubectl apply -f server 
configmap/scdf-server created
deployment.apps/scdf-server created
rolebinding.rbac.authorization.k8s.io/scdf-rb created
role.rbac.authorization.k8s.io/scdf-role created
service/scdf-server created
serviceaccount/scdf-sa created

$ kubectl get all -l app=scdf-server -l app=skipper
NAME                           READY   STATUS    RESTARTS   AGE
pod/skipper-5c7fd7c688-dxxc5   1/1     Running   0          4m41s

NAME              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/skipper   LoadBalancer   10.108.116.18   <pending>     80:31657/TCP   5m53s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/skipper   1/1     1            1           5m53s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/skipper-5c7fd7c688   1         1         1       4m41s

$ kubectl get all -l app=scdf-server   
NAME                              READY   STATUS    RESTARTS   AGE
pod/scdf-server-c8bfd6dbd-crjd4   1/1     Running   0          4m13s

NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/scdf-server   LoadBalancer   10.110.90.203   <pending>     80:30987/TCP   5m23s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/scdf-server   1/1     1            1           5m23s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/scdf-server-c8bfd6dbd    1         1         1       4m13s
```  

### 테스트
모든 구성이 정상적으로 동작하는 지와 간단한 `Stream Application` 을 구성해본다.  

`Minikube` 에서 `Kubernetes` 내부 `Service` 에 접속하기 위해 아래 명령어를 사용해 준다.  

```bash
$ minikube service --url scdf-server
http://127.0.0.1:50939
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.

```  

웹 브라우저에 `http://127.0.0.1:50939/dashboard` 로 접속하면 `SCDF Server` 의 `Web UI` 에 접속할 수 있다.  

spring-cloud-data-flow-installation-1.png

`Add Aplication` 을 누르면 아래 화면이 나온다. 

spring-cloud-data-flow-installation-2.png

위 화면에서 `IMPORT APPLICATION` 을 눌러주면 `Kafaka/Docker` 기반의 기본으로 제공되는 `Stream Application` 들이 추가 된다.  

spring-cloud-data-flow-installation-3.png

기본으로 제공되는 `Stream Application` 이 모두 추가 됐으면 아래와 같이 `Streams` 탭을 눌러 스트림을 생성하는 페이지로 진입한다. 

spring-cloud-data-flow-installation-4.png

위 화면에서 `Create Stream` 버튼을 누르면 `Stream Application` 을 사용해서 `Stream` 을 생성할 수 있는 화면이 나온다. 

spring-cloud-data-flow-installation-5.png

테스트로 생성해볼 스트림은 `Time(Source) - Log(Sink)` 으로 
별도의 `Prcessor` 는 없고 1초마다 시간 값을 보내 콘솔 로그를 찍는 스트림이다. 
`Create Stream` 을 눌러 스트림 이름까지 입력해 생성해 준다.  

spring-cloud-data-flow-installation-6.png

`test-1` 이라는 스트림이 생성됐지만 아직 배포가 되지 않은 상태이다. 

spring-cloud-data-flow-installation-7.png

위 화면에서 `Deploy` 를 눌러 배포 화면으로 넘어간다. 

spring-cloud-data-flow-installation-8.png

위 화면에서 `Stream Application` 실행에 필요한 다양한 프로퍼티를 각각 주입할 수 있다. 
`Kafka` 의 주소는 `Skipper` 배포시에 프로퍼티 설정이 돼있다면 동일한 주소로 자동 설정된다. 
배포 플랫폼은 `default(kubernetes)` 로 설정하고, 
`kubernetes.limit` 에서 `cpu` 는 `1`, `memory` 는 `1Gi` 로 설정해준다. 
그리고 `app.log`, `app.time` 와 같이 각 애플리케이션에 대한 프로퍼티도 설정할 수 있다. 
설정을 다 마치면 `Deploy the stream` 을 눌러 배포를 진행해 준다.  

spring-cloud-data-flow-installation-9.png

배포 동작이 성공하면 위 화면으로 넘어가 현재 배포가 진행 중인 상태를 확인 할 수 있다. 

spring-cloud-data-flow-installation-10.png

그리고 `Streams` 에서 생성한 `test-1` 스트림을 클릭하면 해당 스트림의 상태 및 자세한 정보를 확인 할 수 있다.
배포를 수행하면 아래와 같이 `Kubernetes` 에 스트림 구성이 실행된다. 

```bash
$ kubectl get all | grep test-1
pod/test-1-log-v2-cb994db47-8prl6           1/1     Running   0          2m39s
pod/test-1-time-v2-9fc55b9bb-42qv9          1/1     Running   0          2m39s
service/test-1-log          ClusterIP      10.103.172.106   <none>        8080/TCP       2m39s
service/test-1-time         ClusterIP      10.108.181.100   <none>        8080/TCP       2m40s
deployment.apps/test-1-log-v2          1/1     1            1           2m39s
deployment.apps/test-1-time-v2         1/1     1            1           2m39s
replicaset.apps/test-1-log-v2-cb994db47           1         1         1       2m39s
replicaset.apps/test-1-time-v2-9fc55b9bb          1         1         0       2m39s
```  

`SCDF` 에서 `Messaging Middleware` 로 사용한 `Kafka` 의 토픽 리스트와 해당 토픽의 메시지를 확인 하면 아래와 같다. 

```bash
$ kubectl exec -it kafka-deployment-5d9fff8db8-knj6q -- /bin/bash
[appuser@kafka-deployment-5d9fff8db8-knj6q ~]$ kafka-topics --bootstrap-server localhost:29092 --list
__consumer_offsets
test-1.time
[appuser@kafka-deployment-5d9fff8db8-knj6q ~]$ kafka-console-consumer --bootstrap-server localhost:29092 --topic test-1.time
05/20/23 08:53:35
05/20/23 08:53:36
05/20/23 08:53:37
05/20/23 08:53:38
05/20/23 08:53:39
05/20/23 08:53:40
05/20/23 08:53:41
05/20/23 08:53:42
05/20/23 08:53:43

.....
```  

`Kafka` 토픽은 `<stream name>.<application name>` 으로 생성되는 것을 확인 할 수 있고, 
해당 토픽에는 `Source Application` 에서 생성하는 시간 값이 계속해서 들어오는 것을 확인 할 수 있다. 
최종적으로 `Sink Application` 역할을 하는 `Log` 애플리케이션의 콘솔 로그를 확인하면 아래와 같이 `Kafka` 토픽의 시간값이 콘솔 로그로 찍히는 것을 확인 할 수 있다.  

```bash
$ kubectl logs -f pod/test-1-log-v2-cb994db47-8prl6 

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.8)

2023-05-20 08:45:23.357  INFO [log-sink,,] 1 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at : http://localhost:8888
2023-05-20 08:45:24.677  INFO [log-sink,,] 1 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Connect Timeout Exception on Url - http://localhost:8888. Will be trying the next url if available
2023-05-20 08:45:24.678  WARN [log-sink,,] 1 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Could not locate PropertySource: I/O error on GET request for "http://localhost:8888/log-sink/default": Connection refused (Connection refused); nested exception is java.net.ConnectException: Connection refused (Connection refused)
2023-05-20 08:45:24.785  INFO [log-sink,,] 1 --- [           main] o.s.c.s.a.l.s.k.LogSinkKafkaApplication  : No active profile set, falling back to 1 default profile: "default"
2023-05-20 08:45:44.000  INFO [log-sink,,] 1 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2023-05-20 08:45:44.188  INFO [log-sink,,] 1 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2023-05-20 08:45:44.784  INFO [log-sink,,] 1 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=602ed8c7-39d6-341e-a15c-ad595ce34e90
2023-05-20 08:45:48.658  INFO [log-sink,,] 1 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.integration.config.IntegrationManagementConfiguration' of type [org.springframework.integration.config.IntegrationManagementConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2023-05-20 08:45:49.056  INFO [log-sink,,] 1 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationChannelResolver' of type [org.springframework.integration.support.channel.BeanFactoryChannelResolver] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2023-05-20 08:45:49.662  INFO [log-sink,,] 1 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.stream.app.postprocessor.ContentTypeHeaderBeanPostProcessorAutoConfiguration' of type [org.springframework.cloud.stream.app.postprocessor.ContentTypeHeaderBeanPostProcessorAutoConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2023-05-20 08:45:56.896  INFO [log-sink,,] 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2023-05-20 08:45:57.003  INFO [log-sink,,] 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-05-20 08:45:57.004  INFO [log-sink,,] 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.63]
2023-05-20 08:45:59.078  INFO [log-sink,,] 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-05-20 08:45:59.079  INFO [log-sink,,] 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 33921 ms

.. 생략 ..

2023-05-20 08:56:38.016  INFO [log-sink,1a69451f3d6ad551,f1f8df9b1101e7f6] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:37
2023-05-20 08:56:39.011  INFO [log-sink,fc85b48a005c5f63,f5c6053d0539f200] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:38
2023-05-20 08:56:40.166  INFO [log-sink,05a8abc801b13f2b,b9de20671cfee22b] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:39
2023-05-20 08:56:41.019  INFO [log-sink,f6baa5ba87dc108b,3416ebfcf287070b] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:40
2023-05-20 08:56:42.011  INFO [log-sink,d9cf0fd8162d77cd,78aeb952b46a1d1d] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:41
2023-05-20 08:56:43.008  INFO [log-sink,bd466e2fd14f8cfc,6cea73e897c812a2] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:42
2023-05-20 08:56:44.010  INFO [log-sink,aee69fd319d7c995,88bbbdb4f8e4abe5] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:43
2023-05-20 08:56:45.009  INFO [log-sink,a36a6b08c91cfdd5,d84a4c9a374d9128] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:44
2023-05-20 08:56:46.009  INFO [log-sink,ebc4afb7cadc9f70,b56fb4462538c92b] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:45
2023-05-20 08:56:47.012  INFO [log-sink,883397b3b67643ef,67243b4c19009fd4] 1 --- [container-0-C-1] test-log                                 : 05/20/23 08:56:46
```  

`SCDF` 를 `Kubernetes` 환경에 직접 구축하고 테스트 스트림 애플리케이션까지 배포해서 동작을 확인했다. 
`Spring Cloud Stream` 기반으로 애플리케이션을 개발하면 스트림의 구조와 흐름을 `SCDF` 를 사용해서 괸라하고 파악할 수 있을 것이다.  


---  
## Reference
[Spring Cloud Data Flow Installation Kubectl](https://dataflow.spring.io/docs/installation/kubernetes/kubectl/)
